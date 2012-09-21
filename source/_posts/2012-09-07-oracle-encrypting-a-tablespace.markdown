---
layout: post
title: "[Oracle] 暗号化された表領域の作成"
date: 2012-09-07 17:58
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

暗号化された表領域を作成する。  
"透過的データ暗号化"という機能で、表の列もしくは表領域全体を暗号化できるが、今回は表領域の暗号化を行う。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* 2日でセキュリティ・ガイド  
  -> 6 データの保護  
  -> [透過的データ暗号化を使用するためのデータの構成](http://docs.oracle.com/cd/E16338_01/server.112/b56296/tdpsg_securing_data.htm#CHDBHJEG)

* 管理者ガイド  
  -> 14 表領域の管理  
  -> [暗号化された表領域](http://docs.oracle.com/cd/E16338_01/server.112/b56301/tspaces002.htm#BABBJGAF)

## 作成

"透過的データ暗号化"を使用して表領域を暗号化するためには以下の手順が必要。

1. ウォレット(マスター暗号化キーが格納される)を DB 外の OS 上のファイルとして作成
2. ウォレットを開く(データの暗号化が開始される)
3. 表領域の作成

以下、マニュアルの手順に従い作成する。

* ウォレットを格納するディレクトリを作成

OS 上でディレクトリ作成

    $ mkdir /home/oracle/ORA_WALLETS

* sqlnet.ora ファイルにウォレットを格納するディレクトリを指定

sqlnet.ora ファイルを編集

    $ vi $ORACLE_HOME/network/admin/sqlnet.ora
    $ cat $ORACLE_HOME/network/admin/sqlnet.ora
    ENCRYPTION_WALLET_LOCATION=
     (SOURCE=
      (METHOD=file)
       (METHOD_DATA=
        (DIRECTORY=/home/oracle/ORA_WALLETS)))

* (ここで compatible 初期化バラメータの値が 10.2 より小さい場合は DB 再起動が必要)

* ウォレットを作成

パスワード(以下の例では `"oracle"`)を指定する。このとき、パスワードは二重引用府(")で囲む必要がある。
これでウォレット格納ディレクトリにウォレットの OS ファイルができる。

    $ sqlplus '/as sysdba'
    SQL> ALTER SYSTEM SET ENCRYPTION KEY IDENTIFIED BY "oracle";

    System altered.

    $ ls /home/oracle/ORA_WALLETS
    ewallet.p12

* ウォレットを開く

以下の `ALTER SYSTEM` によりウォレットを開く。この場合もパスワードは二重引用符で囲む必要がある。
ちなみにウォレット作成直後は開かれた状態となっている。

    SQL> ALTER SYSTEM SET ENCRYPTION WALLET OPEN IDENTIFIED BY "oracle";
    
    System altered.

* 暗号化された表領域の作成

以下のように、デフォルトの暗号化アルゴリズムを使用して暗号化された表領域を作成。暗号化アルゴリズムを指定する場合は `ENCRYPTION USING 'AES256'` などとする。

    SQL> CREATE TABLESPACE securespace
         DATAFILE '/u01/app/oracle/oradata/PROD1/secure01.dbf' SIZE 100M
         ENCRYPTION
         DEFAULT STORAGE(ENCRYPT);

    Tablespace created.

## 確認方法

* ウォレットが開いているか閉じているかのチェック

`V$ENCRYPTION_VIEW` ビューを問い合わせる。

    SQL> SELECT * FROM V$ENCRYPTION_WALLET;

    WRL_TYPE   WRL_PARAMETER                  STATUS
    ---------- ------------------------------ ----------
    file       /home/oracle/ORA_WALLETS       OPEN

* 表領域が暗号化されているかのチェック

`DBA_TABLESPACES.ENCRYPTED` 列を確認する。

    SQL> SELECT TABLESPACE_NAME, ENCRYPTED FROM DBA_TABLESPACES;
    
    TABLESPACE_NAME      ENCRYPTED
    -------------------- ----------
    SYSTEM               NO
    SYSAUX               NO
    UNDOTBS1             NO
    TEMP                 NO
    USERS                NO
    EXAMPLE              NO
    SECURESPACE          YES