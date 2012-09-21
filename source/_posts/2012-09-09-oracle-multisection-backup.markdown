---
layout: post
title: "[Oracle] マルチセクション・バックアップ"
date: 2012-09-09 03:25
comments: true
categories: [Oracle, DB, 11gR2, Backup and Recovery]
---
## 目的

マルチセクション・バックアップにより効率的にバックアップを行う。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* バックアップおよびリカバリ・ユーザーズ・ガイド  
  -> 10 データベースのバックアップ: 高度なトピック  
  -> [セクションへの大規模なデータファイルのバックアップの分割](http://docs.oracle.com/cd/E16338_01/backup.112/b56269/rcmbckad.htm#CHDBAHAD)

## マルチセクション・バックアップの実行

マルチセクション・バックアップにより、ファイルサイズが大きいデータ・ファイルを複数のバックアップ・ピースに分割してバックアップすることができる。パラレル化の設定がされていれば、複数のチャネルを使用して平行でバックアップされるためパフォーマンスが向上する。

マルチセクション・バックアップを行うためには、`SECTION SIZE` パラメータを指定して `BACKUP` を実行すればよい。

* 必要に応じて、パラレル化の設定をする。(パラレル化の設定がされていないとマルチセクション・バックアップを実施しても複数バックアップ・ピースに分割されるがシリアルにバックアップされてあまり意味がない。)

`CONFIGURE` コマンドで設定

    RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 2;

    new RMAN configuration parameters:
    CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;
    new RMAN configuration parameters are successfully stored
    starting full resync of recovery catalog
    full resync complete

* マルチセクション・バックアップ実施

以下の例では 約 350M の 1 つデータファイルからなる example 表領域のバックアップを行っている。パラレル度を 2 にしたので、チャネルが 2 つ(`ORA_DISK_1` と `ORA_DISK_2`)割り当てられてパラレルで動作する。また、セクションサイズを 100M で指定したので、データファイルを 100M に分けて 4 つのバックアップ・ピースでバックアップしている。(1 つのピースは 12800 blocks * 8k = 100M)

    RMAN> BACKUP
          SECTION SIZE 100M
          TABLESPACE example;

    Starting backup at 12-09-09
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=145 device type=DISK
    allocated channel: ORA_DISK_2
    channel ORA_DISK_2: SID=140 device type=DISK
    channel ORA_DISK_1: starting full datafile backup set
    channel ORA_DISK_1: specifying datafile(s) in backup set
    input datafile file number=00005 name=/u01/app/oracle/oradata/PROD1/example01.dbf
    backing up blocks 1 through 12800
    channel ORA_DISK_1: starting piece 1 at 12-09-09
    channel ORA_DISK_2: starting full datafile backup set
    channel ORA_DISK_2: specifying datafile(s) in backup set
    input datafile file number=00005 name=/u01/app/oracle/oradata/PROD1/example01.dbf
    backing up blocks 12801 through 25600
    channel ORA_DISK_2: starting piece 2 at 12-09-09
    channel ORA_DISK_1: finished piece 1 at 12-09-09
    piece handle=/u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4tsts_.bkp tag=TAG20120909T035001 comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
    channel ORA_DISK_1: starting full datafile backup set
    channel ORA_DISK_1: specifying datafile(s) in backup set
    input datafile file number=00005 name=/u01/app/oracle/oradata/PROD1/example01.dbf
    backing up blocks 25601 through 38400
    channel ORA_DISK_1: starting piece 3 at 12-09-09
    channel ORA_DISK_2: finished piece 2 at 12-09-09
    piece handle=/u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4tsws_.bkp tag=TAG20120909T035001 comment=NONE
    channel ORA_DISK_2: backup set complete, elapsed time: 00:00:01
    channel ORA_DISK_2: starting full datafile backup set
    channel ORA_DISK_2: specifying datafile(s) in backup set
    input datafile file number=00005 name=/u01/app/oracle/oradata/PROD1/example01.dbf
    backing up blocks 38401 through 44240
    channel ORA_DISK_2: starting piece 4 at 12-09-09
    channel ORA_DISK_1: finished piece 3 at 12-09-09
    piece handle=/u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4ttyy_.bkp tag=TAG20120909T035001 comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:02
    channel ORA_DISK_2: finished piece 4 at 12-09-09
    piece handle=/u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4tv1o_.bkp tag=TAG20120909T035001 comment=NONE
    channel ORA_DISK_2: backup set complete, elapsed time: 00:00:01
    Finished backup at 12-09-09

バックアップの確認。確かに 4 つのバックアップ・ピースに分かれている。

    RMAN> list backup;
    ...(*snip*)
    
    BS Key  Type LV Size       Device Type Elapsed Time Completion Time
    ------- ---- -- ---------- ----------- ------------ ---------------
    956     Full    69.01M     DISK        00:00:02     12-09-09
      List of Datafiles in backup set 956
      File LV Type Ckp SCN    Ckp Time Name
      ---- -- ---- ---------- -------- ----
      5       Full 1160848    12-09-09 /u01/app/oracle/oradata/PROD1/example01.dbf

      Backup Set Copy #1 of backup set 956
      Device Type Elapsed Time Completion Time Compressed Tag
      ----------- ------------ --------------- ---------- ---
      DISK        00:00:02     12-09-09        NO         TAG20120909T035001

        List of Backup Pieces for backup set 956 Copy #1
        BP Key  Pc# Status      Piece Name
        ------- --- ----------- ----------
        958     1   AVAILABLE   /u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4tsts_.bkp
        959     2   AVAILABLE   /u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4tsws_.bkp
        960     3   AVAILABLE   /u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4ttyy_.bkp
        961     4   AVAILABLE   /u01/app/oracle/fast_recovery_area/PROD1/backupset/2012_09_09/o1_mf_nnndf_TAG20120909T035001_84q4tv1o_.bkp

