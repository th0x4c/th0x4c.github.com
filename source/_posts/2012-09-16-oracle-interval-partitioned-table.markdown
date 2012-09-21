---
layout: post
title: "[Oracle] 時間隔パーティション表"
date: 2012-09-16 16:46
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

時間隔パーティション表を作成する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* VLDBおよびパーティショニング・ガイド  
  -> 4 パーティションの管理  
  -> [時間隔パーティション表の作成](http://docs.oracle.com/cd/E16338_01/server.112/b56316/part_admin001.htm#BAJHFFBE)

## 時間隔パーティション表の作成

時間隔パーティション表はレンジ・パーティションの一種で、例えば月単位の時間隔パーティション表を作成すると、パーティションを明示的に作成しなくても自動で必要な月のパーティションをデータ insert 時に作成してくれる。

`DATE` 型の `time_id` をパーティション・キーとして、2009年までは年単位でパーティション化し、2010年1月1日以降は月単位でパーティションを自動作成するような時間隔パーティション表を作成してみる。

    SQL> CREATE TABLE interval_sales
             ( prod_id        NUMBER(6)
             , cust_id        NUMBER
             , time_id        DATE
             , channel_id     CHAR(1)
             , promo_id       NUMBER(6)
             , quantity_sold  NUMBER(3)
             , amount_sold    NUMBER(10,2)
             ) 
           PARTITION BY RANGE (time_id) 
           INTERVAL(NUMTOYMINTERVAL(1, 'MONTH'))
             ( PARTITION p0 VALUES LESS THAN (TO_DATE('1-1-2008', 'DD-MM-YYYY')),
               PARTITION p1 VALUES LESS THAN (TO_DATE('1-1-2009', 'DD-MM-YYYY')),
               PARTITION p2 VALUES LESS THAN (TO_DATE('1-1-2010', 'DD-MM-YYYY')) );
      
    Table created.

ちなみに追加されるパーティションを日単位にする場合は、`NUMTODSINTERVAL(1, 'DAY')` 関数を利用する。

パーティション情報の確認。

    SQL> col table_name for a30
         col partitioning_type for a20
         set lines 200
    SQL> SELECT table_name, partitioning_type FROM user_part_tables;

    TABLE_NAME                     PARTITIONING_TYPE   
    ------------------------------ --------------------
    INTERVAL_SALES                 RANGE

    SQL> col partition_name for a30
    SQL> SELECT table_name, partition_name, high_value FROM user_tab_partitions ORDER BY table_name, partition_position;
    
    TABLE_NAME                     PARTITION_NAME                 HIGH_VALUE
    ------------------------------ ------------------------------ --------------------------------------------------------------------------------
    INTERVAL_SALES                 P0                             TO_DATE(' 2008-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    INTERVAL_SALES                 P1                             TO_DATE(' 2009-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    INTERVAL_SALES                 P2                             TO_DATE(' 2010-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA

最初はパーティションは作られていないが、上記の範囲を超える行を insert するとパーティションが自動で作成される。

    SQL> INSERT INTO interval_sales(prod_id, time_id) VALUES(1, TO_DATE('2012-09-16', 'YYYY-MM-DD'));

    1 row created.

    SQL> COMMIT;

    Commit complete.

    SQL> SELECT table_name, partition_name, high_value FROM user_tab_partitions ORDER BY table_name, partition_position;

    TABLE_NAME                     PARTITION_NAME                 HIGH_VALUE
    ------------------------------ ------------------------------ --------------------------------------------------------------------------------
    INTERVAL_SALES                 P0                             TO_DATE(' 2008-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    INTERVAL_SALES                 P1                             TO_DATE(' 2009-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    INTERVAL_SALES                 P2                             TO_DATE(' 2010-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    INTERVAL_SALES                 SYS_P41                        TO_DATE(' 2012-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA

`SYS_P41` というパーティションが自動で作成された。
