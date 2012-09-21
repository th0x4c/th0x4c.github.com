---
layout: post
title: "[Oracle] 複数列の統計"
date: 2012-09-16 20:21
comments: true
categories: [Oracle, DB, 11gR2, Performance Tuning]
---
## 目的

相関のある複数列を列グループとして統計情報採取することでよりよい実行計画となるようにする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* パフォーマンス・チューニング・ガイド  
  -> 13 オプティマイザ統計の管理  
  -> [13.3.1.6 複数列の統計](http://docs.oracle.com/cd/E16338_01/server.112/b56312/stats.htm#CIHFICCB)

## 複数列の統計収集

相関のある複数列を列グループとして統計情報を採取することで、オプティマイザが個々の列でなく列グループとして実行計画を計算するようにする。

* 検証で使用する表の作成

C1 列、C2 列が同じになる(つまり相関がある)ような表を作成する。

    SQL> CREATE TABLE multi_col_tab (c1 NUMBER, c2 NUMBER);
    
    Table created.
    
    SQL> BEGIN
           FOR i IN 1..5 LOOP
             FOR j IN 1..100 LOOP
               INSERT INTO multi_col_tab VALUES(i, i);
             END LOOP;
           END LOOP;
           COMMIT;
         END;
         /

    PL/SQL procedure successfully completed.

1 〜 5 までそれぞれ 100 行ずつ、計 500 行を INSERT。

* 複数列統計を採取しない場合

複数列統計を採取しない場合の実行計画

    SQL> BEGIN
           DBMS_STATS.GATHER_TABLE_STATS(null,'MULTI_COL_TAB',
             METHOD_OPT => 'FOR ALL COLUMNS SIZE SKEWONLY');
         END;
         /

    PL/SQL procedure successfully completed.

    SQL> set autotrace on
    SQL> SELECT COUNT(*) FROM multi_col_tab WHERE c1 = 1 AND c2 = 1;


      COUNT(*)
    ----------
           100


    Execution Plan
    ----------------------------------------------------------
    Plan hash value: 610516676

    ------------------------------------------------------------------------------------
    | Id  | Operation          | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT   |               |     1 |     6 |     3   (0)| 00:00:01 |
    |   1 |  SORT AGGREGATE    |               |     1 |     6 |            |          |
    |*  2 |   TABLE ACCESS FULL| MULTI_COL_TAB |    20 |   120 |     3   (0)| 00:00:01 |
    ------------------------------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------

       2 - filter("C1"=1 AND "C2"=1)


    Statistics
    ----------------------------------------------------------
              1  recursive calls
              0  db block gets
              6  consistent gets
              0  physical reads
              0  redo size
            526  bytes sent via SQL*Net to client
            523  bytes received via SQL*Net from client
              2  SQL*Net roundtrips to/from client
              0  sorts (memory)
              0  sorts (disk)
              1  rows processed

実行計画の `MULTI_COL_TAB` の出力行に注目

    ------------------------------------------------------------------------------------
    | Id  | Operation          | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT   |               |     1 |     6 |     3   (0)| 00:00:01 |
    |   1 |  SORT AGGREGATE    |               |     1 |     6 |            |          |
    |*  2 |   TABLE ACCESS FULL| MULTI_COL_TAB |    20 |   120 |     3   (0)| 00:00:01 |
    ------------------------------------------------------------------------------------

`Rows` が 20 となっている。
条件が `WHERE c1 = 1 AND c2 = 1` だが、相関があることを知らないので、選択行が `500行 * 1/5(c1 の選択率) * 1/5(c2 の選択率)` = 20行 となっていて正確でない。

* 複数列統計を採取した場合

複数列統計を採取する。`c1` および `c2` 列で構成させる列グループを追加して、統計情報取得。

    SQL> DECLARE
           cg_name varchar2(30);
         BEGIN
           cg_name := dbms_stats.create_extended_stats(null,'multi_col_tab',  
                     '(c1,c2)');
         END;
         /

    PL/SQL procedure successfully completed.

    SQL> BEGIN
           DBMS_STATS.GATHER_TABLE_STATS(null,'MULTI_COL_TAB',
             METHOD_OPT => 'FOR ALL COLUMNS SIZE SKEWONLY');
         END;
         /
    
    PL/SQL procedure successfully completed.

なお、`DBMS_STATS.CREATE_EXTENDED_STATS` で明示的に列グループを作成しない場合は、以下のように統計情報収集時に `METHOD_OPT` で複数列を指定してもよい。

    SQL> BEGIN
           DBMS_STATS.GATHER_TABLE_STATS(null,'MULTI_COL_TAB',
             METHOD_OPT => 'FOR ALL COLUMNS SIZE SKEWONLY FOR COLUMNS (C1,C2) SIZE SKEWONLY');
         END;
         /
    
    PL/SQL procedure successfully completed.

先ほどと同じ SQL で実行計画を確認してみる。

    SQL> set autotrace on
    SQL> SELECT COUNT(*) FROM multi_col_tab WHERE c1 = 1 AND c2 = 1;

      COUNT(*)
    ----------
           100


    Execution Plan
    ----------------------------------------------------------
    Plan hash value: 610516676

    ------------------------------------------------------------------------------------
    | Id  | Operation          | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT   |               |     1 |     6 |     3   (0)| 00:00:01 |
    |   1 |  SORT AGGREGATE    |               |     1 |     6 |            |          |
    |*  2 |   TABLE ACCESS FULL| MULTI_COL_TAB |   100 |   600 |     3   (0)| 00:00:01 |
    ------------------------------------------------------------------------------------

    Predicate Information (identified by operation id):
    ---------------------------------------------------

       2 - filter("C1"=1 AND "C2"=1)


    Statistics
    ----------------------------------------------------------
              1  recursive calls
              0  db block gets
              6  consistent gets
              0  physical reads
              0  redo size
            526  bytes sent via SQL*Net to client
            523  bytes received via SQL*Net from client
              2  SQL*Net roundtrips to/from client
              0  sorts (memory)
              0  sorts (disk)
              1  rows processed

実行計画の `MULTI_COL_TAB` の出力行に注目

    ------------------------------------------------------------------------------------
    | Id  | Operation          | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT   |               |     1 |     6 |     3   (0)| 00:00:01 |
    |   1 |  SORT AGGREGATE    |               |     1 |     6 |            |          |
    |*  2 |   TABLE ACCESS FULL| MULTI_COL_TAB |   100 |   600 |     3   (0)| 00:00:01 |
    ------------------------------------------------------------------------------------

`Rows` が 100 となっている。
条件が `WHERE c1 = 1 AND c2 = 1` で相関があることを認識して正確に 100 行と計算している。

* 複数列統計の確認

拡張統計として、複数列が認識されていることを確認。

    SQL> col extension_name for a30
         col extension for a30
    SQL> Select extension_name, extension 
         from user_stat_extensions 
         where table_name='MULTI_COL_TAB';

    EXTENSION_NAME                 EXTENSION
    ------------------------------ ------------------------------
    SYS_STUF3GLKIOP5F4B0BTTCFTMX0W ("C1","C2")

実際に複数列の統計が採られていることを確認

    SQL> col col_group for a30
    SQL> select e.extension col_group, t.num_distinct, t.histogram
         from user_stat_extensions e, user_tab_col_statistics t
         where e.extension_name=t.column_name
         and e.table_name=t.table_name
         and t.table_name='MULTI_COL_TAB';

    COL_GROUP                      NUM_DISTINCT HISTOGRAM
    ------------------------------ ------------ ---------------------------------------------
    ("C1","C2")                               5 FREQUENCY

`HISTGRAM` 列が `NONE` 以外であればヒストグラムが採られている。

