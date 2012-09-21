---
layout: post
title: "[Oracle] 結果キャッシュ"
date: 2012-09-16 18:40
comments: true
categories: [Oracle, DB, 11gR2, Performance Tuning]
---
## 目的

"結果キャッシュ" 機能を使用する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* パフォーマンス・チューニング・ガイド  
  -> 7 メモリーの構成および使用方法  
  -> [7.6 サーバーおよびクライアントの結果キャッシュの管理](http://docs.oracle.com/cd/E16338_01/server.112/b56312/memory.htm#BGBCABED)

## 結果キャッシュの使用

"結果キャッシュ" 機能を使用することで、問い合わせ結果を再利用してパフォーマンスを向上させることができる。

* 初期化パラメータの設定

結果キャッシュを有効にするためには初期化パラメータ `RESULT_CACHE_MAX_SIZE` を 0 より大きくする必要がある。

0 の場合は結果キャッシュは無効となる。(以下のように `DBMS_RESULT_CACHE.STATUS` の結果が `DISABLED` となっている。)

    SQL> show parameter result_cache_max_size

    NAME                                 TYPE                              VALUE
    ------------------------------------ --------------------------------- ------------------------------
    result_cache_max_size                big integer                       0
    SQL> SELECT DBMS_RESULT_CACHE.STATUS FROM DUAL;

    STATUS
    ----------
    DISABLED

初期化パラメータ `RESULT_CACHE_MAX_SIZE` は以下のように動的に変更できるが、実際はインスタンスを再起動しないと有効にならなかった。

    SQL> ALTER SYSTEM SET result_cache_max_size = 15M SCOPE=both;

    System altered.

    SQL> show parameter result_cache_max_size

    NAME                                 TYPE                              VALUE
    ------------------------------------ --------------------------------- ------------------------------
    result_cache_max_size                big integer                       15M
    SQL> SELECT DBMS_RESULT_CACHE.STATUS FROM DUAL;

    STATUS
    ----------
    DISABLED

インスタンスを再起動すると、有効になり、`DBMS_RESULT_CACHE.STATUS` も `ENABLED` に変わる。

    SQL> shutdown immediate
    Database closed.
    Database dismounted.
    ORACLE instance shut down.
    SQL> startup
    ORACLE instance started.

    Total System Global Area  835104768 bytes
    Fixed Size                  2232960 bytes
    Variable Size             734006656 bytes
    Database Buffers           92274688 bytes
    Redo Buffers                6590464 bytes
    Database mounted.
    Database opened.
    SQL> show parameter result_cache_max_size

    NAME                                 TYPE                              VALUE
    ------------------------------------ --------------------------------- ------------------------------
    result_cache_max_size                big integer                       15M
    SQL> SELECT DBMS_RESULT_CACHE.STATUS FROM DUAL;

    STATUS
    ----------
    ENABLED

* 結果キャッシュの使用

結果キャッシュを使用するには `/*+ RESULT_CACHE */` ヒントを付与して問い合わせを実行すればよい。

    SQL> SELECT /*+ RESULT_CACHE */ AVG(sal) FROM scott.emp;

* 結果キャッシュの確認

実行計画で、`RESULT CACHE` と出ていると結果キャッシュが使われている。

    SQL> set lines 200
    SQL> set autotrace on
    SQL> SELECT /*+ RESULT_CACHE */ AVG(sal) FROM scott.emp;

      AVG(SAL)
    ----------
    2083.21429


    Execution Plan
    ----------------------------------------------------------
    Plan hash value: 2083865914

    --------------------------------------------------------------------------------------------------
    | Id  | Operation           | Name                       | Rows  | Bytes | Cost (%CPU)| Time     |
    --------------------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT    |                            |     1 |     4 |     3   (0)| 00:00:01 |
    |   1 |  RESULT CACHE       | 4z1ag1pqa9zqm704zf4nuj9tq4 |       |       |            |          |
    |   2 |   SORT AGGREGATE    |                            |     1 |     4 |            |          |
    |   3 |    TABLE ACCESS FULL| EMP                        |    14 |    56 |     3   (0)| 00:00:01 |
    --------------------------------------------------------------------------------------------------

    Result Cache Information (identified by operation id):
    ------------------------------------------------------

       1 - column-count=1; dependencies=(SCOTT.EMP); attributes=(single-row); name="SELECT /*+ RESULT_CACHE */ AVG(sal) FROM scott.emp"


    Statistics
    ----------------------------------------------------------
              0  recursive calls
              0  db block gets
              0  consistent gets
              0  physical reads
              0  redo size
            545  bytes sent via SQL*Net to client
            523  bytes received via SQL*Net from client
              2  SQL*Net roundtrips to/from client
              0  sorts (memory)
              0  sorts (disk)
              1  rows processed

また、`V$RESULT_CACHE_OBJECTS` からも結果キャッシュについて確認できる。

    SQL> SELECT type, status, name FROM v$result_cache_objects;

    TYPE                           STATUS
    ------------------------------ ---------------------------
    NAME
    --------------------------------------------------------------------------------
    Dependency                     Published
    SCOTT.EMP

    Result                         Published
    SELECT /*+ RESULT_CACHE */ AVG(sal) FROM scott.emp

