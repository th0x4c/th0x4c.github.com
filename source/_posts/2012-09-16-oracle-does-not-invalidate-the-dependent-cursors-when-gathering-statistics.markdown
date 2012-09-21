---
layout: post
title: "[Oracle] カーソルを無効にしない統計情報の取得"
date: 2012-09-16 21:34
comments: true
categories: [Oracle, DB, 11gR2, Performance Tuning]
---
## 目的

カーソルを無効化せずに統計情報を取得する。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)

## マニュアル

* PL/SQLパッケージ・プロシージャおよびタイプ・リファレンス  
  -> 141 DBMS_STATS  
  -> [GATHER_TABLE_STATSプロシージャ](http://docs.oracle.com/cd/E16338_01/appdev.112/b56262/d_stats.htm#i1036461)  
     [SET_TABLE_PREFSプロシージャ](http://docs.oracle.com/cd/E16338_01/appdev.112/b56262/d_stats.htm#BEIBJJHC)

## カーソルを無効にしない統計情報の取得

カーソルを無効化せずに統計情報を取得する。
方法としては、以下の2つがある。

1. `DBMS_STATS.GATHER_TABLE_STATS` プロシージャ実行時に指定する方法
2. `SET_*_PREFS` プロシージャによりデフォルト値を変更する方法

## `DBMS_STATS.GATHER_TABLE_STATS` プロシージャ実行時に指定する方法

`DBMS_STATS.GATHER_TABLE_STATS` 実行時に指定する場合は、`no_invalidate` パラメータを `TRUE` にする。

    SQL> EXEC DBMS_STATS.GATHER_TABLE_STATS('scott', 'emp', no_invalidate => TRUE);

    PL/SQL procedure successfully completed.

## `SET_*_PREFS` プロシージャによりデフォルト値を変更する方法

表単位でデフォルト値を変更する場合は `SET_TABLE_PREFS` プロシージャを使う。

`GET_PREFS` プロシージャで現在の設定の確認。

    SQL> SELECT DBMS_STATS.GET_PREFS('NO_INVALIDATE', 'scott', 'emp') FROM DUAL;

    DBMS_STATS.GET_PREFS('NO_INVALIDATE','SCOTT','EMP')
    ---------------------------------------------------
    DBMS_STATS.AUTO_INVALIDATE

`SET_TABLE_PREFS` プロシージャにて設定の変更。

    SQL> EXEC DBMS_STATS.SET_TABLE_PREFS('scott', 'emp', 'NO_INVALIDATE', 'TRUE');

    PL/SQL procedure successfully completed.

`GET_PREFS` プロシージャで設定が変更されたことを確認。

    SQL> SELECT DBMS_STATS.GET_PREFS('NO_INVALIDATE', 'scott', 'emp') FROM DUAL;

    DBMS_STATS.GET_PREFS('NO_INVALIDATE','SCOTT','EMP')
    ---------------------------------------------------
    TRUE

あとは `no_invalidate` オプションを明示的に指定せずに統計情報を採取すると変更した内容がデフォルトとして採用される。

    SQL> EXEC DBMS_STATS.GATHER_TABLE_STATS('scott', 'emp');
    
    PL/SQL procedure successfully completed.

