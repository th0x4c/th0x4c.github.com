---
layout: post
title: "[Oracle] マテリアライズド・ビューの高速リフレッシュ"
date: 2012-09-12 21:14
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

集計(`AVG(sal)` など)や結合(`emp.deptno = dept.deptno` など)を含むマテリアライズド・ビューを高速リフレッシュ可能にする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* データ・ウェアハウス・ガイド  
  -> 9 基本的なマテリアライズド・ビュー  
  -> [集計を含むマテリアライズド・ビュー](http://docs.oracle.com/cd/E16338_01/server.112/b56309/basicmv.htm#i1006519)  
     [集計を含むマテリアライズド・ビューの高速リフレッシュに関する制限](http://docs.oracle.com/cd/E16338_01/server.112/b56309/basicmv.htm#i1007028)  
     [結合のみを含むマテリアライズド・ビュー](http://docs.oracle.com/cd/E16338_01/server.112/b56309/basicmv.htm#i1006674)  
     [結合のみを含むマテリアライズド・ビューの高速リフレッシュに関する制限](http://docs.oracle.com/cd/E16338_01/server.112/b56309/basicmv.htm#i1007013)  
     [DBMS_MVIEW.EXPLAIN_MVIEWプロシージャの使用](http://docs.oracle.com/cd/E16338_01/server.112/b56309/basicmv.htm#sthref257)

## 集計を含むマテリアライズド・ビュー

集計(`AVG(sal)` など)を含むマテリアライズド・ビューを高速リフレッシュ可能にする。

マニュアルから、集計関数(`SUM`, `COUNT(*)`, `AVG` など)を含むマテリアライズド・ビューを高速リフレッシュ可能にするためにはいくつか制限がある。
ざっくり含める必要がある要件をまとめると以下。

* マテリアライズド・ビュー・ログの要件
  * マテリアライズド・ビューで参照される列をすべて含める
  * `ROWID` および `INCLUDING NEW VALUES` 句を指定
  * 表の更新を行う場合は `SEQUENCE` 句を指定

* マテリアライズド・ビューの SELECT 文に含める要件
  * `COUNT(*)` を含める
  * `AVG(expr)` などの集計ごとに、対応する `COUNT(expr)` および `SUM(expr)` を含める。([集計関数によって要件は異なる](http://docs.oracle.com/cd/E16338_01/server.112/b56309/basicmv.htm#sthref182))

例えば以下の SELECT を含むマテリアライズド・ビューを考える。

    SELECT d.dname, AVG(e.sal) avg_sal
    FROM emp e, dept d
    WHERE e.deptno = d.deptno
    GROUP BY d.dname;

高速リフレッシュ可能なマテリアライズド・ビューを作成する。

* マテリアライズド・ビュー・ログの作成

`CREATE MATERIALIZED VIEW LOG` 文を実行

    SQL> CREATE MATERIALIZED VIEW LOG ON emp
         WITH SEQUENCE, ROWID
         (sal, deptno)
         INCLUDING NEW VALUES;
    
    Materialized view log created.
    
    SQL> CREATE MATERIALIZED VIEW LOG ON dept
         WITH SEQUENCE, ROWID
         (deptno, dname)
         INCLUDING NEW VALUES;
    
    Materialized view log created.

* マテリアライズド・ビューの作成

`CREATE MATERIALIZED VIEW` 文を実行

    SQL> CREATE MATERIALIZED VIEW emp_dept_mv
         BUILD IMMEDIATE
         REFRESH FAST
         ENABLE QUERY REWRITE
         AS SELECT d.dname, AVG(e.sal) avg_sal,
                   COUNT(*) cnt, COUNT(e.sal) cnt_sal, SUM(e.sal) sum_sal
         FROM emp e, dept d
         WHERE e.deptno = d.deptno
         GROUP BY d.dname;
    
    Materialized view created.

## 高速リフレッシュ可能か確認する方法

`DBMS_MVIEW.EXPLAIN_MVIEW` プロシージャを使用することで簡単にマテリアライズド・ビューに関して次のことを確認できる。

* 高速リフレッシュ可能かどうか
* 実行できるクエリー・リライトのタイプ
* PCT リフレッシュ(パーティション単位での高速リフレッシュ)が可能かどうか(パーティション表でないと意味なし)

使い方は以下

1. `$ORACLE_HOME/rdbms/admin/utlxmv.sql` を流して `MV_CAPABILITIES_TABLE` 表を作成
2. `DBMS_MVIEW.EXPLAIN_MVIEW` プロシージャの引数にマテリアライズド・ビュー名かマテリアライズド・ビューで使用している SELECT 文を与えて実行
3. `MV_CAPABILITIES_TABLE` 表を SELECT して確認
4. 再度 `DBMS_MVIEW.EXPLAIN_MVIEW` プロシージャを実行する場合は `TRUNCATE TABLE MV_CAPABILITIES_TABLE` で結果をクリア

先ほど作成したマテリアライズド・ビューで確認してみる。

    SQL> @?/rdbms/admin/utlxmv.sql

    Table created.

    SQL> EXEC DBMS_MVIEW.EXPLAIN_MVIEW('emp_dept_mv');

    PL/SQL procedure successfully completed.

    SQL> set lines 200
         set pages 100
         col capability_name for a30
         col rel_text for a10
         col msgtxt for a60
    SQL> SELECT capability_name,  possible, SUBSTR(related_text,1,8)
           AS rel_text, SUBSTR(msgtxt,1,60) AS msgtxt
         FROM MV_CAPABILITIES_TABLE
         ORDER BY seq;

    CAPABILITY_NAME                POS REL_TEXT   MSGTXT
    ------------------------------ --- ---------- ------------------------------------------------------------
    PCT                            N
    REFRESH_COMPLETE               Y
    REFRESH_FAST                   Y
    REWRITE                        Y
    PCT_TABLE                      N   EMP        relation is not a partitioned table
    PCT_TABLE                      N   DEPT       relation is not a partitioned table
    REFRESH_FAST_AFTER_INSERT      Y
    REFRESH_FAST_AFTER_ONETAB_DML  Y
    REFRESH_FAST_AFTER_ANY_DML     Y
    REFRESH_FAST_PCT               N              PCT is not possible on any of the detail tables in the mater
    REWRITE_FULL_TEXT_MATCH        Y
    REWRITE_PARTIAL_TEXT_MATCH     Y
    REWRITE_GENERAL                Y
    REWRITE_PCT                    N              general rewrite is not possible or PCT is not possible on an
    PCT_TABLE_REWRITE              N   EMP        relation is not a partitioned table
    PCT_TABLE_REWRITE              N   DEPT       relation is not a partitioned table

    16 rows selected.

`possible` 列が `Y` であることから高速リフレッシュ可能であることが分かる。(`PCT*` に関してはパーティション表ではないので無視)

例えば、以下のようにマテリアライズド・ビュー作成前に高速リフレッシュ可能か確認でき、可能でない場合は理由を表示してくれる。

    SQL> TRUNCATE TABLE MV_CAPABILITIES_TABLE;

    Table truncated.

    SQL> -- わざと COUNT(e.sal), SUM(e.sal) を外してみる
    SQL> BEGIN
           DBMS_MVIEW.EXPLAIN_MVIEW('
             SELECT d.dname, AVG(e.sal) avg_sal,
                    COUNT(*) cnt
             FROM emp e, dept d
             WHERE e.deptno = d.deptno
             GROUP BY d.dname');
         END;
         /

    PL/SQL procedure successfully completed.

    SQL> SELECT capability_name,  possible, SUBSTR(related_text,1,8)
           AS rel_text, SUBSTR(msgtxt,1,60) AS msgtxt
         FROM MV_CAPABILITIES_TABLE
         ORDER BY seq;

    CAPABILITY_NAME                POS REL_TEXT   MSGTXT
    ------------------------------ --- ---------- ------------------------------------------------------------
    PCT                            N
    REFRESH_COMPLETE               Y
    REFRESH_FAST                   N
    REWRITE                        Y
    PCT_TABLE                      N   EMP        relation is not a partitioned table
    PCT_TABLE                      N   DEPT       relation is not a partitioned table
    REFRESH_FAST_AFTER_INSERT      N   AVG_SAL    agg(expr) requires correspondng COUNT(expr) function
    REFRESH_FAST_AFTER_ONETAB_DML  N              see the reason why REFRESH_FAST_AFTER_INSERT is disabled
    REFRESH_FAST_AFTER_ANY_DML     N              see the reason why REFRESH_FAST_AFTER_ONETAB_DML is disabled
    REFRESH_FAST_PCT               N              PCT is not possible on any of the detail tables in the mater
    REWRITE_FULL_TEXT_MATCH        Y
    REWRITE_PARTIAL_TEXT_MATCH     Y
    REWRITE_GENERAL                Y
    REWRITE_PCT                    N              general rewrite is not possible or PCT is not possible on an
    PCT_TABLE_REWRITE              N   EMP        relation is not a partitioned table
    PCT_TABLE_REWRITE              N   DEPT       relation is not a partitioned table

    16 rows selected.

高速リフレッシュが不可で、`AVG_SAL` 列に関して対応する `COUNT(expr)` が必要なことを指摘してくれる。

## 結合のみを含むマテリアライズド・ビュー

集計関数は含まずに結合(`emp.deptno = dept.deptno` など)のみを含むマテリアライズド・ビューを高速リフレッシュ可能にする。

マニュアルから、結合のみを含むマテリアライズド・ビューを高速リフレッシュ可能にするためにはいくつか制限がある。
ざっくり含める必要がある要件をまとめると以下。

* マテリアライズド・ビュー・ログの要件
  * `ROWID` 句を指定

* マテリアライズド・ビューの SELECT 文に含める要件
  * `FROM` 内の全ての表の ROWID を含める

例えば以下の SELECT を含むマテリアライズド・ビューを考える。

    SELECT e.ename, d.loc
    FROM emp e, dept d
    WHERE e.deptno = d.deptno;

高速リフレッシュ可能なマテリアライズド・ビューを作成する。

* マテリアライズド・ビュー・ログの作成

`CREATE MATERIALIZED VIEW LOG` 文を実行

    SQL> CREATE MATERIALIZED VIEW LOG ON emp
         WITH ROWID;
    
    Materialized view log created.
    
    SQL> CREATE MATERIALIZED VIEW LOG ON dept
         WITH ROWID;
    
    Materialized view log created.

* マテリアライズド・ビューの作成

`CREATE MATERIALIZED VIEW` 文を実行

    SQL> CREATE MATERIALIZED VIEW emp_dept_mv
         BUILD IMMEDIATE
         REFRESH FAST
         ENABLE QUERY REWRITE
         AS SELECT e.ename, d.loc,
                   e.rowid e_rowid, d.rowid d_rowid
         FROM emp e, dept d
         WHERE e.deptno = d.deptno;
    
    Materialized view created.

* 高速リフレッシュ可能か確認

先ほど行ったように `DBMS_MVIEW.EXPLAIN_MVIEW` プロシージャで確認する。

    SQL> TRUNCATE TABLE MV_CAPABILITIES_TABLE;

    Table truncated.

    SQL> EXEC DBMS_MVIEW.EXPLAIN_MVIEW('emp_dept_mv');

    PL/SQL procedure successfully completed.

    SQL> SELECT capability_name,  possible, SUBSTR(related_text,1,8)
           AS rel_text, SUBSTR(msgtxt,1,60) AS msgtxt
         FROM MV_CAPABILITIES_TABLE
         ORDER BY seq;

    CAPABILITY_NAME                POS REL_TEXT   MSGTXT
    ------------------------------ --- ---------- ------------------------------------------------------------
    PCT                            N
    REFRESH_COMPLETE               Y
    REFRESH_FAST                   Y
    REWRITE                        Y
    PCT_TABLE                      N   EMP        relation is not a partitioned table
    PCT_TABLE                      N   DEPT       relation is not a partitioned table
    REFRESH_FAST_AFTER_INSERT      Y
    REFRESH_FAST_AFTER_ONETAB_DML  Y
    REFRESH_FAST_AFTER_ANY_DML     Y
    REFRESH_FAST_PCT               N              PCT is not possible on any of the detail tables in the mater
    REWRITE_FULL_TEXT_MATCH        Y
    REWRITE_PARTIAL_TEXT_MATCH     Y
    REWRITE_GENERAL                Y
    REWRITE_PCT                    N              general rewrite is not possible or PCT is not possible on an
    PCT_TABLE_REWRITE              N   EMP        relation is not a partitioned table
    PCT_TABLE_REWRITE              N   DEPT       relation is not a partitioned table

    16 rows selected.

`PCT*` 以外は `possible` 列が `Y` になっているので高速リフレッシュ可能。
