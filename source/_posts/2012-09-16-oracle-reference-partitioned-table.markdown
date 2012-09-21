---
layout: post
title: "[Oracle] 参照パーティション表"
date: 2012-09-16 16:13
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

参照パーティション表を作成する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* VLDBおよびパーティショニング・ガイド  
  -> 4 パーティションの管理  
  -> [参照パーティション表の作成](http://docs.oracle.com/cd/E16338_01/server.112/b56316/part_admin001.htm#BAJDDEEC)

## 参照パーティション表の作成

参照パーティション表は、例えば次のようなケースで役立つ。注文を管理する orders 表に注文日が格納される order_date 列がある。また、注文に含まれる品目を管理する order_items 表があり、orders 表を order_id 列で外部参照しているとする。orders 表が order_date 列をキー値としてパーティション化されている場合、order_items 表も同じように注文日によりパーティション化したかったら、従来ならば order_date 列を order_itmes 表にも加える必要があった。参照パーティション表を使えば、order_items 列に余計な order_date 列を加えることなく、外部参照している orders と同じ単位でパーティション化することができる。

実際に上記シナリオで参照パーティション表を作成してみる。

    SQL> CREATE TABLE orders
             ( order_id           NUMBER(12),
               order_date         DATE,
               order_mode         VARCHAR2(8),
               customer_id        NUMBER(6),
               order_status       NUMBER(2),
               order_total        NUMBER(8,2),
               sales_rep_id       NUMBER(6),
               promotion_id       NUMBER(6),
               CONSTRAINT orders_pk PRIMARY KEY(order_id)
             )
           PARTITION BY RANGE(order_date)
             ( PARTITION Q1_2005 VALUES LESS THAN (TO_DATE('01-APR-2005','DD-MON-YYYY')),
               PARTITION Q2_2005 VALUES LESS THAN (TO_DATE('01-JUL-2005','DD-MON-YYYY')),
               PARTITION Q3_2005 VALUES LESS THAN (TO_DATE('01-OCT-2005','DD-MON-YYYY')),
               PARTITION Q4_2005 VALUES LESS THAN (TO_DATE('01-JAN-2006','DD-MON-YYYY'))
             );

    Table created.

    SQL> CREATE TABLE order_items
             ( order_id           NUMBER(12) NOT NULL,
               line_item_id       NUMBER(3)  NOT NULL,
               product_id         NUMBER(6)  NOT NULL,
               unit_price         NUMBER(8,2),
               quantity           NUMBER(8),
               CONSTRAINT order_items_fk
               FOREIGN KEY(order_id) REFERENCES orders(order_id)
             )
             PARTITION BY REFERENCE(order_items_fk);
    
    Table created. 

パーティション情報の確認。

    SQL> col table_name for a30
         col partitioning_type for a20
         col ref_ptn_constraint_name for a30
         set lines 200
    SQL> SELECT table_name, partitioning_type, ref_ptn_constraint_name FROM user_part_tables;

    TABLE_NAME                     PARTITIONING_TYPE    REF_PTN_CONSTRAINT_NAME
    ------------------------------ -------------------- ------------------------------
    ORDERS                         RANGE
    ORDER_ITEMS                    REFERENCE            ORDER_ITEMS_FK

    SQL> col partition_name for a30
    SQL> SELECT table_name, partition_name, high_value FROM user_tab_partitions ORDER BY table_name, partition_position;

    TABLE_NAME                     PARTITION_NAME                 HIGH_VALUE
    ------------------------------ ------------------------------ --------------------------------------------------------------------------------
    ORDERS                         Q1_2005                        TO_DATE(' 2005-04-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    ORDERS                         Q2_2005                        TO_DATE(' 2005-07-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    ORDERS                         Q3_2005                        TO_DATE(' 2005-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    ORDERS                         Q4_2005                        TO_DATE(' 2006-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIA
    ORDER_ITEMS                    Q1_2005
    ORDER_ITEMS                    Q2_2005
    ORDER_ITEMS                    Q3_2005
    ORDER_ITEMS                    Q4_2005

