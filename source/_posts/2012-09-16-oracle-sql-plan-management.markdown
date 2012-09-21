---
layout: post
title: "[Oracle] SQL 計画管理"
date: 2012-09-16 23:28
comments: true
categories: [Oracle, DB, 11gR2, Performance Tuning]
---
## 目的

SQL 計画管理により、実行計画を固定化する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* パフォーマンス・チューニング・ガイド  
  -> 15 SQL計画の管理の使用方法  
  -> [15.2 SQL計画ベースラインの管理](http://docs.oracle.com/cd/E16338_01/server.112/b56312/optplanmgmt.htm#BABEGJGB)

## SQL 計画管理

SQL 計画管理により、実行計画を固定化することができる。具体的には次のステップで行う。

1. SQL 計画の自動取得(`OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES=TRUE`)
2. 実行計画を管理する SQL を 2 回以上実行
3. 取得した SQL 計画の使用(`OPTIMIZER_USE_SQL_PLAN_BASELINES=TRUE`)

実際に実行してみる。

* SQL 計画の自動取得(`OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES=TRUE`)

`OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES` 初期化パラメータ(デフォルト `FALSE`)を `TRUE` に設定することで SQL 計画の自動取得が有効になる。

    SQL> ALTER SESSION SET OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES=TRUE;

    Session altered.

* SQL を 2 回以上実行

SQL を 2 回以上実行して、SQL 計画ベースラインに保存する。

    SQL> SELECT * FROM scott.emp WHERE empno = 7900;

         EMPNO ENAME                          JOB                                MGR HIREDATE        SAL       COMM     DEPTNO
    ---------- ------------------------------ --------------------------- ---------- -------- ---------- ---------- ----------
          7900 JAMES                          CLERK                             7698 81-12-03        960                    30

    SQL> SELECT * FROM scott.emp WHERE empno = 7900;

         EMPNO ENAME                          JOB                                MGR HIREDATE        SAL       COMM     DEPTNO
    ---------- ------------------------------ --------------------------- ---------- -------- ---------- ---------- ----------
          7900 JAMES                          CLERK                             7698 81-12-03        960                    30

* SQL 計画の自動取得を無効化(`OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES=FALSE`)

SQL 計画の自動取得を無効化する。

    SQL> ALTER SESSION SET OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES=FALSE;

    Session altered.

* SQL が SQL 計画ベースラインに保存されたことを確認

`DBA_SQL_PLAN_BASELINES` を確認する。

    SQL> col sql_text for a60
         col sql_handle for a30
         col plan_name for a30
         set lines 200
    SQL> SELECT SQL_TEXT, SQL_HANDLE, PLAN_NAME, ENABLED, ACCEPTED, FIXED
         FROM   DBA_SQL_PLAN_BASELINES;

    SQL_TEXT                                                     SQL_HANDLE                     PLAN_NAME                      ENABLED   ACCEPTED  FIXED
    ------------------------------------------------------------ ------------------------------ ------------------------------ --------- --------- ---------
    SELECT * FROM scott.emp WHERE empno = 7900                   SQL_84ec680ef31d6de4           SQL_PLAN_89v381vtjuvg4695cc014 YES       YES       NO

実行計画を確認する。

    SQL> SELECT * FROM TABLE(
           DBMS_XPLAN.DISPLAY_SQL_PLAN_BASELINE(
             sql_handle=>'SQL_84ec680ef31d6de4',
             format=>'basic'));

    PLAN_TABLE_OUTPUT
    --------------------------------------------------------------------------------

    --------------------------------------------------------------------------------
    SQL handle: SQL_84ec680ef31d6de4
    SQL text: SELECT * FROM scott.emp WHERE empno = 7900
    --------------------------------------------------------------------------------

    --------------------------------------------------------------------------------
    Plan name: SQL_PLAN_89v381vtjuvg4695cc014         Plan id: 1767686164
    Enabled: YES     Fixed: NO      Accepted: YES     Origin: AUTO-CAPTURE
    --------------------------------------------------------------------------------


    PLAN_TABLE_OUTPUT
    ----------------------------------------------
    Plan hash value: 2949544139

    ----------------------------------------------
    | Id  | Operation                   | Name   |
    ----------------------------------------------
    |   0 | SELECT STATEMENT            |        |
    |   1 |  TABLE ACCESS BY INDEX ROWID| EMP    |
    |   2 |   INDEX UNIQUE SCAN         | PK_EMP |
    ----------------------------------------------

    20 rows selected.

* 取得した SQL 計画の使用(`OPTIMIZER_USE_SQL_PLAN_BASELINES=TRUE`)

`OPTIMIZER_USE_SQL_PLAN_BASELINES` 初期化パラメータ(デフォルト `TRUE`)を `TRUE` に設定することで取得した SQL 計画を使用する。

    SQL> ALTER SESSION SET OPTIMIZER_USE_SQL_PLAN_BASELINES=TRUE;

    Session altered.

あとは普通に SQL を時刻するだけで取得した SQL 計画が使用される。

    SQL> set autotrace on
    SQL> SELECT * FROM scott.emp WHERE empno = 7900;

         EMPNO ENAME                          JOB                                MGR HIREDATE        SAL       COMM     DEPTNO
    ---------- ------------------------------ --------------------------- ---------- -------- ---------- ---------- ----------
          7900 JAMES                          CLERK                             7698 81-12-03        960                    30


    Execution Plan
    ----------------------------------------------------------
    Plan hash value: 2949544139

    --------------------------------------------------------------------------------------
    | Id  | Operation                   | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
    --------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT            |        |     1 |    39 |     1   (0)| 00:00:01 |
    |   1 |  TABLE ACCESS BY INDEX ROWID| EMP    |     1 |    39 |     1   (0)| 00:00:01 |
    |*  2 |   INDEX UNIQUE SCAN         | PK_EMP |     1 |       |     0   (0)| 00:00:01 |
    --------------------------------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------

       2 - access("EMPNO"=7900)

    Note 
    -----
       - SQL plan baseline "SQL_PLAN_89v381vtjuvg4695cc014" used for this statement


    Statistics
    ----------------------------------------------------------
              1  recursive calls
              0  db block gets
              2  consistent gets
              0  physical reads
              0  redo size
            889  bytes sent via SQL*Net to client
            512  bytes received via SQL*Net from client
              1  SQL*Net roundtrips to/from client
              0  sorts (memory)
              0  sorts (disk)
              1  rows processed

Note の `SQL plan baseline "SQL_PLAN_89v381vtjuvg4695cc014" used for this statement` という出力から取得した SQL 計画が使用されていることが分かる。
