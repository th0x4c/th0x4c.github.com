---
layout: post
title: "[Oracle] Data Guard ファスト・スタート・フェイルオーバー"
date: 2012-09-11 19:54
comments: true
categories: [Oracle, DB, 11gR2, EM, Data Guard]
---
## 目的

Data Guard のファスト・スタート・フェイルオーバーにより、プライマリ DB での障害発生時に自動でフェイルオーバーするようにする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)
* EM: Oracle Enterprise Manager Grid Control 11g (11.1)

## ファスト・スタート・フェイルオーバーの構成

Data Guard のファスト・スタート・フェイルオーバーにより、プライマリ DB での障害発生時に自動でフェイルオーバーするようにできる。
ファスト・スタート・フェイルオーバーには、プライマリ DB、スタンバイ DB を監視する"オブザーバ"が必要。オブザーバはプライマリ DB、スタンバイ DB とは別のサーバに配置することが望ましいが、今回はスタンバイ DB と同じサーバに構成する。

EM Grid Control を使用してファスト・スタート・フェイルオーバーを有効にする。

* プライマリ DB インスタンスの "可用性" タブ内の Data Guard の項目から "設定および管理" をクリックして、Data Guard のページを開く

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-14.png 720 450 %}

* "ファスト・スタート・フェイルオーバー" の "無効" をクリック

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-1.png 720 450 %}

* "オブザーバの構成" をクリック

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-2.png 720 450 %}

* オブザーバの場所として今回はスタンバイ DB と同じサーバを指定。ORACLE_HOME の位置も入力する。

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-3.png 720 450 %}

* "続行"

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-4.png 720 450 %}

* オブザーバを起動する OS ユーザを入力

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-5.png 720 450 %}

* "続行"

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-6.png 720 450 %}

* "はい"

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-7.png 720 450 %}

* 処理中 -> 完了

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-8.png 720 450 %}

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-9.png 720 450 %}

なぜか上記画面のまま "概要" のページに戻らない。alert.log を確認して処理が終わっていそうだったら、EM の画面を切り替えてしまってよい。

* Data Guard の構成画面に戻ると "ファスト・スタート・フェイルオーバー" が "stdby1.localに対して有効" になり、"オブザーバの場所" が "sv2.local" になっている。(`stdby1.local` はスタンバイ DB, `sv2.local` はオブザーバを構成したサーバで今回はスタンバイ DB と同じサーバ)

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-10.png 720 450 %}

## ファスト・スタート・フェイルオーバーの検証

ファスト・スタート・フェイルオーバーが動作するか検証してみる。
[マニュアル Data Guard Broker](http://docs.oracle.com/cd/E16338_01/server.112/b56304/sofo.htm#BCGHEJFH) によると次の場合にファスト・スタート・フェイルオーバーが試行される。

* プライマリ・データベースと、オブザーバおよびターゲット・スタンバイ・データベースの両方との接続が失われた場合
* インスタンス障害
* 強制終了(ABORT オプションでの停止)

正常にデータベースを停止した場合(NORMAL, IMMEDIATE, TRANSACTIONAL)では、ファスト・スタート・フェイルオーバーは試行されないので、プライマリ DB を停止するときは通常通り ABORT 以外のオプションで SHUTDOWN すればいい。

ファスト・スタート・フェイルオーバーを検証するためにプライマリ DB を `shutdown abort` で強制終了する。

    # プライマリ側で実施
    $ export ORACLE_SID=PROD1
    $ sqlplus '/as sysdba'
    SQL> shutdown abort
    ORACLE instance shut down.

これで、プライマリ DB の異常をオブザーバが検知して、フェイルオーバーが自動で開始される。
オブザーバは Data Guard Broker の dgmgrl コマンドで実行されているので、ログは `$ORACLE_HOME/rdbms/log/dgmgrl_XXXX_XXXX.log` に出力されている。

    # オブザーバ配置サーバで実施
    $ ps -ef | grep dgmgrl
    oracle   12805     1  0 20:27 ?        00:00:01 /u01/app/oracle/product/11.2.0/dbhome_1/bin/dgmgrl -logfile /u01/app/oracle/product/11.2.0/dbhome_1/rdbms/log/dgmgrl_PROD1_12804.log

    $ cat $ORACLE_HOME/rdbms/log/dgmgrl_PROD1_12804.log 
    Observer started
    [W000 09/11 20:27:30.89] Observer started.

    21:07:33.37  2012年9月11日 Tuesday
    Initiating Fast-Start Failover to database "stdby1"...
    Performing failover NOW, please wait...
    Failover succeeded, new primary is "stdby1"
    21:07:35.66  2012年9月11日 Tuesday

これで、フェイルオーバーが実施された。EM の元スタンバイ DB 側を確認すると以下のようになっている。

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-11.png 720 450 %}

