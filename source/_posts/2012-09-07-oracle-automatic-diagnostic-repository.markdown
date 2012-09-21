---
layout: post
title: "[Oracle] 自動診断リポジトリの設定"
date: 2012-09-07 21:27
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

Oracle Database のログ出力先を変更する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* 管理者ガイド  
  -> 9 診断データの管理  
  -> [自動診断リポジトリの構造、内容および場所](http://docs.oracle.com/cd/E16338_01/server.112/b56301/diag001.htm#CHDIABAA)

* リファレンス  
  -> 1 初期化パラメータ  
  -> [DIAGNOSTIC_DEST](http://docs.oracle.com/cd/E16338_01/server.112/b56311/initparams077.htm#I1010280)

* Net Services管理者ガイド  
  -> 16 Oracle Net Servicesのトラブルシューティング  
  -> [自動診断リポジトリの理解](http://docs.oracle.com/cd/E16338_01/network.112/b56288/trouble.htm#BABHDHGG)

## 設定

Oracle Database 11gR1 より、各種ログの出力先が"自動診断リポジトリ(Automatic Diagnostic Repository, ADR)"という構造になった。

* Oracle Database のログ

Oracle Database のログの出力先は初期化パラメータ `DIAGNOSTIC_DEST` にて設定する。
例えば、`/home/oracle` 配下に変更する場合は以下のようにパラメータ変更を行う。
(動的に変更可能なので DB 再起動は不要。)

    $ sqlplus '/as sysdba'
    SQL> ALTER SYSTEM SET DIAGNOSTIC_DEST = "/home/oracle" SCOPE = BOTH;
    
    System altered.

指定したディレクトリ配下に `diag/rdbms` ディレクトリが作成され、その配下に alert.log やトレースが出力される。

* Listener のログ

Listener のログ出力先は `listener.ora` 内の `ADR_BASE_<リスナー名>` で設定する。

    $ vi $ORACLE_HOME/network/admin/listener.ora # 以下を編集 or 追加
    ADR_BASE_LISTENER = /home/oracle

設定を反映させるために Listener を再起動

    $ lsnrctl stop
    $ lsnrctl start

これで、指定したディレクトリ配下に `diag/tnslsnr` ディレクトリができて、その配下に listener.log が出力される。

* Oracle Net Client のログ

Oracle Net Client のログは `sqlnet.ora` 内の `ADR_BASE` パラメータで設定する。

    $ vi $ORACLE_HOME/network/admin/sqlnet.ora # 以下を編集 or 追加
    ADR_BASE = /home/oracle

これで、ログ出力時は指定したディレクトリ配下に `diag/clients` ディレクトリができて、そこに出力される。

