---
layout: post
title: "[Oracle] EM エージェントのインストール"
date: 2012-09-07 23:28
comments: true
categories: [Oracle, DB, EM, 11gR2, 11gR1]
---
## 目的

EM エージェントをインストールする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)
* EM: Oracle Enterprise Manager Grid Control 11g (11.1)

## EM エージェントのインストール

EM からインストールする。
事前に EM エージェントをインストールするサーバに ssh 接続できることを確認しておくこと。

* "デプロイ" タブから "エージェントのインストール" をクリック

{% img /images/2012-09-07-oracle-em-agent-installation/em-1.png 720 450 %}

* "フレッシュ・インストール" をクリック

{% img /images/2012-09-07-oracle-em-agent-installation/em-2.png 720 450 %}

* 以下のように設定
  * "ホスト・リストの指定" : DB サーバのホスト名
  * "OS 資格証明" : DB サーバの OS ユーザ
  * "root.sh の実行" のチェックははずす(sudo が使えるようになっていないといけないため。)
  * "インストールのベース・ディレクトリ" : /u01/app/oracle/Middleware
  * "管理サーバの登録パスワード" : oracle11g (EM Grid Control インストール時に指定した SYSMAN パスワードと同じにした)

{% img /images/2012-09-07-oracle-em-agent-installation/em-3.png 720 450 %}
{% img /images/2012-09-07-oracle-em-agent-installation/em-4.png 720 450 %}
{% img /images/2012-09-07-oracle-em-agent-installation/em-5.png 720 450 %}

* "セキュリティ・アップデートをMy Oracle Support経由で受け取ります。" のチェックははずす。

{% img /images/2012-09-07-oracle-em-agent-installation/em-6.png 720 450 %}

* "はい"

{% img /images/2012-09-07-oracle-em-agent-installation/em-7.png 720 450 %}

* インストール中

{% img /images/2012-09-07-oracle-em-agent-installation/em-8.png 720 450 %}

* インストール完了。"完了" をクリック。

{% img /images/2012-09-07-oracle-em-agent-installation/em-9.png 720 450 %}

* DB サーバ上で root.sh を実行する。

DB サーバ上で実行。

    $ cd /u01/app/oracle/Middleware/agent11g
    $ su
    # ./root.sh

## DB インスタンスを EM に登録する

EM エージェントをインストールしただけでは DB インスタンスのステータスが保留状態なので構成する必要がある。

{% img /images/2012-09-07-oracle-em-agent-installation/em-10.png 720 450 %}

* DB インスタンスを選択して "構成" をクリック

{% img /images/2012-09-07-oracle-em-agent-installation/em-11.png 720 450 %}

* dbsnmp のパスワードを変更

dbsnmp のパスワード(デフォルト:dbsnmp)を入力して "接続テスト" をクリックすると、失敗するので dbsnmp のパスワード変更を行う。

{% img /images/2012-09-07-oracle-em-agent-installation/em-12.png 720 450 %}

{% img /images/2012-09-07-oracle-em-agent-installation/em-13.png 720 450 %}

{% img /images/2012-09-07-oracle-em-agent-installation/em-14.png 720 450 %}

dbsnmp のパスワードを変更したらもう一度 "接続テスト" を行い成功することを確認して "次へ" をクリック

{% img /images/2012-09-07-oracle-em-agent-installation/em-15.png 720 450 %}

* 確認して "発行"

{% img /images/2012-09-07-oracle-em-agent-installation/em-16.png 720 450 %}

* 構成中

{% img /images/2012-09-07-oracle-em-agent-installation/em-17.png 720 450 %}

* "OK"

{% img /images/2012-09-07-oracle-em-agent-installation/em-18.png 720 450 %}

* 他の DB がある場合は同様に構成して、ステータスが正常になることを確認

{% img /images/2012-09-07-oracle-em-agent-installation/em-19.png 720 450 %}