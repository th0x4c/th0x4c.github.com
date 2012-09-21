---
layout: post
title: "[Oracle] フラッシュバック・データ・アーカイブ(Oracle Total Recall)"
date: 2012-09-16 17:16
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

フラッシュバック・データ・アーカイブ(Oracle Total Recall) を使用して、表への更新履歴を長期間保存する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* アドバンスト・アプリケーション開発者ガイド  
  -> 12 Oracle Flashback Technologyの使用  
  -> [フラッシュバック・データ・アーカイブの使用(Oracle Total Recall)](http://docs.oracle.com/cd/E16338_01/appdev.112/b56259/adfns_flashback.htm#BJFFDCEH)

## フラッシュバック・データ・アーカイブの使用

フラッシュバック・データ・アーカイブ(Oracle Total Recall) を使用して、表への更新履歴を長期間保存できる。例えば、コンプライアンス上変更履歴を年単位で長時間保存しておく必要がある場合などに有効。

* フラッシュバック・データ・アーカイブの作成

`CREATE FLASHBACK ARCHIVE` 文を使用してフラッシュバック・データ・アーカイブを作成する。
データを1年間保持するデフォルトのフラッシュバック・データ・アーカイブ `fla1` を `USERS` 表領域に作成。
作成するには `FLASHBACK ARCHIVE ADMINISTER` システム権限を持つユーザか、SYSDBA として接続する必要がある。

    SQL> connect /as sysdba
    Connected.
    SQL> CREATE FLASHBACK ARCHIVE DEFAULT fla1 TABLESPACE users
         RETENTION 1 YEAR;

    Flashback archive created.

* 表のフラッシュバック・アーカイブの有効化

`scott.emp` 表のフラッシュバック・アーカイブを有効にして、フラッシュバック・データ・アーカイブ `fla1` に履歴データを格納するようにする。
`scott` スキーマに `fla1` に対する `FLASHBACK ARCHIVE` オブジェクト権限を与えて、`ALTER TABLE` 文を発行する。

    SQL> connect /as sysdba
    Connected.
    SQL> GRANT FLASHBACK ARCHIVE ON fla1 TO scott;
    
    Grant succeeded.
    
    SQL> connect scott/tiger
    Connected.
    SQL> ALTER TABLE scott.emp FLASHBACK ARCHIVE fla1;

    Table altered.

* フラッシュバック・データ・アーカイブが有効になっていることの確認

`scott` スキーマで `emp` 表のフラッシュバック・データ・アーカイブが有効になっていることの確認

    SQL> connect scott/tiger
    Connected.
    SQL> col table_name for a30
         col owner_name for a30
         col flashback_archive_name for a30
         col archive_table_name for a30
         col status for a10
         set lines 200
    SQL> SELECT * FROM USER_FLASHBACK_ARCHIVE_TABLES;

    TABLE_NAME                     OWNER_NAME                     FLASHBACK_ARCHIVE_NAME         ARCHIVE_TABLE_NAME             STATUS
    ------------------------------ ------------------------------ ------------------------------ ------------------------------ ----------
    EMP                            SCOTT                          FLA1                           SYS_FBA_HIST_75335             ENABLED