この後、元プライマリ DB を起動すると、自動でスタンバイ DB として構成してくれる。

    # 元プライマリ側で実施
    $ export ORACLE_SID=PROD1
    $ sqlplus '/as sysdba'
    SQL> shutdown abort
    ORACLE instance shut down.
    SQL> startup
    ORACLE instance started.
    
    Total System Global Area  835104768 bytes
    Fixed Size                  2232960 bytes
    Variable Size             511708544 bytes
    Database Buffers          314572800 bytes
    Redo Buffers                6590464 bytes
    Database mounted.
    ORA-16649: possible failover to another database prevents this database from
    being opened

オブザーバのログを確認すると、元プライマリ DB が新たにスタンバイ DB としてマウントされたログ出力がある。

    # オブザーバ配置サーバで実施
    $ cat $ORACLE_HOME/rdbms/log/dgmgrl_PROD1_12804.log 
    21:18:47.02  2012年9月11日 Tuesday
    Initiating reinstatement for database "PROD1"...
    Reinstating database "PROD1", please wait...
    Operation requires shutdown of instance "PROD1" on database "PROD1"
    Shutting down instance "PROD1"...
    ORA-01109: database not open
    
    Database dismounted.
    ORACLE instance shut down.
    Operation requires startup of instance "PROD1" on database "PROD1"
    Starting instance "PROD1"...
    ORACLE instance started.
    Database mounted.
    Continuing to reinstate database "PROD1" ...
    Reinstatement of database "PROD1" succeeded
    21:19:33.19  2012年9月11日 Tuesday

EM の元スタンバイ DB 側を確認すると以下のように元プライマリ DB が新たにスタンバイ DB として構成されていることが分かる。

{% img /images/2012-09-11-oracle-data-guard-fast-start-failover/dgfsf-12.png 720 450 %}

この後、スイッチオーバすれば、元の構成に戻る。

## ファスト・スタート・フェイルオーバー環境でのデータベースの停止

[マニュアル](http://docs.oracle.com/cd/E16338_01/server.112/b56304/sofo.htm#CHDJCDDB) に記載があるように、ファスト・スタート・フェイルオーバー環境でのデータベースの停止する場合は次のようにする。

1. オブザーバを停止し、プライマリ DB およびターゲット・スタンバイ DB の両方について、V$DATABASE.FS_FAILOVER_OBSERVER_PRESENT 列が `NO` になるまで待機する。こうすると、プライマリ DB の停止中に、ファスト・スタート・フェイルオーバーは実行されない。

2. プライマリ DB およびターゲット・スタンバイ DB を停止(`SHUTDOWN`)する。

以下が実行例。

`dgmgrl` にてオブザーバを停止

    $ export ORACLE_SID=stdby1
    $ dgmgrl
    DGMGRL for Linux: Version 11.2.0.3.0 - 64bit Production

    Copyright (c) 2000, 2009, Oracle. All rights reserved.

    Welcome to DGMGRL, type "help" for information.
    DGMGRL> connect sys/oracle
    Connected.
    DGMGRL> stop observer;
    Done.
    DGMGRL> exit

プライマリ DB およびスタンバイ DB にてV$DATABASE.FS_FAILOVER_OBSERVER_PRESENT 列が `NO` になることを確認して `SHUTDOWN` する。

    # プライマリ DB および スタンバイ DB でそれぞれ実施
    $ sqlplus 'as sysdba'
    SQL> SELECT FS_FAILOVER_OBSERVER_PRESENT FROM V$DATABASE;
 
    FS_FAILOVER_OBSERVER_
    ---------------------
    YES

    SQL> /

    FS_FAILOVER_OBSERVER_
    ---------------------
    NO

    SQL> shutdown immediate
    Database closed.
    Database dismounted.
    ORACLE instance shut down.
    SQL> exit

起動時はプライマリ DB およびスタンバイ DB を起動した後に、`dgmgrl` でオブザーバを起動(`start observer`)する。