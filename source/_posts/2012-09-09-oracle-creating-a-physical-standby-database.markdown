---
layout: post
title: "[Oracle] フィジカル・スタンバイ・データベースの作成"
date: 2012-09-09 05:29
comments: true
categories: [Oracle, DB, 11gR2, EM, Data Guard]
---
## 目的

Data Guard のフィジカル・スタンバイ・データベースを作成する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)
* EM: Oracle Enterprise Manager Grid Control 11g (11.1)

## フィジカル・スタンバイ・データベースの作成

EM Grid Control を使用してフィジカル・スタンバイ・データベースを作成する。

* プライマリ DB インスタンスの "可用性" タブ内の "スタンバイ・データベースの追加" をクリック

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-1.png 720 450 %}

* SYSDBA 権限で接続

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-2.png 720 450 %}

* "スタンバイ・データベースの追加" をクリック

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-3.png 720 450 %}

* "新規のフィジカル・スタンバイ・データベースの作成" を選択し、"続行" をクリック

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-4.png 720 450 %}

* "Recovery Manager (RMAN)を使用してデータベース・ファイルをコピーします" のままで "次へ" をクリック

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-5.png 720 450 %}

* "プライマリ・ホスト資格証明" に OS ユーザとパスワードを入力。"スタンバイREDOログ・ファイルにOracle Managed Files(OMF)を使用" のチェックを外す

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-6.png 720 450 %}

* スタンバイ DB の "インスタンス名", "ホスト" を環境に合わせて変更

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-7.png 720 450 %}

* スタンバイ DB のファイルの場所として "ファイル名と場所をプライマリ・データベースと同じ場所に保持" を選択

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-8.png 720 450 %}

* "続行"

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-9.png 720 450 %}

* "一意のデータベース名", "ターゲット名" (EM で表示されるターゲット名)を変更、スタンバイ DB の資格証明として "SYSDBA監視資格証明の使用" を選択

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-10.png 720 450 %}

* "終了"

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-11.png 720 450 %}

* 処理中 -> 完了となるが、なぜか完了の画面のまま "概要" のページに戻らない。

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-12.png 720 450 %}

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-13.png 720 450 %}

alert.log を確認して処理が終わっていそうだったら、EM の画面を切り替えてしまってよい。

* もう一度プライマリ DB インスタンスの "可用性" タブに行くと Data Guard の項目が変わっているので "設定および管理" をクリックして、正常にスタンバイ DB が作成されていることを確認

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-14.png 720 450 %}

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-15.png 720 450 %}


