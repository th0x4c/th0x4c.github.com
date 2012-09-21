---
layout: post
title: "[Oracle] スター型変換の使用"
date: 2012-09-13 01:14
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

スター型変換を行い、スター・クエリーをチューニングする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* データ・ウェアハウス・ガイド  
  -> 20 スキーマのモデリング化技法  
  -> [スター・クエリーの最適化](http://docs.oracle.com/cd/E16338_01/server.112/b56309/schemas.htm#CIHGHEFB)

## スター型変換を使用したスター・クエリーのチューニング

スター型変換を行い、スター・クエリーをチューニングする。
スター・クエリーは1つの大規模なファクト表と、複数の小規模なディメンション表を結合するようなクエリー。ファクト表が大きいため、ファクト表をなるべく条件を絞り込んだ後でアクセスするような特別な結合(スター結合)をしたほうが効率がよい。このために暗黙的に SQL をリライト(または変換)する機能をスター型変換という。

スター型変換を行うためには、以下に従う必要がある。

1. ビットマップ索引をファクト表の各外部キー列上に作成する必要がある。
2. 初期化パラメータ `STAR_TRANSFORMATION_ENABLED` を `TRUE` に設定する必要がある。

検証してみる。

* 表、データを準備

ディメンション表 `jobs`, `dates`, ファクト表 `emp_fact` を作成

    $ sqlplus scott/tiger
    SQL> -- ディメンション表 jobs
    SQL> CREATE TABLE jobs
         AS SELECT rownum job_id, job
         FROM (SELECT DISTINCT job FROM emp);

    Table created.

    SQL> ALTER TABLE jobs ADD (CONSTRAINT jobs_pk PRIMARY KEY (job_id) VALIDATE);

    Table altered.

    SQL> -- ディメンション表 dates
    SQL> CREATE TABLE dates
         AS SELECT rownum date_id, hiredate, to_char(hiredate, 'YYYY') year
         FROM (SELECT DISTINCT hiredate FROM emp);

    Table created.

    SQL> ALTER TABLE dates ADD (CONSTRAINT dates_pk PRIMARY KEY (date_id) VALIDATE);

    Table altered.

    SQL> -- ファクト表 emp_fact
    SQL> CREATE TABLE emp_fact
         AS SELECT e.empno, e.ename, j.job_id, e.mgr, d.date_id, e.sal, e.comm, e.deptno
         FROM emp e, dates d, jobs j
         WHERE e.job = j.job AND e.hiredate = d.hiredate;

    Table created.

    SQL> ALTER TABLE emp_fact ADD (CONSTRAINT emp_fact_jobs_fk FOREIGN KEY(job_id) REFERENCES jobs(job_id) VALIDATE);

    Table altered.

    SQL> ALTER TABLE emp_fact ADD (CONSTRAINT emp_fact_dates_fk FOREIGN KEY(date_id) REFERENCES dates(date_id) VALIDATE);

    Table altered.

    SQL> -- データロード
    SQL> INSERT INTO emp_fact SELECT * FROM emp_fact;

    14 rows created.

    SQL> /

    28 rows created.
    ...(以下繰り返す)

    SQL> /

    114688 rows created.

    SQL> COMMIT;

    Commit complete.

    SQL> -- 統計情報取得
    SQL> EXEC DBMS_STATS.GATHER_SCHEMA_STATS('scott');

    PL/SQL procedure successfully completed.

* スター変換の確認方法

スター変換されているかどうかは `set autotrace trace exp` 等の実行計画から確認する。
例えば `set autotrace trace exp` で `Note` に以下の出力があればスター変換されている。

    Note 
    -----
       - star transformation used for this statement

ちなみに `set autotrace` を使用するためには `PLUSTRACE` ロールが必要で、以下のように作成する。

    SQL> connect /as sysdba              
    Connected.
    SQL> @?/sqlplus/admin/plustrce.sql

    SQL> GRANT plustrace TO scott;

    Grant succeeded.

* スター変換なしの場合

以下の SELECT 文で試してみる。

    SQL> set timing on
    SQL> set autotrace on
    SQL> set lines 200
    SQL> SELECT j.job, d.year, SUM(e.sal), d.dname
         FROM emp_fact e, jobs j, dates d, dept d
         WHERE e.job_id = j.job_id
         AND   e.date_id = d.date_id
         AND   e.deptno = d.deptno
         AND   j.job = 'CLERK'
         AND   d.year in ('1980', '1987')
         GROUP BY j.job, d.year, d.dname;

    JOB                         YEAR         SUM(E.SAL) DNAME
    --------------------------- ------------ ---------- ------------------------------------------
    CLERK                       1987           18022400 RESEARCH
    CLERK                       1980           13107200 RESEARCH

    Elapsed: 00:00:00.04

    Execution Plan
    ----------------------------------------------------------
    Plan hash value: 3373499036

    ------------------------------------------------------------------------------------
    | Id  | Operation               | Name     | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT        |          |     4 |   180 |   324   (3)| 00:00:04 |
    |   1 |  HASH GROUP BY          |          |     4 |   180 |   324   (3)| 00:00:04 |
    |*  2 |   HASH JOIN             |          | 22938 |  1008K|   323   (2)| 00:00:04 |
    |   3 |    TABLE ACCESS FULL    | DEPT     |     4 |    52 |     3   (0)| 00:00:01 |
    |*  4 |    HASH JOIN            |          | 22938 |   716K|   319   (2)| 00:00:04 |
    |   5 |     MERGE JOIN CARTESIAN|          |     7 |   133 |     6   (0)| 00:00:01 |
    |*  6 |      TABLE ACCESS FULL  | JOBS     |     1 |    11 |     3   (0)| 00:00:01 |
    |   7 |      BUFFER SORT        |          |     7 |    56 |     3   (0)| 00:00:01 |
    |*  8 |       TABLE ACCESS FULL | DATES    |     7 |    56 |     3   (0)| 00:00:01 |
    |   9 |     TABLE ACCESS FULL   | EMP_FACT |   229K|  2912K|   312   (2)| 00:00:04 |
    ------------------------------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------

       2 - access("E"."DEPTNO"="D"."DEPTNO")
       4 - access("E"."JOB_ID"="J"."JOB_ID" AND "E"."DATE_ID"="D"."DATE_ID")
       6 - filter("J"."JOB"='CLERK')
       8 - filter("D"."YEAR"='1980' OR "D"."YEAR"='1987')


    Statistics
    ----------------------------------------------------------
              0  recursive calls
              1  db block gets
           1098  consistent gets
              0  physical reads
              0  redo size
            824  bytes sent via SQL*Net to client
            524  bytes received via SQL*Net from client
              2  SQL*Net roundtrips to/from client
              1  sorts (memory)
              0  sorts (disk)
              2  rows processed

0.04 秒かかっている。

* スター型変換

スター型変換するようにビットマップ索引を作成し、初期化パラメータ`ALTER SESSION SET STAR_TRANSFORMATION_ENABLED` を `TRUE` にして再実行する。

    SQL> -- ビットマップ索引を作成
    SQL> CREATE BITMAP INDEX emp_fact_job_bix ON emp_fact(job_id);

    Index created.

    SQL> CREATE BITMAP INDEX emp_fact_date_bix ON emp_fact(date_id);

    Index created.

    SQL> CREATE BITMAP INDEX emp_fact_dept_bix ON emp_fact(deptno);

    Index created.

    SQL> -- 統計情報採取
    SQL> EXEC DBMS_STATS.GATHER_SCHEMA_STATS('scott');

    PL/SQL procedure successfully completed.

    SQL> -- 初期化パラメータ変更
    SQL> SQL> ALTER SESSION SET STAR_TRANSFORMATION_ENABLED = TRUE

    Session altered.

先ほどと同じ SELECT を実行する。

    SQL> set timing on
    SQL> set autotrace on
    SQL> set lines 200
    SQL> SELECT j.job, d.year, SUM(e.sal), d.dname
         FROM emp_fact e, jobs j, dates d, dept d
         WHERE e.job_id = j.job_id
         AND   e.date_id = d.date_id
         AND   e.deptno = d.deptno
         AND   j.job = 'CLERK'
         AND   d.year in ('1980', '1987')
         GROUP BY j.job, d.year, d.dname;

    JOB                         YEAR         SUM(E.SAL) DNAME
    --------------------------- ------------ ---------- ------------------------------------------
    CLERK                       1987           18022400 RESEARCH
    CLERK                       1980           13107200 RESEARCH

    Elapsed: 00:00:00.02

    Execution Plan
    ----------------------------------------------------------
    Plan hash value: 4243144607

    ------------------------------------------------------------------------------------------------------
    | Id  | Operation                        | Name              | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT                 |                   |     4 |   216 |   322   (1)| 00:00:04 |
    |   1 |  HASH GROUP BY                   |                   |     4 |   216 |   322   (1)| 00:00:04 |
    |*  2 |   HASH JOIN                      |                   |  8823 |   465K|   321   (1)| 00:00:04 |
    |   3 |    TABLE ACCESS FULL             | DEPT              |     4 |    52 |     3   (0)| 00:00:01 |
    |*  4 |    HASH JOIN                     |                   |  8823 |   353K|   318   (1)| 00:00:04 |
    |   5 |     MERGE JOIN CARTESIAN         |                   |     3 |    57 |     6   (0)| 00:00:01 |
    |*  6 |      TABLE ACCESS FULL           | JOBS              |     1 |    11 |     3   (0)| 00:00:01 |
    |   7 |      BUFFER SORT                 |                   |     3 |    24 |     3   (0)| 00:00:01 |
    |*  8 |       TABLE ACCESS FULL          | DATES             |     3 |    24 |     3   (0)| 00:00:01 |
    |   9 |     VIEW                         | VW_ST_7A68B670    | 10587 |   227K|   311   (0)| 00:00:04 |
    |  10 |      NESTED LOOPS                |                   | 10587 |   444K|   305   (0)| 00:00:04 |
    |  11 |       BITMAP CONVERSION TO ROWIDS|                   | 10586 |   186K|     6   (0)| 00:00:01 |
    |  12 |        BITMAP AND                |                   |       |       |            |          |
    |  13 |         BITMAP MERGE             |                   |       |       |            |          |
    |  14 |          BITMAP KEY ITERATION    |                   |       |       |            |          |
    |* 15 |           TABLE ACCESS FULL      | JOBS              |     1 |    11 |     3   (0)| 00:00:01 |
    |* 16 |           BITMAP INDEX RANGE SCAN| EMP_FACT_JOB_BIX  |       |       |            |          |
    |  17 |         BITMAP MERGE             |                   |       |       |            |          |
    |  18 |          BITMAP KEY ITERATION    |                   |       |       |            |          |
    |* 19 |           TABLE ACCESS FULL      | DATES             |     3 |    24 |     3   (0)| 00:00:01 |
    |* 20 |           BITMAP INDEX RANGE SCAN| EMP_FACT_DATE_BIX |       |       |            |          |
    |  21 |       TABLE ACCESS BY USER ROWID | EMP_FACT          |     1 |    25 |   305   (1)| 00:00:04 |
    ------------------------------------------------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------

       2 - access("ITEM_1"="D"."DEPTNO")
       4 - access("ITEM_3"="J"."JOB_ID" AND "ITEM_2"="D"."DATE_ID")
       6 - filter("J"."JOB"='CLERK')
       8 - filter("D"."YEAR"='1980' OR "D"."YEAR"='1987')
      15 - filter("J"."JOB"='CLERK')
      16 - access("E"."JOB_ID"="J"."JOB_ID")
      19 - filter("D"."YEAR"='1980' OR "D"."YEAR"='1987')
      20 - access("E"."DATE_ID"="D"."DATE_ID")

    Note
    -----
       - star transformation used for this statement


    Statistics
    ----------------------------------------------------------
              1  recursive calls
              0  db block gets
           1075  consistent gets
              0  physical reads
              0  redo size
            824  bytes sent via SQL*Net to client
            524  bytes received via SQL*Net from client
              2  SQL*Net roundtrips to/from client
              1  sorts (memory)
              0  sorts (disk)
              2  rows processed

今回は、`star transformation used for this statement` の出力があるようにスター型変換されており、実行時間も 0.02秒と改善されている。
