---
layout: post
title: "[Oracle] RMAN の構成"
date: 2012-09-09 01:47
comments: true
categories: [Oracle, DB, 11gR2, Backup and Recovery]
---
## 目的

RMAN の構成を変更して、以下ができるようにする。

* 制御ファイルと SPFILE の自動バックアップ取得
* バックアップの最適化(保存方針によって同一ファイルの不要なバックアップをしない)
* ブロック・チェンジ・トラッキングを有効化(DB で変更されたブロックを記録しておくことで、増分バックアップ時のパフォーマンスが向上する。)

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* バックアップおよびリカバリ・ユーザーズ・ガイド  
  -> 5 Recovery Manager環境の構成  
  -> [Recovery Managerの永続的な構成の表示およびクリア](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmconfb.htm#i1019739)  
     [制御ファイルおよびサーバー・パラメータ・ファイルの自動バックアップの構成](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmconfb.htm#i1017907)  
     [バックアップの最適化の構成](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmconfb.htm#sthref449)  
  
  -> 9 データベースのバックアップ  
  -> [ブロック・チェンジ・トラッキングの有効化および無効化](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmbckba.htm#sthref883)

## RMAN 構成の確認

現在の RMAN 構成は `show all` にて確認できる。

    $ rman target / catalog rman/rman@EMREP
    RMAN> show all;

    RMAN configuration parameters for database with db_unique_name PROD1 are:
    CONFIGURE RETENTION POLICY TO REDUNDANCY 1; # default
    CONFIGURE BACKUP OPTIMIZATION OFF; # default
    CONFIGURE DEFAULT DEVICE TYPE TO DISK; # default
    CONFIGURE CONTROLFILE AUTOBACKUP OFF; # default
    CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '%F'; # default
    CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO BACKUPSET; # default
    CONFIGURE DATAFILE BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
    CONFIGURE ARCHIVELOG BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
    CONFIGURE MAXSETSIZE TO UNLIMITED; # default
    CONFIGURE ENCRYPTION FOR DATABASE OFF; # default
    CONFIGURE ENCRYPTION ALGORITHM 'AES128'; # default
    CONFIGURE COMPRESSION ALGORITHM 'BASIC' AS OF RELEASE 'DEFAULT' OPTIMIZE FOR LOAD TRUE ; # default
    CONFIGURE ARCHIVELOG DELETION POLICY TO NONE; # default
    CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/u01/app/oracle/product/11.2.0/dbhome_1/dbs/snapcf_PROD1.f'; # default

## 制御ファイル, SPFILE の自動バックアップ

`BACKUP` コマンドが実行されるときに、制御ファイルと SPFILE も自動的にバックアップするように設定する。

    RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;

    new RMAN configuration parameters:
    CONFIGURE CONTROLFILE AUTOBACKUP ON;
    new RMAN configuration parameters are successfully stored
    starting full resync of recovery catalog
    full resync complete

## バックアップの最適化

バックアップの最適化を有効にすると、保存方針によって同一ファイルの不要なバックアップをしなくなる。例えば、保存方針が `REDUNDANCY 1` として USERS 表領域を READ ONLY にして、`BACKUP DATABASE` を実行すると USERS 表領域のバックアップは 2 つ(REDUNDANCY + 1)あればよいので、3 回目以降の `BACKUP DATABASE` では USERS 表領域のバックアップが取得されなくなる。

    RMAN> CONFIGURE BACKUP OPTIMIZATION ON;

    new RMAN configuration parameters:
    CONFIGURE BACKUP OPTIMIZATION ON;
    new RMAN configuration parameters are successfully stored
    starting full resync of recovery catalog
    full resync complete

## ブロック・チェンジ・トラッキングを有効化

DB で変更されたブロックをブロック・チェンジ・トラッキング・ファイル上に記録するようにする。このことで、増分バックアップ時に変更があったブロックだけを読むだけでよくなり、データファイルのすべてのブロックの読み込みが不要になり、パフォーマンスが向上する。

マニュアルでは `DB_CREATE_FILE_DEST` の設定の記載があるが、ブロック・チェンジ・トラッキング・ファイルのパスをフルパスで指定する場合は `DB_CREATE_FILE_DEST` の設定は不要。

    SQL> ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
         USING FILE '/u01/app/oracle/oradata/PROD1/rman_change_track.f' REUSE;

    Database altered.

    SQL> SELECT STATUS, FILENAME
         FROM V$BLOCK_CHANGE_TRACKING;

    STATUS   FILENAME
    -------- ------------------------------------------------------------
    ENABLED  /u01/app/oracle/oradata/PROD1/rman_change_track.f
