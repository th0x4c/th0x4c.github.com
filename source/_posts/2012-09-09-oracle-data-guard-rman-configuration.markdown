---
layout: post
title: "[Oracle] Data Guard 環境での RMAN の構成"
date: 2012-09-09 23:00
comments: true
categories: [Oracle, DB, 11gR2, Backup and Recovery, Data Guard]
---
## 目的

Data Guard 環境での RMAN のアーカイブ・ログ削除方針を変更する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* Data Guard概要および管理  
  -> 11 Recovery Managerを使用したファイルのバックアップおよびリストア  
  -> [11.3.2 プライマリ・データベースでのRecovery Manager構成](http://docs.oracle.com/cd/E16338_01/server.112/b56302/rman.htm#BAJHHAEB)

## Data Guard 環境での RMAN の構成

Data Guard 環境での RMAN のアーカイブ・ログ削除方針を変更する。

* 現在の設定の確認

プライマリ DB をターゲットとして接続し、`show all` コマンドで現在の設定を確認

    $ export ORACLE_SID=PROD1
    $ rman target /
    RMAN> show all;

    using target database control file instead of recovery catalog
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

アーカイブ・ログの削除方針(`CONFIGURE ARCHIVELOG DELETION POLICY`)はデフォルトの `NONE` になっている。

* アーカイブ・ログの削除方針を変更する。

`CONFIGURE ARCHIVELOG DELETION POLICY` コマンドによりアーカイブ・ログの削除方針を変更する。

すべてのスタンバイ DB への**送信(SHIPPED)**を確認するまでアーカイブ・ログを削除しないようにする場合は、次のようにする。

    RMAN> CONFIGURE ARCHIVELOG DELETION POLICY TO SHIPPED TO ALL STANDBY;

    new RMAN configuration parameters:
    CONFIGURE ARCHIVELOG DELETION POLICY TO SHIPPED TO ALL STANDBY;
    new RMAN configuration parameters are successfully stored

すべてのスタンバイ DB への**適用(APPLIED)**を確認するまでアーカイブ・ログを削除しないようにする場合は、次のようにする。

    RMAN> CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON ALL STANDBY;

    new RMAN configuration parameters:
    CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON ALL STANDBY;
    new RMAN configuration parameters are successfully stored