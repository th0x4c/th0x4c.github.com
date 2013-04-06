---
layout: post
title: "[Oracle] トランスポータブル表領域"
date: 2012-09-12 23:18
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

トランスポータブル表領域により DB 間で表領域の移動を行う。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* 管理者ガイド  
  -> 14 表領域の管理  
  -> [データベース間で表領域をトランスポートする手順および例](http://docs.oracle.com/cd/E16338_01/server.112/b56301/tspaces013.htm#i1007252)

## トランスポータブル表領域

マニュアルに記載の手順に従い、トランスポータブル表領域により DB 間で表領域の移動を行う。

* 今回の検証用に表領域を作成

`tt_emp` 表領域を作成する。

    SQL> CREATE TABLESPACE tt_emp
         DATAFILE '/u01/app/oracle/oradata/PROD1/tt_emp01.dbf'
         SIZE 10M REUSE AUTOEXTEND ON;
    
    Tablespace created.
    
    SQL> CREATE TABLE scott.tt_emp TABLESPACE tt_emp AS SELECT * FROM scott.emp;
    
    Table created.

* endianness の確認

異なるプラットフォーム間で表領域のトランスポートを行う場合は endianness が異なると変換が必要になる。
endianness を確認する。

    SQL> set pages 100
         col platform_name for a40
         col endian_format for a15
    SQL> SELECT * FROM V$TRANSPORTABLE_PLATFORM;

    PLATFORM_ID PLATFORM_NAME                            ENDIAN_FORMAT
    ----------- ---------------------------------------- ---------------
              1 Solaris[tm] OE (32-bit)                  Big
              2 Solaris[tm] OE (64-bit)                  Big
              7 Microsoft Windows IA (32-bit)            Little
             10 Linux IA (32-bit)                        Little
              6 AIX-Based Systems (64-bit)               Big
              3 HP-UX (64-bit)                           Big
              5 HP Tru64 UNIX                            Little
              4 HP-UX IA (64-bit)                        Big
             11 Linux IA (64-bit)                        Little
             15 HP Open VMS                              Little
              8 Microsoft Windows IA (64-bit)            Little
              9 IBM zSeries Based Linux                  Big
             13 Linux x86 64-bit                         Little
             16 Apple Mac OS                             Big
             12 Microsoft Windows x86 64-bit             Little
             17 Solaris Operating System (x86)           Little
             18 IBM Power Based Linux                    Big
             19 HP IA Open VMS                           Little
             20 Solaris Operating System (x86-64)        Little
             21 Apple Mac OS (x86-64)                    Little

    20 rows selected.

    SQL> SELECT PLATFORM_NAME FROM V$DATABASE;

    PLATFORM_NAME
    ----------------------------------------
    Linux x86 64-bit

    SQL> SELECT d.PLATFORM_NAME, ENDIAN_FORMAT
         FROM V$TRANSPORTABLE_PLATFORM tp, V$DATABASE d
         WHERE tp.PLATFORM_NAME = d.PLATFORM_NAME;

    PLATFORM_NAME                            ENDIAN_FORMAT
    ---------------------------------------- ---------------
    Linux x86 64-bit                         Little

* 表領域が自己完結型かどうかの確認

表領域内に別の表領域の表の索引が含まれている場合などは自己完結型でないためトランスポートできない。
表領域が自己完結型かどうかチェックする。

    SQL> EXECUTE DBMS_TTS.TRANSPORT_SET_CHECK('tt_emp', TRUE);

    PL/SQL procedure successfully completed.

    SQL> SELECT * FROM TRANSPORT_SET_VIOLATIONS;

    no rows selected

もし、違反している場合は `TRANSPORT_SET_VIOLATIONS` ビューに違反している内容が表示される。

* 表領域を読み取り専用にする

表領域を読取り専用にする。

    SQL> ALTER TABLESPACE tt_emp READ ONLY;

    Tablespace altered.

* データ・ポンプによりエクスポート

`expdp` によりエクスポートする。

    $ expdp system/oracle dumpfile=expdat.dmp directory=data_pump_dir \
            transport_tablespaces=tt_emp logfile=tts_export.log

    Export: Release 11.2.0.3.0 - Production on Wed Sep 12 23:53:49 2012

    Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

    Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
    With the Partitioning, OLAP, Data Mining and Real Application Testing options
    FLASHBACK automatically enabled to preserve database integrity.
    Starting "SYSTEM"."SYS_EXPORT_TRANSPORTABLE_01":  system/******** dumpfile=expdat.dmp directory=data_pump_dir transport_tablespaces=tt_emp logfile=tts_export.log 
    Processing object type TRANSPORTABLE_EXPORT/PLUGTS_BLK
    Processing object type TRANSPORTABLE_EXPORT/TABLE
    Processing object type TRANSPORTABLE_EXPORT/POST_INSTANCE/PLUGTS_BLK
    Master table "SYSTEM"."SYS_EXPORT_TRANSPORTABLE_01" successfully loaded/unloaded
    ******************************************************************************
    Dump file set for SYSTEM.SYS_EXPORT_TRANSPORTABLE_01 is:
      /u01/app/oracle/admin/PROD1/dpdump/expdat.dmp
    ******************************************************************************
    Datafiles required for transportable tablespace TT_EMP:
      /u01/app/oracle/oradata/PROD1/tt_emp01.dbf
    Job "SYSTEM"."SYS_EXPORT_TRANSPORTABLE_01" successfully completed at 23:54:17

* endianness の変換

もし、endianness の異なるプラットフォームにトランスポートする場合は、RMAN でデータファイルの変換を行う。

    $ export ORACLE_SID=PROD1
    $ rman target /
    RMAN> CONVERT TABLESPACE tt_emp
          TO PLATFORM 'Apple Mac OS'
          FORMAT '/tmp/%U';

    Starting conversion at source at 12-09-12
    using target database control file instead of recovery catalog
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=43 device type=DISK
    channel ORA_DISK_1: starting datafile conversion
    input datafile file number=00006 name=/u01/app/oracle/oradata/PROD1/tt_emp01.dbf
    converted datafile=/tmp/data_D-PROD1_I-2014160803_TS-TT_EMP_FNO-6_07nl25cg
    channel ORA_DISK_1: datafile conversion complete, elapsed time: 00:00:01
    Finished conversion at source at 12-09-12  


* 表領域のデータファイルと、エクスポート・ダンプ・ファイルをトランスポート先に移動

今回は同一サーバ上の別 DB にした。

    # エクスポート・ダンプ・ファイル をトランスポート先 DB の DATA_PUMP_DIR にコピー
    $ cp -i /u01/app/oracle/admin/PROD1/dpdump/expdat.dmp /u01/app/oracle/admin/PROD2/dpdump/

    # 通常はデータファイルをトランスポート先のデータファイルの位置にコピー
    # 今回はテストとして endianness を変換したファイルをコピーした
    $ cp -i /tmp/data_D-PROD1_I-2014160803_TS-TT_EMP_FNO-6_07nl25cg /tmp/tt_emp01.dbf

* endianness の変換

もし、endianness の異なるプラットフォームにトランスポートする場合でトランスポート元で変換を行っていない場合は、RMAN でデータファイルの変換を行う。


    $ export ORACLE_SID=PROD2
    $ rman target /
    RMAN> CONVERT DATAFILE
          '/tmp/tt_emp01.dbf'
          TO PLATFORM 'Linux x86 64-bit'
          FROM PLATFORM 'Apple Mac OS'
          DB_FILE_NAME_CONVERT='/tmp/', '/u01/app/oracle/oradata/PROD2/';

    Starting conversion at target at 12-09-13
    using target database control file instead of recovery catalog
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=36 device type=DISK
    channel ORA_DISK_1: starting datafile conversion
    input file name=/tmp/tt_emp01.dbf
    converted datafile=/u01/app/oracle/oradata/PROD2/tt_emp01.dbf
    channel ORA_DISK_1: datafile conversion complete, elapsed time: 00:00:01
    Finished conversion at target at 12-09-13

* トランスポート元の表領域を読取り/書込みモードに戻す。

`ALTER TABLESPACE` にて戻す。

    $ export ORACLE_SID=PROD1
    $ sqlplus '/as sysdba'
    SQL> ALTER TABLESPACE tt_emp READ WRITE;
    
    Tablespace altered.

* データ・ポンプによりインポート

`impdp` によりインポートする。

    $ export ORACLE_SID=PROD2
    $ impdp system/oracle dumpfile=expdat.dmp directory=data_pump_dir \
            transport_datafiles='/u01/app/oracle/oradata/PROD2/tt_emp01.dbf' \
            logfile=tts_import.log

    Import: Release 11.2.0.3.0 - Production on Thu Sep 13 00:22:31 2012

    Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

    Connected to: Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
    With the Partitioning, OLAP, Data Mining and Real Application Testing options
    Master table "SYSTEM"."SYS_IMPORT_TRANSPORTABLE_01" successfully loaded/unloaded
    Starting "SYSTEM"."SYS_IMPORT_TRANSPORTABLE_01":  system/******** dumpfile=expdat.dmp directory=data_pump_dir transport_datafiles=/u01/app/oracle/oradata/PROD2/tt_emp01.dbf logfile=tts_import.log 
    Processing object type TRANSPORTABLE_EXPORT/PLUGTS_BLK
    Processing object type TRANSPORTABLE_EXPORT/TABLE
    Processing object type TRANSPORTABLE_EXPORT/POST_INSTANCE/PLUGTS_BLK
    Job "SYSTEM"."SYS_IMPORT_TRANSPORTABLE_01" successfully completed at 00:22:34

* トランスポート先の表領域を読取り/書込みモードに戻す。

`ALTER TABLESPACE` にて戻す。

    $ export ORACLE_SID=PROD2
    $ sqlplus '/as sysdba'
    SQL> ALTER TABLESPACE tt_emp READ WRITE;
    
    Tablespace altered.
