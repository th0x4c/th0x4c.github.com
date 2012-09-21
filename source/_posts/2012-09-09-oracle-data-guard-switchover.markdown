---
layout: post
title: "[Oracle] Data Guard スイッチオーバー"
date: 2012-09-09 21:19
comments: true
categories: [Oracle, DB, 11gR2, EM, Data Guard]
---
## 目的

Data Guard のスイッチオーバーを実行して、プライマリとスタンバイを切り替える。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)
* EM: Oracle Enterprise Manager Grid Control 11g (11.1)

## スイッチオーバーの実施

EM Grid Control を使用してスイッチオーバーを行う。

* プライマリ DB インスタンスの "可用性" タブ内の Data Guard の項目から "設定および管理" をクリックして、Data Guard のページを開く

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-14.png 720 450 %}

* "スイッチオーバー" をクリック

{% img /images/2012-09-09-oracle-data-guard-switchover/dgso-1.png 720 450 %}

* スタンバイ・サイト に接続する OS ユーザを入力

{% img /images/2012-09-09-oracle-data-guard-switchover/dgso-2.png 720 450 %}

* プライマリ・サイト に接続する OS ユーザを入力

{% img /images/2012-09-09-oracle-data-guard-switchover/dgso-3.png 720 450 %}

* "はい"

{% img /images/2012-09-09-oracle-data-guard-switchover/dgso-4.png 720 450 %}

* 処理中 -> 完了

{% img /images/2012-09-09-oracle-data-guard-switchover/dgso-5.png 720 450 %}

{% img /images/2012-09-09-oracle-data-guard-switchover/dgso-6.png 720 450 %}

なぜか上記画面のまま "概要" のページに戻らない。alert.log を確認して処理が終わっていそうだったら、EM の画面を切り替えてしまってよい。

* 今度は元スタンバイでプライマリになったインスタンスの "可用性" タブ内の Data Guard の項目から "設定および管理" をクリックして、スイッチオーバーされている(プライマリ/スタンバイが切り替わっている)ことを確認

{% img /images/2012-09-09-oracle-data-guard-switchover/dgso-7.png 720 450 %}