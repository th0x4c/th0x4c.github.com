---
layout: post
title: "[Oracle] ダイレクト NFS クライアント"
date: 2012-09-07 19:24
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

OS の NFS クライアントのかわりに、Oracle の"ダイレクト NFS クライアント"を使用して NFS 上のファイルにアクセスするように Oracle の構成を変更する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* インストレーション・ガイド for Linux  
  -> 5 Oracle Databaseのインストール後の作業  
  -> [5.3.9.1 ダイレクトNFSクライアント](http://docs.oracle.com/cd/E16338_01/install.112/b56273/post_inst_task.htm#CHDIDHCH)

## ダイレクト NFS クライアントの有効化

* (マニュアルにある `oranfstab` ファイルの作成は NFS サーバへのアクセスするネットワーク・パスが複数あってロード・バランスやフェイルオーバする場合は必要だが、ネットワーク・パスが1つしか無い場合は不要だそうだ。)

* libodm11.so の置き換え

`$ORACLE_HOME/lib/libnfsodm11.so` を元々存在する `$ORACLE_HOME/lib/libodm11.so` と置き換えるだけでよい。11gR2 ではこれを実行するために `$ORACLE_HOME/rdbms/lib/ins_rdbms.mk` に `dnfs_on` というターゲットがある。11gR1 では、手動でコピーが必要だった。

    $ cd $ORACLE_HOME/rdbms/lib
    $ make -f ins_rdbms.mk dnfs_on

内容は `libodm11.so` をバックアップしておいて `libnfsodm11.so` と置き換えているだけ。

    $ make -n -f ins_rdbms.mk dnfs_on
    if [ ! -f /u01/app/oracle/product/11.2.0/dbhome_1/rdbms/lib/libodm11.so.dummy ]; then \
            cp /u01/app/oracle/product/11.2.0/dbhome_1/lib/libodm11.so /u01/app/oracle/product/11.2.0/dbhome_1/rdbms/lib/libodm11.so.dummy; \
            fi
    rm -f /u01/app/oracle/product/11.2.0/dbhome_1/lib/libodm11.so; cp /u01/app/oracle/product/11.2.0/dbhome_1/lib/libnfsodm11.so /u01/app/oracle/product/11.2.0/dbhome_1/lib/libodm11.so

* DB 再起動

設定変更を有効にするために DB インスタンスを再起動する。

## ダイレクト NFS クライアントの無効化

* libodm11.so の置き換え

バックアップしておいた `libodm11.so` を戻すだけでよい。11gR2 ではこれを実行するために `$ORACLE_HOME/rdbms/lib/ins_rdbms.mk` に `dnfs_off` というターゲットがある。11gR1 では、手動でコピーが必要だった。

    $ cd $ORACLE_HOME/rdbms/lib
    $ make -f ins_rdbms.mk dnfs_off

* DB 再起動

設定変更を有効にするために DB インスタンスを再起動する。

## 確認方法

ダイレクト NFS クライアントが有効な場合、DB 起動時に `alert.log` に以下の出力がある。

    Oracle instance running with ODM: Oracle Direct NFS ODM Library Version 3.0
