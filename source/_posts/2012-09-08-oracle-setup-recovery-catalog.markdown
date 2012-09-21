---
layout: post
title: "[Oracle] リカバリ・カタログのセットアップ"
date: 2012-09-08 23:44
comments: true
categories: [Oracle, DB, 11gR2, Backup and Recovery]
---
## 目的

Recovery Manager(RMAN) のリカバリ・カタログをセットアップする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* バックアップおよびリカバリ・ユーザーズ・ガイド  
  -> 13 リカバリ・カタログの管理  
  -> [リカバリ・カタログの作成](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmcatdb.htm#i1013599)  
     [リカバリ・カタログへのデータベースの登録](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmcatdb.htm#CIHECCEF)

## リカバリ・カタログとは

リカバリ・カタログには、RMAN で使用するメタデータ情報(バックアップ・ファイルの情報など)が格納されている。RMAN が使用するメタデータは通常は制御ファイルに格納されているが、リカバリ・カタログとして(リカバリのターゲットとなる DB とは別の) DB に格納することができる。リカバリ・カタログを使用するメリットは次のようなものがある。

* 制御ファイルに格納されているメタデータと同様の情報が格納されているので、冗長性が確保される。制御ファイルが消失した場合も、リカバリ・カタログを使用すればいい。制御ファイルとの同期はバックアップ実行時などに自動で実行される。

* 複数の DB の RMAN メタデータを集中管理できる。

* リカバリ・カタログには、制御ファイルより長期のメタデータ履歴を格納できる。よって、制御ファイルの履歴より前の時点にリカバリすることができる。

* 一部の RMAN 機能は、リカバリ・カタログが必須。例えば、リカバリ・カタログには Recovery Manager スクリプトを格納することができる。

* Data Guard 環境で RMAN を使用する場合は、リカバリ・カタログが必要。

## リカバリ・カタログの作成

リカバリ・カタログを作成する DB にリカバリ・カタログを所有するスキーマを作成して、必要な権限を与え、リカバリ・カタログを作成する。

* リカバリ・カタログを所有するスキーマを作成

リカバリ・カタログを格納する DB に接続してスキーマを作成。

    $ sqlplus sys/oracle@EMREP as sysdba
    SQL> CREATE USER rman IDENTIFIED BY rman
         TEMPORARY TABLESPACE temp
         DEFAULT TABLESPACE users
         QUOTA UNLIMITED ON users;
    
    User created.

* 必要な権限の付与

リカバリ・カタログの管理に必要なすべての権限を含む `RECOVERY_CATALOG_OWNER` ロールを付与する。

    SQL> GRANT RECOVERY_CATALOG_OWNER TO rman;
    
    Grant succeeded.

* リカバリ・カタログの作成

RMAN を使用して先ほど作成したリカバリ・カタログを所有するスキーマで接続して、リカバリ・カタログを作成。

    $ rman catalog rman/rman@EMREP
    RMAN> CREATE CATALOG;
    
    recovery catalog created

## リカバリ・カタログへの DB の登録

リカバリ・カタログにターゲットとなる DB を登録する。

* ターゲット DB がマウントしていない場合は、マウントまたはオープンする。

* RMAN からターゲット DB, リカバリ・カタログに接続する。

RMAN から以下のようにして接続

    $ export ORACLE_SID=PROD1
    $ rman target / catalog rman/rman@EMREP

    Recovery Manager: Release 11.2.0.3.0 - Production on Sun Sep 9 00:37:33 2012

    Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

    connected to target database: PROD1 (DBID=2014160803)
    connected to recovery catalog database

* リカバリ・カタログにターゲット DB を登録

`REGISTER DATABASE` によりターゲット DB を登録する。

    RMAN> REGISTER DATABASE;

    database registered in recovery catalog
    starting full resync of recovery catalog
    full resync complete

* 正常に登録されていることを確認

`REPORT SCHEMA` により正常に登録されていることを確認する。

    RMAN> REPORT SCHEMA;
    
    Report of database schema for database with db_unique_name PROD1
    
    List of Permanent Datafiles
    ===========================
    File Size(MB) Tablespace           RB segs Datafile Name
    ---- -------- -------------------- ------- ------------------------
    1    720      SYSTEM               YES     /u01/app/oracle/oradata/PROD1/system01.dbf
    2    550      SYSAUX               NO      /u01/app/oracle/oradata/PROD1/sysaux01.dbf
    3    95       UNDOTBS1             YES     /u01/app/oracle/oradata/PROD1/undotbs01.dbf
    4    5        USERS                NO      /u01/app/oracle/oradata/PROD1/users01.dbf
    5    345      EXAMPLE              NO      /u01/app/oracle/oradata/PROD1/example01.dbf
    
    List of Temporary Files
    =======================
    File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
    ---- -------- -------------------- ----------- --------------------
    1    29       TEMP                 32767       /u01/app/oracle/oradata/PROD1/temp01.dbf

