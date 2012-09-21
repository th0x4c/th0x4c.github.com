---
layout: post
title: "[Oracle] Data Guard スナップショット・スタンバイ・データベース"
date: 2012-09-09 21:51
comments: true
categories: [Oracle, DB, 11gR2, EM, Data Guard]
---
## 目的

Data Guard の "スナップショット・スタンバイ・データベース" 機能により、スタンバイ DB を一時的に更新可能にする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)
* EM: Oracle Enterprise Manager Grid Control 11g (11.1)

## スナップショット・スタンバイ・データベースへの変換

Data Guard の "スナップショット・スタンバイ・データベース" 機能により、スタンバイ DB が一時的に更新可能にできる。"スナップショット・スタンバイ・データベース" となっている間はプライマリからの REDO は適用されず、元に戻すとスタンバイ DB での更新は破棄されて、プライマリからの REDO 適用が再開される。例えば、一時的にスタンバイ DB をテスト環境として使用する場合に使える。

EM Grid Control を使用すると変換が楽にできる。("変換" をクリックするだけ)

* プライマリ DB インスタンスの "可用性" タブ内の Data Guard の項目から "設定および管理" をクリックして、Data Guard のページを開く

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-14.png 720 450 %}

* "変換" をクリック

{% img /images/2012-09-09-oracle-data-guard-snapshot-standby-database/dgss-1.png 720 450 %}

* "はい"

{% img /images/2012-09-09-oracle-data-guard-snapshot-standby-database/dgss-2.png 720 450 %}

* 処理中 -> 完了

{% img /images/2012-09-09-oracle-data-guard-snapshot-standby-database/dgss-3.png 720 450 %}

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-13.png 720 450 %}

なぜか上記画面のまま "概要" のページに戻らない。alert.log を確認して処理が終わっていそうだったら、EM の画面を切り替えてしまってよい。

* ロールが "スナップショット・スタンバイ" になる。

{% img /images/2012-09-09-oracle-data-guard-snapshot-standby-database/dgss-4.png 720 450 %}

スナップショット・スタンバイ DB に対しては DB に変更も可能。

    $ sqlplus scott/tiger@STDBY1_DGMGRL
    SQL> CREATE TABLE emp2 AS SELECT * FROM emp;

    Table created.

フィジカル・スタンバイ DB に戻す場合はもう一度 EM から "変換" をクリックすればよい。
