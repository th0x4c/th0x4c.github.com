---
layout: post
title: "[Oracle] v$session, v$active_session_history の階層問合せによる待機のブロッキング・セッションの特定"
date: 2013-11-05 21:51
comments: true
categories: [Oracle, DB, Performance Tuning]
---
## 目的

v$session, v$active_session_history の blocking_session 列で待機をブロックしているセッションが分かるが、階層問合せによって待機をツリー状に表示して辿ることで最終的にブロックしているセッションを特定する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## 前提知識

### v$session.blocking_session 列、v$active_session_history.blocking_session 列

v$session, v$active_session_history の blocking_session 列でそのセッションをブロックしているセッションの SID が分かる。
(例えばそのセッションが要求しているロックを保持しているセッションの SID)

### 階層問合せ

階層問合せ(もしくは 再帰クエリ)によって、親子関係があるような階層構造、ツリー構造を持つデータの問合せができる。
Oracle Database では独自の `CONNECT BY` 句を使った問合せを行う。
(11gR2 からは業界標準 SQL99 の再帰 `with` 句での問合せも可能)

例えば EMP 表には各従業員の EMPNO 列と、その上位マネージャの EMPNO である MGR 列があるが、以下の階層問合せでマネージャと部下の親子関係を表示できる。
(以下は Wikipedia [再帰クエリ](http://ja.wikipedia.org/wiki/再帰クエリ) から抜粋)

{% codeblock lang:sql %}
SELECT LEVEL, LPAD (' ', 2 * (LEVEL - 1)) || ename employee, empno, mgr
FROM emp
START WITH mgr IS NULL
CONNECT BY PRIOR empno = mgr;
{% endcodeblock %}

     LEVEL EMPLOYEE         EMPNO    MGR
    ------ --------------- ------ ------
         1 KING              7839
         2   JONES           7566   7839
         3     SCOTT         7788   7566
         4       ADAMS       7876   7788
         3     FORD          7902   7566
         4       SMITH       7369   7902
         2   BLAKE           7698   7839
         3     ALLEN         7499   7698
         3     WARD          7521   7698
         3     MARTIN        7654   7698
         3     TURNER        7844   7698
         3     JAMES         7900   7698
         2   CLARK           7782   7839
         3     MILLER        7934   7782

## v$session の階層問合せによる待機のブロッキング・セッションの特定

実際に複数セッションで待機状態を作って試す。

- セッション1:
  - `UPDATE emp SET sal = sal WHERE empno = 7369;`

- セッション2:
  - `UPDATE emp SET sal = sal WHERE empno = 7499;`
  - `UPDATE emp SET sal = sal WHERE empno = 7369;` (セッション1を待機)

- セッション3:
  - `UPDATE emp SET sal = sal WHERE empno = 7521;`
  - `UPDATE emp SET sal = sal WHERE empno = 7499;` (セッション2を待機)

- セッション4:
  - `UPDATE emp SET sal = sal WHERE empno = 7566;`
  - `UPDATE emp SET sal = sal WHERE empno = 7369;` (セッション1を待機)

- セッション5:
  - `UPDATE emp SET sal = sal WHERE empno = 7521;` (セッション3を待機)

- セッション6:
  - `UPDATE emp SET sal = sal WHERE empno = 7499;` (セッション2を待機)

- セッション7:
  - `UPDATE emp SET sal = sal WHERE empno = 7566;` (セッション4を待機)

階層問合せを行い、待機状態を確認する。

{% codeblock lang:sql %}
set pages 1000
set lines 200
col level     for 9999
col hierarchy for a20
col path      for a20
col sid       for 9999
col event     for a32

SELECT
  LEVEL,
  LPAD(' ', 2 * (LEVEL - 1)) || sid hierarchy,
  SYS_CONNECT_BY_PATH(sid, '<-') path,
  sid,
  blocking_session,
  event
FROM v$session
START WITH blocking_session IS NULL
CONNECT BY PRIOR sid = blocking_session
ORDER SIBLINGS BY sid
/
{% endcodeblock %}

    SQL> SELECT
           LEVEL,
           LPAD(' ', 2 * (LEVEL - 1)) || sid hierarchy,
           SYS_CONNECT_BY_PATH(sid, '<-') path,
           sid,
           blocking_session,
           event
         FROM v$session
         START WITH blocking_session IS NULL
         CONNECT BY PRIOR sid = blocking_session
         ORDER SIBLINGS BY sid
         /
    
    LEVEL HIERARCHY            PATH                   SID BLOCKING_SESSION EVENT
    ----- -------------------- -------------------- ----- ---------------- --------------------------------
        1 1                    <-1                      1                  pmon timer
        1 2                    <-2                      2                  VKTM Logical Idle Wait
        1 3                    <-3                      3                  DIAG idle wait
    
    ...(*snip*)
    
        1 141                  <-141                  141                  SQL*Net message from client
        2   10                 <-141<-10               10              141 enq: TX - row lock contention
        3     20               <-141<-10<-20           20               10 enq: TX - row lock contention
        3     136              <-141<-10<-136         136               10 enq: TX - row lock contention
        4       143            <-141<-10<-136<-143    143              136 enq: TX - row lock contention
        2   19                 <-141<-19               19              141 enq: TX - row lock contention
        3     142              <-141<-19<-142         142               19 enq: TX - row lock contention

- セッション1(SID=141) を セッション2(SID=10) と セッション4(SID=19) が待機して、
- セッション2(SID=10) を セッション3(SID=136) と セッション6(SID=20) が待機して、
- セッション3(SID=136) を セッション5(SID=143) が待機して、
- セッション4(SID=19) を セッション7(SID=142) が待機している

つまり、セッション1(SID=141)が最終的にブロックしている起源(ルート)であることがわかる。

なお、`LEVEL` 疑似列により階層のレベルが分かる。また、`SYS_CONNECT_BY_PATH` により階層のパスを表すことができる。

## v$active_session_history の階層問合せによる待機のブロッキング・セッションの特定

v$active_session_history では、アイドル以外の待機状態もしくは CPU 使用中のセッションしか現れない。
上述の例のように最終的に待機させていたセッション(セッション1:SID=141)がアイドル状態だと、ルートのセッションが現れないので単純には階層表示できない。
従って、blocking_session がすべて現れるように工夫する。

{% codeblock lang:sql %}
set pages 1000
set lines 200
col level         for 9999
col hierarchy     for a20
col path          for a20
col session_id    for 9999
col session_state for a13
col event         for a32

SELECT MAX(sample_id) FROM v$active_session_history;

define sample_id = &sample_id

WITH x AS
(
  SELECT session_id, blocking_session, session_state, event
  FROM v$active_session_history
  WHERE sample_id = &sample_id
  UNION ALL
  SELECT blocking_session, null, null, 'IDLE?'
  FROM v$active_session_history
  WHERE sample_id = &sample_id
  AND blocking_session NOT IN (SELECT session_id
                               FROM v$active_session_history
                               WHERE sample_id = &sample_id)
  GROUP BY blocking_session
)
SELECT
  LEVEL,
  LPAD(' ', 2 * (LEVEL - 1)) || session_id hierarchy,
  SYS_CONNECT_BY_PATH(session_id, '<-') path,
  session_id,
  blocking_session,
  session_state,
  event
FROM x
START WITH blocking_session IS NULL
CONNECT BY PRIOR session_id = blocking_session
ORDER SIBLINGS BY session_id
/

undefine sample_id
{% endcodeblock %}

    SQL> SELECT MAX(sample_id) FROM v$active_session_history;
    
    MAX(SAMPLE_ID)
    --------------
            244679
    
    SQL> define sample_id = &sample_id
    Enter value for sample_id: 244679

    SQL> WITH x AS
         (
           SELECT session_id, blocking_session, session_state, event
           FROM v$active_session_history
           WHERE sample_id = &sample_id
           UNION ALL
           SELECT blocking_session, null, null, 'IDLE?'
           FROM v$active_session_history
           WHERE sample_id = &sample_id
           AND blocking_session NOT IN (SELECT session_id
                                        FROM v$active_session_history
                                        WHERE sample_id = &sample_id)
           GROUP BY blocking_session
         )
         SELECT
           LEVEL,
           LPAD(' ', 2 * (LEVEL - 1)) || session_id hierarchy,
           SYS_CONNECT_BY_PATH(session_id, '<-') path,
           session_id,
           blocking_session,
           session_state,
           event
         FROM x
         START WITH blocking_session IS NULL
         CONNECT BY PRIOR session_id = blocking_session
         ORDER SIBLINGS BY session_id
         /
        
    LEVEL HIERARCHY            PATH                 SESSION_ID BLOCKING_SESSION SESSION_STATE EVENT
    ----- -------------------- -------------------- ---------- ---------------- ------------- --------------------------------
        1 141                  <-141                       141                                IDLE?
        2   10                 <-141<-10                    10              141 WAITING       enq: TX - row lock contention
        3     20               <-141<-10<-20                20               10 WAITING       enq: TX - row lock contention
        3     136              <-141<-10<-136              136               10 WAITING       enq: TX - row lock contention
        4       143            <-141<-10<-136<-143         143              136 WAITING       enq: TX - row lock contention
        2   19                 <-141<-19                    19              141 WAITING       enq: TX - row lock contention
        3     142              <-141<-19<-142              142               19 WAITING       enq: TX - row lock contention


アイドル状態のために現れないセッションを `UNION ALL` で仮想的に v$active_session_history に追加したインラインビュー x を使用した。
(もっといいやり方があるかもしれない。)

## 参考

* Wikipedia [再帰クエリ](http://ja.wikipedia.org/wiki/再帰クエリ)
* Oracle Database SQL言語リファレンス 11gリリース2 (11.2) [階層問合わせ](http://download.oracle.com/docs/cd/E16338_01/server.112/b56299/queries003.htm#i2053935)
* 図でイメージするOracle DatabaseのSQL全集 [第6回 階層問い合わせ](http://www.oracle.com/technetwork/jp/articles/otnj-sql-image6-1352143-ja.html), [第7回 再帰with句](http://www.oracle.com/technetwork/jp/articles/otnj-sql-image7-1525406-ja.html)
* Let's postgres [再帰SQL](http://lets.postgresql.jp/documents/technical/with_recursive)
* @IT [新しい業界標準「SQL99」詳細解説](http://www.atmarkit.co.jp/fnetwork/tokusyuu/01sql99/sql99_1b.html)
