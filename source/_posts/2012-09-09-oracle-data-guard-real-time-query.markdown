---
layout: post
title: "[Oracle] Data Guard リアルタイム問い合わせ"
date: 2012-09-09 22:28
comments: true
categories: [Oracle, DB, 11gR2, EM, Data Guard]
---
## 目的

Data Guard の "リアルタイム問い合わせ" 機能により、REDO 適用を中断せずプライマリの更新をリアルタイムで適用しつつ、スタンバイ DB を読み取り専用でオープンする。

## 環境

* OS: Oracle Enterprise Linux 5.8
* DB: Oracle Database 11g Release 2 (11.2.0.3)
* EM: Oracle Enterprise Manager Grid Control 11g (11.1)

## リアルタイム問い合わせ

Data Guard の "リアルタイム問い合わせ" 機能により、REDO 適用を中断せずプライマリの更新をリアルタイムで適用しつつ、スタンバイ DB を読み取り専用でオープンすることができる。

今回は EM Grid Control を使用してこの機能を有効にする。

* プライマリ DB インスタンスの "可用性" タブ内の Data Guard の項目から "設定および管理" をクリックして、Data Guard のページを開く

{% img /images/2012-09-09-oracle-creating-a-physical-standby-database/dg-14.png 720 450 %}

* "リアルタイム問い合わせ" の "無効" をクリック

{% img /images/2012-09-09-oracle-data-guard-real-time-query/dgrq-1.png 720 450 %}

* "リアルタイム問い合わせの有効化" をチェックして、"適用" をクリック

{% img /images/2012-09-09-oracle-data-guard-real-time-query/dgrq-2.png 720 450 %}

* 処理中 -> 正常に適用された

{% img /images/2012-09-09-oracle-data-guard-real-time-query/dgrq-3.png 720 450 %}

{% img /images/2012-09-09-oracle-data-guard-real-time-query/dgrq-4.png 720 450 %}

* Data Guard の構成画面に戻ると "リアルタイム問い合わせ" が "有効" になっている

{% img /images/2012-09-09-oracle-data-guard-real-time-query/dgrq-5.png 720 450 %}

実際にスタンバイ DB に接続すると、SELECT が実行可能でプライマリの変更が REDO 適用された時点で即時反映されることが確認できる。

    SQL> connect scott/tiger@PROD1 -- プライマリに接続
    Connected.
    SQL> CREATE TABLE emp2 AS SELECT * FROM emp;

    Table created.

    SQL> connect scott/tiger@STDBY1_DGMGRL -- スタンバイに接続
    Connected.
    SQL> SELECT * FROM emp2 WHERE ROWNUM = 1; -- プライマリの更新が反映されている

         EMPNO ENAME                          JOB                                MGR
    ---------- ------------------------------ --------------------------- ----------
    HIREDATE        SAL       COMM     DEPTNO
    -------- ---------- ---------- ----------
          7369 SMITH                          CLERK                             7902
    80-12-17        800                    20


    SQL> UPDATE emp2 SET sal = sal * 10 WHERE empno = 7369; -- スタンバイの更新はできない
    UPDATE emp2 SET sal = sal * 10 WHERE empno = 7369
           *
    ERROR at line 1:
    ORA-16000: database open for read-only access

