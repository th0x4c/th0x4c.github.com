---
layout: post
title: "[Oracle] パスワードの管理"
date: 2012-09-07 22:09
comments: true
categories: [Oracle, DB, 11gR2]
---
## 目的

パスワードを大文字・小文字の区別をするようにする。
DB のパスワードについてはその他複雑なパスワードにするように強制する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* 管理者ガイド  
  -> 1 データベース管理スタート・ガイド  
  -> [パスワード・ファイルの作成とメンテナンス](http://docs.oracle.com/cd/E16338_01/server.112/b56301/dba007.htm#i1006789)

* セキュリティ・ガイド  
  -> 3 認証の構成  
  -> [パスワードでの大/小文字の区別の有効化または無効化](http://docs.oracle.com/cd/E16338_01/network.112/b56285/authentication.htm#CHDJDCGI)  
  [パスワードの複雑度検証の規定](http://docs.oracle.com/cd/E16338_01/network.112/b56285/authentication.htm#i1007341)


## SYSDBA でのログイン時のパスワード

SYSDBA でのログイン(SYS ユーザで as sysdba でのログイン)時のパスワードを大文字・小文字の区別させるためにパスワードファイル(`$ORACLE_HOME/dbs/orapw<ORACLE_SID>`)を再作成する。

`orapwd`を引数なしで実行すると、使い方が分かる。

    $ orapwd
    Usage: orapwd file=<fname> entries=<users> force=<y/n> ignorecase=<y/n> nosysdba=<y/n>

      where
        file - name of password file (required),
        password - password for SYS will be prompted if not specified at command line,
        entries - maximum number of distinct DBA (optional),
        force - whether to overwrite existing file (optional),
        ignorecase - passwords are case-insensitive (optional),
        nosysdba - whether to shut out the SYSDBA logon (optional Database Vault only).
        
      There must be no spaces around the equal-to (=) character.

`ignorecase=n` で再作成すれば大文字・小文字が区別される。
ちなみにパスワードファイルのファイル名は `orapw<ORACLE_SID>` でなければならない。

    $ cd $ORACLE_HOME/dbs
    $ rm orapwPROD1
    $ orapwd file=orapwPROD1 password=oracle ignorecase=n

大文字・小文字が区別されるか確認。

    SQL> connect sys/oracle@PROD1 as sysdba
    Connected.
    SQL> connect sys/ORACLE@PROD1 as sysdba
    ERROR:
    ORA-01017: invalid username/password; logon denied


    Warning: You are no longer connected to ORACLE.

## DB ユーザのパスワード

### 大文字・小文字の区別

DB ユーザのパスワードの大文字・小文字を区別するためには初期かパラメータ `SEC_CASE_SENSITIVE_LOGON` を `TRUE` にする。(デフォルトは `TRUE`)

    SQL> ALTER SYSTEM SET SEC_CASE_SENSITIVE_LOGON = TRUE;

    System altered.

### 複雑なパスワードを強制

`$ORACLE_HOME/rdbms/admin/utlpwdmg.sql` を実行することで、デフォルトのプロファイルを使用するユーザがパスワード変更する際に以下がチェックされるようになる。

* パスワードの長さが8文字以上30文字以内であること。
* パスワードがユーザー名と同一でないこと。ユーザー名のスペルを逆にしたり、ユーザー名に数字を追加したパスワードでないこと。
* サーバー名と同一であったり、サーバー名に数字1から100を追加したパスワードでないこと。
* 単純なパスワードでないこと(例: welcome1、database1、account1、user1234、password1、oracle、oracle123、computer1、abcdefg1、change_on_install)。
* パスワードに少なくとも数字が1つと英字が1つ含まれていること。
* 以前のパスワードとの違いが3文字以上あること。

`$ORACLE_HOME/rdbms/admin/utlpwdmg.sql` の内容を確認すると分かるように、パスワード検証関数 `verify_function_11G` が作成されてデフォルトプロファイル(`DEFAULT`)が変更されている。(パスワードの存続期間等も設定される)

    $ cat $ORACLE_HOME/rdbms/admin/utlpwdmg.sql
    ...(*snip*)

    -- This script sets the default password resource parameters
    -- This script needs to be run to enable the password features.
    -- However the default resource parameters can be changed based
    -- on the need.
    -- A default password complexity function is also provided.
    -- This function makes the minimum complexity checks like
    -- the minimum length of the password, password not same as the
    -- username, etc. The user may enhance this function according to
    -- the need.
    -- This function must be created in SYS schema.
    -- connect sys/<password> as sysdba before running the script

    CREATE OR REPLACE FUNCTION verify_function_11G
    ...(*snip*)
    END;
    /

    -- This script alters the default parameters for Password Management
    -- This means that all the users on the system have Password Management
    -- enabled and set to the following values unless another profile is
    -- created with parameter values set to different value or UNLIMITED
    -- is created and assigned to the user.

    ALTER PROFILE DEFAULT LIMIT
    PASSWORD_LIFE_TIME 180
    PASSWORD_GRACE_TIME 7
    PASSWORD_REUSE_TIME UNLIMITED
    PASSWORD_REUSE_MAX UNLIMITED
    FAILED_LOGIN_ATTEMPTS 10
    PASSWORD_LOCK_TIME 1
    PASSWORD_VERIFY_FUNCTION verify_function_11G;
    ...(*snip*)

ということで、`utlpwdmg.sql` を実行する。

    $ sqlplus '/as sysdba'
    SQL> @?/rdbms/admin/utlpwdmg.sql

    Function created.


    Profile altered.


    Function created.

他のプロファイルも変更する場合は、`ALTER PROFILE` を実行する必要がある。

    SQL> ALTER PROFILE <profile名> LIMIT
         PWSSWORD_VERIFY_FUNCTION verify_function_11G;

実際にパスワードを変更してチェックに引っかかるか確認してみる。

    SQL> alter user scott identified by tiger;
    alter user scott identified by tiger
    *
    ERROR at line 1:
    ORA-28003: password verification for the specified password failed
    ORA-20001: Password length less than 8

