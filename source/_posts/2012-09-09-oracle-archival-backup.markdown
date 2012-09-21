---
layout: post
title: "[Oracle] アーカイブ・バックアップの作成"
date: 2012-09-09 04:16
comments: true
categories: [Oracle, DB, 11gR2, Backup and Recovery]
---
## 目的

RMAN でバックアップの保存方針(RETENTION POLICY)から除外するアーカイブ・バックアップを作成する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* バックアップおよびリカバリ・ユーザーズ・ガイド  
  -> 9 データベースのバックアップ  
  -> [長期格納用のデータベース・バックアップの作成](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmbckba.htm#CHDHAFHJ)

## アーカイブ・バックアップの作成

RMAN でバックアップの保存方針(RETENTION POLICY)から除外するアーカイブ・バックアップを作成する。これは `DELETE OBSOLETE` しても削除されないバックアップになる。

`BACKUP ... KEEP` によりアーカイブ・バックアップが作成される。

* **注意**
  * **`KEEP FOREVER` の場合にはリカバリ・カタログが必要。**
  * **アーカイブ・バックアップは初期化パラメータ `DB_RECOVERY_FILE_DEST` で指定したディレクトリ(高速リカバリ領域)以外に保存する必要がある。**
もし、保存先を高速リカバリ領域のままにしていると、`ORA-19811: cannot have files in DB_RECOVERY_FILE_DEST with keep attributes` が発生して失敗する。
保存先は `FORMAT` で指定できる。

以下の例では、タグと通常のリストア・ポイントを指定して、アーカイブ・バックアップを作成している。

    RMAN> BACKUP DATABASE
          FORMAT '/home/oracle/backup/%U'
          TAG quarterly
          KEEP FOREVER
          RESTORE POINT FY12Q3;

    Starting backup at 12-09-09
    current log archived

    using channel ORA_DISK_1
    using channel ORA_DISK_2
    backup will never be obsolete
    archived logs required to recover from this backup will be backed up
    channel ORA_DISK_1: starting full datafile backup set
    channel ORA_DISK_1: specifying datafile(s) in backup set
    input datafile file number=00001 name=/u01/app/oracle/oradata/PROD1/system01.dbf
    input datafile file number=00005 name=/u01/app/oracle/oradata/PROD1/example01.dbf
    channel ORA_DISK_1: starting piece 1 at 12-09-09
    channel ORA_DISK_2: starting full datafile backup set
    channel ORA_DISK_2: specifying datafile(s) in backup set
    input datafile file number=00002 name=/u01/app/oracle/oradata/PROD1/sysaux01.dbf
    input datafile file number=00003 name=/u01/app/oracle/oradata/PROD1/undotbs01.dbf
    input datafile file number=00004 name=/u01/app/oracle/oradata/PROD1/users01.dbf
    channel ORA_DISK_2: starting piece 1 at 12-09-09
    channel ORA_DISK_1: finished piece 1 at 12-09-09
    piece handle=/home/oracle/backup/2mnko5ao_1_1 tag=QUARTERLY comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:07
    channel ORA_DISK_2: finished piece 1 at 12-09-09
    piece handle=/home/oracle/backup/2nnko5ao_1_1 tag=QUARTERLY comment=NONE
    channel ORA_DISK_2: backup set complete, elapsed time: 00:00:07

    using channel ORA_DISK_1
    using channel ORA_DISK_2
    backup will never be obsolete
    archived logs required to recover from this backup will be backed up
    channel ORA_DISK_1: starting full datafile backup set
    channel ORA_DISK_1: specifying datafile(s) in backup set
    including current SPFILE in backup set
    channel ORA_DISK_1: starting piece 1 at 12-09-09
    channel ORA_DISK_1: finished piece 1 at 12-09-09
    piece handle=/home/oracle/backup/2onko5av_1_1 tag=QUARTERLY comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01


    current log archived
    using channel ORA_DISK_1
    using channel ORA_DISK_2
    backup will never be obsolete
    archived logs required to recover from this backup will be backed up
    channel ORA_DISK_1: starting archived log backup set
    channel ORA_DISK_1: specifying archived log(s) in backup set
    input archived log thread=1 sequence=10 RECID=3 STAMP=793515360
    channel ORA_DISK_1: starting piece 1 at 12-09-09
    channel ORA_DISK_1: finished piece 1 at 12-09-09
    piece handle=/home/oracle/backup/2pnko5b1_1_1 tag=QUARTERLY comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01

    using channel ORA_DISK_1
    using channel ORA_DISK_2
    backup will never be obsolete
    archived logs required to recover from this backup will be backed up
    channel ORA_DISK_1: starting full datafile backup set
    channel ORA_DISK_1: specifying datafile(s) in backup set
    including current control file in backup set
    channel ORA_DISK_1: starting piece 1 at 12-09-09
    channel ORA_DISK_1: finished piece 1 at 12-09-09
    piece handle=/home/oracle/backup/2qnko5b3_1_1 tag=QUARTERLY comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
    Finished backup at 12-09-09



    
