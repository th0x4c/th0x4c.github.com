---
layout: post
title: "[Oracle] BasicFiles LOB から SecureFiles LOB への移行"
date: 2012-09-16 14:15
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

従来型の LOB (BasicFiles LOB) から、Oracle Database 11g からの新しい LOB アーキテクチャである SecureFiles を使用した LOB へ移行する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* SecureFilesおよびラージ・オブジェクト開発者ガイド  
  -> 4 Oracle SecureFiles LOBの使用  
  -> [BasicFiles LOBからSecureFiles LOBへの列の移行](http://docs.oracle.com/cd/E16338_01/appdev.112/b56263/adlob_smart.htm#BABDIEGE)  
     [SecureFiles LOBの初期化パラメータdb_securefile](http://docs.oracle.com/cd/E16338_01/appdev.112/b56263/adlob_smart.htm#BABJFEBB)  
     [SecureFiles LOBを含んだCREATE TABLEの使用](http://docs.oracle.com/cd/E16338_01/appdev.112/b56263/adlob_smart.htm#CIHGHEFA)  
     [SecureFiles LOBを含んだALTER TABLEの使用](http://docs.oracle.com/cd/E16338_01/appdev.112/b56263/adlob_smart.htm#CIHJJBIJ)

* 管理者ガイド  
  -> 20 表の管理  
  -> [表のオンライン再定義](http://docs.oracle.com/cd/E16338_01/server.112/b56301/tables007.htm#i1006754)

## BasicFiles LOB から SecureFiles LOB への移行

Oracle Database 11g からの新しい LOB アーキテクチャである SecureFiles を使用した LOB を使用すると、以下のようなことが可能になる。

* 圧縮(LOB データの圧縮)
* 重複除外(列内で重複する LOB データのコピーを1つのみ格納)
* 暗号化(透過的データ暗号化により LOB データを暗号化)

従来型の LOB は、BasicFiles と呼ばれる。BasicFiles を使用した既存の LOB 列を SecureFiles に移行する。`ALTER TABLE` での BasicFiles から SecureFiles への変更はできないため、変更するためには表の再作成を行うか、オンライン再定義を行う必要がある。今回は、マニュアルに例があるように、オンライン再定義により行う。

* 検証用に BasicFiles LOB 列を持つ表、データを準備

BasicFiles LOB 列を持つ `cust` 表を作成してデータを insert。今回この表を SecureFiles LOB に移行してみる。

    SQL> connect scott/tiger
    SQL> CREATE TABLE cust
         (
           c_id  NUMBER PRIMARY KEY,
           c_zip NUMBER,
           c_name VARCHAR(30) DEFAULT NULL,
           c_lob CLOB
         );

    Table created.

    SQL> INSERT INTO cust VALUES(1, 94065, 'hhh', 'ttt');

    1 row created.

    SQL> COMMIT;

    Commit complete.

* オンライン再定義に必要な権限の付与

オンライン再定義に必要な権限を与える

    SQL> connect /as sysdba
    SQL> -- Grant privileges required for online redefinition.
    SQL> GRANT EXECUTE ON DBMS_REDEFINITION TO scott;
    SQL> GRANT ALTER ANY TABLE TO scott;
    SQL> GRANT DROP ANY TABLE TO scott;
    SQL> GRANT LOCK ANY TABLE TO scott;
    SQL> GRANT CREATE ANY TABLE TO scott;
    SQL> GRANT SELECT ANY TABLE TO scott;
    SQL> -- Privileges required to perform cloning of dependent objects.
    SQL> GRANT CREATE ANY TRIGGER TO scott;
    SQL> GRANT CREATE ANY INDEX TO scott;

* 初期化パラメータ `db_securefile` パラメータの変更

SecureFiles LOB の初期化パラメータ `db_securefile` パラメータを変更する。
SecureFiles LOB を使用するためには `PERMITTED` (デフォルト) または `ALWAYS` である必要がある。

例えば、`NEVER` に設定されている場合に SecureFiles LOB の列を作成しようとすると、`ORA-43856` が発生する。

    SQL> CREATE TABLE test_lob (c_lob CLOB) LOB(c_lob) STORE AS SECUREFILE (compress high);
    CREATE TABLE test_lob (c_lob CLOB) LOB(c_lob) STORE AS SECUREFILE (compress high)
    *
    ERROR at line 1:
    ORA-43856: Unsupported LOB type for SECUREFILE LOB operation

`db_securefile` を `ALTER SYSTEM` で変更する。

    SQL> conn /as sysdba
    Connected.
    SQL> ALTER SYSTEM SET db_securefile = PERMITTED SCOPE=both;

    System altered.

* 仮表を作成

オンライン再定義のための仮表を SecureFiles LOB 列を持つようにして作成。元表からコピーされるので主キーなどの制約の指定は不要。

    SQL> connect scott/tiger
    Connected.
    SQL> CREATE TABLE cust_int
         (
           c_id  NUMBER,
           c_zip NUMBER,
           c_name VARCHAR(30) DEFAULT NULL,
           c_lob CLOB
         )
         LOB(c_lob) STORE AS SECUREFILE;
    
    Table created.

* 表のオンライン再定義

表のオンライン再定義を実行する。

    SQL> DECLARE
           col_mapping VARCHAR2(1000);
         BEGIN
           -- map all the columns in the interim table to the original table
           col_mapping :=
             'c_id c_id , '||
             'c_zip c_zip , '||
             'c_name c_name, '||
             'c_lob c_lob';
           DBMS_REDEFINITION.START_REDEF_TABLE('scott', 'cust', 'cust_int', col_mapping);
         END;
         /

    PL/SQL procedure successfully completed.
    
    SQL> set serveroutput on
    SQL> DECLARE
           error_count pls_integer := 0;
         BEGIN
           DBMS_REDEFINITION.COPY_TABLE_DEPENDENTS('scott', 'cust', 'cust_int',
             1, TRUE,TRUE,TRUE,FALSE, error_count);
           DBMS_OUTPUT.PUT_LINE('errors := ' || TO_CHAR(error_count));
         END;
         /
    errors := 0

    PL/SQL procedure successfully completed.
    
    SQL> EXEC DBMS_REDEFINITION.FINISH_REDEF_TABLE('scott', 'cust', 'cust_int');
    
    PL/SQL procedure successfully completed.
    
    SQL> DROP TABLE cust_int;

    Table dropped.

* SecureFiles LOB に移行されたことを確認

`USER_LOBS.SECUREFILE` 列が `YES` であれば SecureFiles になっている。

    SQL> col table_name for a30
    SQL> col column_name for a30
    SQL> SELECT table_name, column_name, securefile FROM user_lobs;

    TABLE_NAME                     COLUMN_NAME                    SECUREFIL
    ------------------------------ ------------------------------ ---------
    CUST                           C_LOB                          YES

## SecureFiles LOB の変更

`ALTER TABLE` 文により既存の SecureFiles LOB 列を変更して、圧縮や重複除外の設定を行う。

* 現在の設定の確認

`USER_LOBS` にて確認できる。

    SQL> col table_name for a30
         col column_name for a30
         col encrypt for a10
         col compression for a15
         col deduplication for a15
         set lines 200
    SQL> SELECT table_name, column_name, encrypt, compression, deduplication FROM user_lobs;

    TABLE_NAME                     COLUMN_NAME                    ENCRYPT    COMPRESSION     DEDUPLICATION
    ------------------------------ ------------------------------ ---------- --------------- ---------------
    CUST                           C_LOB                          NO         NO              NO

* 設定の変更

`ALTER TABLE` により圧縮、重複除外の設定を行う。

    SQL> ALTER TABLE cust MODIFY LOB(c_lob) (COMPRESS HIGH);

    Table altered.

    SQL> ALTER TABLE cust MODIFY LOB(c_lob) (DEDUPLICATE);

    Table altered.

* 設定変更の確認

`USER_LOBS` にて確認。

    SQL> col table_name for a30
         col column_name for a30
         col encrypt for a10
         col compression for a15
         col deduplication for a15
         set lines 200
    SQL> SELECT table_name, column_name, encrypt, compression, deduplication FROM user_lobs;

    TABLE_NAME                     COLUMN_NAME                    ENCRYPT    COMPRESSION     DEDUPLICATION
    ------------------------------ ------------------------------ ---------- --------------- ---------------
    CUST                           C_LOB                          NO         HIGH            LOB
