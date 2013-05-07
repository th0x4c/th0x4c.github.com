---
layout: post
title: "[OS] メモリリークの調査方法"
date: 2013-04-06 17:29
comments: true
categories: [OS, Linux]
---
## 目的

メモリリークの調査方法をまとめる。

## 環境

* OS: CentOS 5.5
* Kernel: 2.6.18-194.el5 x86_64
* GCC: gcc 4.1.2 20080704
* GDB: GNU gdb 7.0.1-23.el5
* Valgrind: valgrind-3.5.0

## サンプルプログラム

メモリリークが起きるサンプルとして以下を利用する。
`leak_func()` が実行される度に 2048 bytes メモリリークする。
合計で 101 回 `leak_func()` が実行されるので 206848bytes(= 2048 * 101 bytes) リークする。

{% codeblock lang:c %}
#include <stdio.h>
#include <stdlib.h>

#define STR_BYTES 2048

void *my_alloc(size_t size)
{
  void *ret = malloc(size);

  if (ret == NULL)
  {
    fprintf(stdout, "Cannot malloc struct\n");
    exit(1);
  }

  return ret;
}

void my_free(void *ptr)
{
  free(ptr);
}

void leak_func()
{
  char *str = NULL;
  char *leak_str = NULL;

  str = (char *)my_alloc(STR_BYTES);
  leak_str = (char *)my_alloc(STR_BYTES);

  snprintf(str, STR_BYTES, "freed memory");
  snprintf(leak_str, STR_BYTES, "leaked memory");

  printf("%s: 0x%016lx, ", str, str);
  printf("%s: 0x%016lx\n", leak_str, leak_str);

  my_free((void *)str);
  leak_str = NULL; /* leak_str が free() されていないのでリークする */
}

int main(int argc, char *argv[])
{
  int i = 0;

  leak_func();
  printf("Press enter key:");
  getchar();

  for (i = 0; i < 100; ++i)
    leak_func();

  printf("Press enter key:");
  getchar();
  return 0;
}
{% endcodeblock %}

実行結果は以下。

    $ gcc -o memory_leak_sample memory_leak_sample.c
    $ ./memory_leak_sample
    freed memory: 0x0000000009a19010, leaked memory: 0x0000000009a19820
    Press enter key:
    freed memory: 0x0000000009a19010, leaked memory: 0x0000000009a1a030
    freed memory: 0x0000000009a19010, leaked memory: 0x0000000009a1a840
    freed memory: 0x0000000009a19010, leaked memory: 0x0000000009a1b050
    <中略>
    freed memory: 0x0000000009a19010, leaked memory: 0x0000000009a4b650
    freed memory: 0x0000000009a19010, leaked memory: 0x0000000009a4be60
    Press enter key:


## valgrind による調査

[Valgrind](http://valgrind.org) は、メモリリークの検出等を行うツール。
プロファイリング等メモリリーク検出以外の機能もあり、メモリリー検出で使用する場合は `--tool=memcheck` を指定する。

Valgrind を使用して上記サンプルを動作させた例が以下。なお、Valgrind を通してプログラムを実行するとすごく遅くなるので注意。

    $ valgrind --tool=memcheck --leak-check=yes --leak-resolution=high --num-callers=40 --undef-value-errors=no --run-libc-freeres=no -v ./memory_leak_sample
    ==16207== Memcheck, a memory error detector
    ==16207== Copyright (C) 2002-2009, and GNU GPL'd, by Julian Seward et al.
    ==16207== Using Valgrind-3.5.0 and LibVEX; rerun with -h for copyright info
    ==16207== Command: ./memory_leak_sample
    ==16207==
    --16207-- Valgrind options:
    --16207--    --tool=memcheck
    --16207--    --leak-check=yes
    --16207--    --leak-resolution=high
    --16207--    --num-callers=40
    --16207--    --undef-value-errors=no
    --16207--    --run-libc-freeres=no
    --16207--    -v
    --16207-- Contents of /proc/version:
    --16207--   Linux version 2.6.18-194.el5 (mockbuild@builder10.centos.org) (gcc version 4.1.2 20080704 (Red Hat 4.1.2-48)) #1 SMP Fri Apr 2
     14:58:14 EDT 2010
    --16207-- Arch and hwcaps: AMD64, amd64-sse3-cx16
    --16207-- Page sizes: currently 4096, max supported 4096
    --16207-- Valgrind library directory: /usr/lib64/valgrind
    --16207-- Reading syms from /home/hashi/tmp/memory_leak_sample (0x400000)
    --16207-- Reading syms from /usr/lib64/valgrind/memcheck-amd64-linux (0x38000000)
    --16207--    object doesn't have a dynamic symbol table
    --16207-- Reading syms from /lib64/ld-2.5.so (0x3e1c600000)
    --16207-- Reading suppressions file: /usr/lib64/valgrind/default.supp
    --16207-- REDIR: 0x3e1c614620 (strlen) redirected to 0x3803e767 (vgPlain_amd64_linux_REDIR_FOR_strlen)
    --16207-- Reading syms from /usr/lib64/valgrind/vgpreload_core-amd64-linux.so (0x4802000)
    --16207-- Reading syms from /usr/lib64/valgrind/vgpreload_memcheck-amd64-linux.so (0x4a03000)
    ==16207== WARNING: new redirection conflicts with existing -- ignoring it
    --16207--     new: 0x3e1c614620 (strlen              ) R-> 0x04a06dc0 strlen
    --16207-- REDIR: 0x3e1c614440 (index) redirected to 0x4a06c30 (index)
    --16207-- REDIR: 0x3e1c6145f0 (strcmp) redirected to 0x4a06e90 (strcmp)
    --16207-- Reading syms from /lib64/libc-2.5.so (0x3e1ca00000)
    --16207-- REDIR: 0x3e1ca79ba0 (rindex) redirected to 0x4a06ae0 (rindex)
    --16207-- REDIR: 0x3e1ca74c70 (malloc) redirected to 0x4a05d9a (malloc)
    --16207-- REDIR: 0x3e1ca797b0 (strlen) redirected to 0x4a06d80 (strlen)
    freed memory: 0x0000000004c24040, leaked memory: 0x0000000004c24880
    --16207-- REDIR: 0x3e1ca72720 (free) redirected to 0x4a059aa (free)
    Press enter key:
    freed memory: 0x0000000004c250c0, leaked memory: 0x0000000004c25900
    freed memory: 0x0000000004c26140, leaked memory: 0x0000000004c26980
    <中略>
    freed memory: 0x0000000004c89140, leaked memory: 0x0000000004c89980
    freed memory: 0x0000000004c8a1c0, leaked memory: 0x0000000004c8aa00
    freed memory: 0x0000000004c8b240, leaked memory: 0x0000000004c8ba80
    Press enter key:
    ==16207==
    ==16207== HEAP SUMMARY:
    ==16207==     in use at exit: 206,848 bytes in 101 blocks
    ==16207==   total heap usage: 202 allocs, 101 frees, 413,696 bytes allocated
    ==16207==
    ==16207== Searching for pointers to 101 not-freed blocks
    ==16207== Checked 73,400 bytes
    ==16207==
    ==16207== 2,048 bytes in 1 blocks are definitely lost in loss record 1 of 2
    ==16207==    at 0x4A05E1C: malloc (vg_replace_malloc.c:195)
    ==16207==    by 0x40069C: my_alloc (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x400719: leak_func (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x4007AE: main (in /home/hashi/tmp/memory_leak_sample)
    ==16207==
    ==16207== 204,800 bytes in 100 blocks are definitely lost in loss record 2 of 2
    ==16207==    at 0x4A05E1C: malloc (vg_replace_malloc.c:195)
    ==16207==    by 0x40069C: my_alloc (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x400719: leak_func (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x4007D5: main (in /home/hashi/tmp/memory_leak_sample)
    ==16207==
    ==16207== LEAK SUMMARY:
    ==16207==    definitely lost: 206,848 bytes in 101 blocks
    ==16207==    indirectly lost: 0 bytes in 0 blocks
    ==16207==      possibly lost: 0 bytes in 0 blocks
    ==16207==    still reachable: 0 bytes in 0 blocks
    ==16207==         suppressed: 0 bytes in 0 blocks
    ==16207==
    ==16207== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
    ==16207== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)

いろいろ出力があるが、以下の出力からどの Call Stack で確保されたメモリがどれだけリークしているか分かる。

    ==16207== 2,048 bytes in 1 blocks are definitely lost in loss record 1 of 2
    ==16207==    at 0x4A05E1C: malloc (vg_replace_malloc.c:195)
    ==16207==    by 0x40069C: my_alloc (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x400719: leak_func (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x4007AE: main (in /home/hashi/tmp/memory_leak_sample)
    ==16207==
    ==16207== 204,800 bytes in 100 blocks are definitely lost in loss record 2 of 2
    ==16207==    at 0x4A05E1C: malloc (vg_replace_malloc.c:195)
    ==16207==    by 0x40069C: my_alloc (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x400719: leak_func (in /home/hashi/tmp/memory_leak_sample)
    ==16207==    by 0x4007D5: main (in /home/hashi/tmp/memory_leak_sample)

上述のやり方だと、シェルから valgrind と共に直接実行できるプログラムでないと調査できない。
デーモンなど直接実行できないプログラムの場合は、ラップするシェルスクリプトを用意するとよい。

例えば、最終的に `some_daemon.bin` というバイナリが実行されるデーモンがあるとして、以下の
ようにラップするシェルスクリプトを同一名で用意して、元のデーモンと同じように起動・停止すればよい。

    $ cp -p /path/to/some_daemon.bin /path/to/some_daemon.bin.backup
    $ mv /path/to/some_daemon.bin /path/to/some_daemon.bin.orig
    $ vi some_daemon.bin # 以下の内容で作成
    --------
    #!/bin/sh
    
    ORG_BIN=/path/to/some_daemon.bin.orig
    LOG_LOC_AND_PREFIX=/tmp/valgrind_instance.%p.log
    VALG_PATH=/usr/bin/valgrind
    VALGRIND_OPTS="--log-file=$LOG_LOC_AND_PREFIX --leak-check=yes --leak-resolution=high --num-callers=40 --undef-value-errors=no --run-libc-freeres=no --error-limit=no -v"
    export VALGRIND_OPTS
    
    exec $VALG_PATH --tool=memcheck $ORG_BIN "$*"
    --------
    $ chown root:root some_daemon.bin # 元のバイナリと同じオーナーにする
    $ chmod 755 some_daemon.bin       # 元のバイナリと同じパーミッションにする

これで上記`LOG_LOC_AND_PREFIX`に指定したファイルにログが出力される。


## pmap と gdb による調査

プロセスのメモリマップを表示する `pmap` を採取してリークしている領域を特定し、その内容を `gdb` から確認する。

まず、`pmap` を採取して増加している領域を特定する。合わせメモリダンプを確認するために `gcore` により core を採取しておく。

プログラムを実行

    $ ./memory_leak_sample
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007b8e820
    Press enter key:

この時の pmap の結果は以下

    $ ps -ef | grep memory_leak_sample
    hashi    16639 29479  0 10:47 pts/10   00:00:00 ./memory_leak_sample
    $ pmap -x 16639
    16639:   ./memory_leak_sample
    Address           Kbytes     RSS   Dirty Mode   Mapping
    0000000000400000       4       4       0 r-x--  memory_leak_sample
    0000000000600000       4       4       4 rw---  memory_leak_sample
    0000000007b8e000     132       8       8 rw---    [ anon ]
    0000003e1c600000     112      96       0 r-x--  ld-2.5.so
    0000003e1c81b000       4       4       4 r----  ld-2.5.so
    0000003e1c81c000       4       4       4 rw---  ld-2.5.so
    0000003e1ca00000    1336     260       0 r-x--  libc-2.5.so
    0000003e1cb4e000    2044       0       0 -----  libc-2.5.so
    0000003e1cd4d000      16      12       8 r----  libc-2.5.so
    0000003e1cd51000       4       4       4 rw---  libc-2.5.so
    0000003e1cd52000      20      16      16 rw---    [ anon ]
    00002b2fac886000      12       8       8 rw---    [ anon ]
    00002b2fac89f000       8       8       8 rw---    [ anon ]
    00007fff1b3a6000      84      12      12 rw---    [ stack ]
    ffffffffff600000    8192       0       0 -----    [ anon ]
    ----------------  ------  ------  ------
    total kB           11976     440      76

合わせて core を採取しておく(上書きされないようにリネームしておく)

    $ gcore 16639
    0x0000003e1cac5ff0 in __read_nocancel () from /lib64/libc.so.6
    Saved corefile core.16639

    $ mv core.16639 core.16639.before

プログラムを進める

    $ ./memory_leak_sample
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007b8e820
    Press enter key:  <=== エンターキーを押下
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007b8f030
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007b8f840
    <中略>
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007bae410
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007baec20
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007baf430
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007bafc40
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007bb0450
    <中略>
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007bc0650
    freed memory: 0x0000000007b8e010, leaked memory: 0x0000000007bc0e60
    Press enter key:

この時の pmap と core を採取しておく

    $ pmap -x 16639
    16639:   ./memory_leak_sample
    Address           Kbytes     RSS   Dirty Mode   Mapping
    0000000000400000       4       4       0 r-x--  memory_leak_sample
    0000000000600000       4       4       4 rw---  memory_leak_sample
    0000000007b8e000     264     208     208 rw---    [ anon ]
    0000003e1c600000     112      96       0 r-x--  ld-2.5.so
    0000003e1c81b000       4       4       4 r----  ld-2.5.so
    0000003e1c81c000       4       4       4 rw---  ld-2.5.so
    0000003e1ca00000    1336     260       0 r-x--  libc-2.5.so
    0000003e1cb4e000    2044       0       0 -----  libc-2.5.so
    0000003e1cd4d000      16      12       8 r----  libc-2.5.so
    0000003e1cd51000       4       4       4 rw---  libc-2.5.so
    0000003e1cd52000      20      16      16 rw---    [ anon ]
    00002b2fac886000      12      12      12 rw---    [ anon ]
    00002b2fac89f000       8       8       8 rw---    [ anon ]
    00007fff1b3a6000      84      12      12 rw---    [ stack ]
    ffffffffff600000    8192       0       0 -----    [ anon ]
    ----------------  ------  ------  ------
    total kB           12108     644     280

    $ gcore 16639
    0x0000003e1cac5ff0 in __read_nocancel () from /lib64/libc.so.6
    Saved corefile core.16639

    $ mv core.16639 core.16639.after

1回目と2回目の pmap の結果を比較すると、以下の個所で仮想メモリ量が増加している(リークしている)ことが分かる。

    5c5
    < 0000000007b8e000     132       8       8 rw---    [ anon ]
    ---
    > 0000000007b8e000     264     208     208 rw---    [ anon ]

仮想メモリ量が 132Kbytes -> 264Kbytes (+132Kbytes)に増加している。
増加したアドレスのメモリダンプを確認して、どのように利用されているか確認してみる。

具体的にはメモリ増加後の core で 0x0000000007b8e000 + 132Kbytes のアドレス(0x7baf000)から
増加した 132Kbytes 分のメモリダンプを確認する。
ダンプを採るときに `x/16896xg 0x7baf000` としているのは、アドレス `0x7baf000` から、
8バイト(ジャイアント・ワード)単位で(`g`)、16896個分を、16進数で(`x`)出力するということ。
つまり、`16896 * 8 = 135168 = 132Kbytes` 分が出力される。
    
    $ gdb
    (gdb) set height 0
    (gdb) file ./memory_leak_sample
    (gdb) core-file core.16639.after
    (gdb) set logging file core.16639.after.gdb.log
    (gdb) set logging on
    (gdb) x/16896xg 0x7baf000
    0x7baf000:      0x0000000000000000      0x0000000000000000
    0x7baf010:      0x0000000000000000      0x0000000000000000
    <中略>
    0x7baf410:      0x0000000000000000      0x0000000000000000
    0x7baf420:      0x0000000000000000      0x0000000000000811
    0x7baf430:      0x6d2064656b61656c      0x00000079726f6d65
    0x7baf440:      0x0000000000000000      0x0000000000000000
    <中略>
    0x7bcffe0:      0x0000000000000000      0x0000000000000000
    0x7bcfff0:      0x0000000000000000      0x0000000000000000
    (gdb) quit

このメモリダンプが pmap 上増加した分。内容を確認すると以下の文字列が見える。

    $ grep -v "0x0000000000000000" core.16639.after.gdb.log
    0x7baf430:      0x6d2064656b61656c      0x00000079726f6d65
    0x7bafc40:      0x6d2064656b61656c      0x00000079726f6d65
    0x7bb0450:      0x6d2064656b61656c      0x00000079726f6d65
    0x7bb0c60:      0x6d2064656b61656c      0x00000079726f6d65
    <中略>
    0x7bbf630:      0x6d2064656b61656c      0x00000079726f6d65
    0x7bbfe40:      0x6d2064656b61656c      0x00000079726f6d65
    0x7bc0650:      0x6d2064656b61656c      0x00000079726f6d65
    0x7bc0e60:      0x6d2064656b61656c      0x00000079726f6d65

`0x6d2064656b61656c 0x00000079726f6d65` は ASCII で直すと "leaked memory"

    $ ruby -e 's="6d2064656b61656c"; s.unpack("a2" * (s.size / 2)) {|c| print c.hex.chr}; puts'
    m dekael
    $ ruby -e 's="00000079726f6d65"; s.unpack("a2" * (s.size / 2)) {|c| print c.hex.chr}; puts'
    yrome

よってプログラム中で "leaked memory" を入れている領域がリークしていると分かる。
こんなに明らかに分かるケースは少なく、実際のプログラムではポインタが見えていたり
して、さらに core を追わないとリーク原因箇所が追えないケースが多いと思うがメモリ
リーク原因究明のとっかかりにはなる。

### 追記

#### gdb によるメモリダンプについて

gdb による core のメモリダンプは `dump binary memory` によって行うことができる。
(こちらのほうが `x` でダンプするより速い。)

    (gdb) help dump binary memory
    Write contents of memory to a raw binary file.
    Arguments are FILE START STOP.  Writes the contents of memory
    within the range [START .. STOP) to the specifed FILE in binary format.

例えば上述したもの同じように core で 0x7baf000 から 132Kbytes 分(0x7bd0000 まで)メモリダンプする場合は
以下のように行う。

    $ gdb
    (gdb) core-file core.16639.after
    (gdb) dump binary memory core.16639.after.gdb.dump.log 0x7baf000 0x7bd0000
    (gdb) quit

これでファイル `core.16639.after.gdb.dump.log` にダンプされている。
中身はバイナリなので `od` や `hexdump` で内容を確認する。

    $ od -v -t x8z -A x ./core.16639.after.gdb.dump.log
    000000 0000000000000000 0000000000000000  >................<
    000010 0000000000000000 0000000000000000  >................<
    <中略>
    000410 0000000000000000 0000000000000000  >................<
    000420 0000000000000000 0000000000000811  >................<
    000430 6d2064656b61656c 00000079726f6d65  >leaked memory...<
    000440 0000000000000000 0000000000000000  >................<
    <中略>
    020fe0 0000000000000000 0000000000000000  >................<
    020ff0 0000000000000000 0000000000000000  >................<
    021000

同じ内容が連続する場合に省略する場合は `-v` を付けなければよい。

    $ od -t x8z -A x ./core.16639.after.gdb.dump.log
    000000 0000000000000000 0000000000000000  >................<
    *
    000420 0000000000000000 0000000000000811  >................<
    000430 6d2064656b61656c 00000079726f6d65  >leaked memory...<
    000440 0000000000000000 0000000000000000  >................<
    *
    000c30 0000000000000000 0000000000000811  >................<
    000c40 6d2064656b61656c 00000079726f6d65  >leaked memory...<
    000c50 0000000000000000 0000000000000000  >................<
    *
    <中略>
    011e50 0000000000000000 0000000000000811  >................<
    011e60 6d2064656b61656c 00000079726f6d65  >leaked memory...<
    011e70 0000000000000000 0000000000000000  >................<
    *
    012660 0000000000000000 000000000000e9a1  >................<
    012670 0000000000000000 0000000000000000  >................<
    *
    021000

#### core から pmap と同じような情報を得る方法

gdb の `info target` とか `info files` で `pmap` と同じような情報を確認できる。
(`pmap` を採っていなくても core から同じような情報が何とか見れる。)

    $ gdb
    (gdb) file ./memory_leak_sample
    (gdb) core-file core.16639.after
    (gdb) info target
    Symbols from "/home/hashi/tmp/memory_leak_sample".
    Local core dump file:
            `/home/hashi/tmp/core.16639.after', file type elf64-x86-64.
            0x0000000000400000 - 0x0000000000400000 is load1
            0x0000000000600000 - 0x0000000000601000 is load2
            0x0000000007b8e000 - 0x0000000007bd0000 is load3
            0x0000003e1c600000 - 0x0000003e1c600000 is load4
            0x0000003e1c81b000 - 0x0000003e1c81b000 is load5
            0x0000003e1c81c000 - 0x0000003e1c81d000 is load6
            0x0000003e1ca00000 - 0x0000003e1ca00000 is load7
            0x0000003e1cd4d000 - 0x0000003e1cd4d000 is load8
            0x0000003e1cd51000 - 0x0000003e1cd52000 is load9
            0x0000003e1cd52000 - 0x0000003e1cd57000 is load10
            0x00002b2fac886000 - 0x00002b2fac889000 is load11
            0x00002b2fac89f000 - 0x00002b2fac8a1000 is load12
            0x00007fff1b3a6000 - 0x00007fff1b3bb000 is load13
    Local exec file:
            `/home/hashi/tmp/memory_leak_sample', file type elf64-x86-64.
            Entry point: 0x4005b0
            0x0000000000400200 - 0x000000000040021c is .interp
            0x000000000040021c - 0x000000000040023c is .note.ABI-tag
            0x0000000000400240 - 0x0000000000400264 is .gnu.hash
            0x0000000000400268 - 0x0000000000400370 is .dynsym
            0x0000000000400370 - 0x00000000004003d8 is .dynstr
            0x00000000004003d8 - 0x00000000004003ee is .gnu.version
            0x00000000004003f0 - 0x0000000000400410 is .gnu.version_r
            0x0000000000400410 - 0x0000000000400440 is .rela.dyn
            0x0000000000400440 - 0x0000000000400500 is .rela.plt
            0x0000000000400500 - 0x0000000000400518 is .init
            0x0000000000400518 - 0x00000000004005a8 is .plt
            0x00000000004005b0 - 0x00000000004008d8 is .text
            0x00000000004008d8 - 0x00000000004008e6 is .fini
            0x00000000004008e8 - 0x0000000000400957 is .rodata
            0x0000000000400958 - 0x0000000000400994 is .eh_frame_hdr
            0x0000000000400998 - 0x0000000000400a8c is .eh_frame
            0x0000000000600a90 - 0x0000000000600aa0 is .ctors
            0x0000000000600aa0 - 0x0000000000600ab0 is .dtors
            0x0000000000600ab0 - 0x0000000000600ab8 is .jcr
            0x0000000000600ab8 - 0x0000000000600c48 is .dynamic
            0x0000000000600c48 - 0x0000000000600c50 is .got
            0x0000000000600c50 - 0x0000000000600ca8 is .got.plt
            0x0000000000600ca8 - 0x0000000000600cac is .data
            0x0000000000600cb0 - 0x0000000000600cc8 is .bss

アドレスぐらいしか分からないが `Local core dump file:` の項が `pmap` の出力と対応している。
(アドレスを引けばサイズが分かる。)

    Local core dump file:
            `/home/hashi/tmp/core.16639.after', file type elf64-x86-64.
            0x0000000000400000 - 0x0000000000400000 is load1  <=== TEXT 領域など
            0x0000000000600000 - 0x0000000000601000 is load2  <=== BSS 領域など
            0x0000000007b8e000 - 0x0000000007bd0000 is load3  <=== size: 264Kbytes (= 0x0000000007bd0000 - 0x0000000007b8e000)
            0x0000003e1c600000 - 0x0000003e1c600000 is load4  <=== size:   0?
            0x0000003e1c81b000 - 0x0000003e1c81b000 is load5  <=== size:   0?
            0x0000003e1c81c000 - 0x0000003e1c81d000 is load6  <=== size:   4Kbytes
            0x0000003e1ca00000 - 0x0000003e1ca00000 is load7  <=== size:   0?
            0x0000003e1cd4d000 - 0x0000003e1cd4d000 is load8  <=== size:   0?
            0x0000003e1cd51000 - 0x0000003e1cd52000 is load9  <=== size:   4Kbytes
            0x0000003e1cd52000 - 0x0000003e1cd57000 is load10 <=== size:  20Kbytes
            0x00002b2fac886000 - 0x00002b2fac889000 is load11 <=== size:  12Kbytes
            0x00002b2fac89f000 - 0x00002b2fac8a1000 is load12 <=== size:   8Kbytes
            0x00007fff1b3a6000 - 0x00007fff1b3bb000 is load13 <=== size:  84Kbytes

ちなみに `maintenace info sections` でも同じような情報が採れる。

    (gdb) maintenance info sections
    Exec file:
        `/home/hashi/tmp/memory_leak_sample', file type elf64-x86-64.
        0x00400200->0x0040021c at 0x00000200: .interp ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x0040021c->0x0040023c at 0x0000021c: .note.ABI-tag ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400240->0x00400264 at 0x00000240: .gnu.hash ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400268->0x00400370 at 0x00000268: .dynsym ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400370->0x004003d8 at 0x00000370: .dynstr ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x004003d8->0x004003ee at 0x000003d8: .gnu.version ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x004003f0->0x00400410 at 0x000003f0: .gnu.version_r ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400410->0x00400440 at 0x00000410: .rela.dyn ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400440->0x00400500 at 0x00000440: .rela.plt ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400500->0x00400518 at 0x00000500: .init ALLOC LOAD READONLY CODE HAS_CONTENTS
        0x00400518->0x004005a8 at 0x00000518: .plt ALLOC LOAD READONLY CODE HAS_CONTENTS
        0x004005b0->0x004008d8 at 0x000005b0: .text ALLOC LOAD READONLY CODE HAS_CONTENTS
        0x004008d8->0x004008e6 at 0x000008d8: .fini ALLOC LOAD READONLY CODE HAS_CONTENTS
        0x004008e8->0x00400957 at 0x000008e8: .rodata ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400958->0x00400994 at 0x00000958: .eh_frame_hdr ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00400998->0x00400a8c at 0x00000998: .eh_frame ALLOC LOAD READONLY DATA HAS_CONTENTS
        0x00600a90->0x00600aa0 at 0x00000a90: .ctors ALLOC LOAD DATA HAS_CONTENTS
        0x00600aa0->0x00600ab0 at 0x00000aa0: .dtors ALLOC LOAD DATA HAS_CONTENTS
        0x00600ab0->0x00600ab8 at 0x00000ab0: .jcr ALLOC LOAD DATA HAS_CONTENTS
        0x00600ab8->0x00600c48 at 0x00000ab8: .dynamic ALLOC LOAD DATA HAS_CONTENTS
        0x00600c48->0x00600c50 at 0x00000c48: .got ALLOC LOAD DATA HAS_CONTENTS
        0x00600c50->0x00600ca8 at 0x00000c50: .got.plt ALLOC LOAD DATA HAS_CONTENTS
        0x00600ca8->0x00600cac at 0x00000ca8: .data ALLOC LOAD DATA HAS_CONTENTS
        0x00600cb0->0x00600cc8 at 0x00000cac: .bss ALLOC
        0x00000000->0x00000114 at 0x00000cac: .comment READONLY HAS_CONTENTS
    Core file:
        `/home/hashi/tmp/core.16639.after', file type elf64-x86-64.
        0x00000000->0x00000528 at 0x00000350: note0 READONLY HAS_CONTENTS
        0x00000000->0x000000d8 at 0x00000470: .reg/16639 HAS_CONTENTS
        0x00000000->0x000000d8 at 0x00000470: .reg HAS_CONTENTS
        0x00000000->0x00000200 at 0x00000564: .reg2/16639 HAS_CONTENTS
        0x00000000->0x00000200 at 0x00000564: .reg2 HAS_CONTENTS
        0x00000000->0x00000100 at 0x00000778: .auxv HAS_CONTENTS
        0x00400000->0x00400000 at 0x00000878: load1 ALLOC READONLY CODE
        0x00600000->0x00601000 at 0x00000878: load2 ALLOC LOAD HAS_CONTENTS
        0x07b8e000->0x07bd0000 at 0x00001878: load3 ALLOC LOAD HAS_CONTENTS
        0x3e1c600000->0x3e1c600000 at 0x00043878: load4 ALLOC READONLY CODE
        0x3e1c81b000->0x3e1c81b000 at 0x00043878: load5 ALLOC READONLY
        0x3e1c81c000->0x3e1c81d000 at 0x00043878: load6 ALLOC LOAD HAS_CONTENTS
        0x3e1ca00000->0x3e1ca00000 at 0x00044878: load7 ALLOC READONLY CODE
        0x3e1cd4d000->0x3e1cd4d000 at 0x00044878: load8 ALLOC READONLY
        0x3e1cd51000->0x3e1cd52000 at 0x00044878: load9 ALLOC LOAD HAS_CONTENTS
        0x3e1cd52000->0x3e1cd57000 at 0x00045878: load10 ALLOC LOAD HAS_CONTENTS
        0x2b2fac886000->0x2b2fac889000 at 0x0004a878: load11 ALLOC LOAD HAS_CONTENTS
        0x2b2fac89f000->0x2b2fac8a1000 at 0x0004d878: load12 ALLOC LOAD HAS_CONTENTS
        0x7fff1b3a6000->0x7fff1b3bb000 at 0x0004f878: load13 ALLOC LOAD HAS_CONTENTS

なお、`info proc mappings` というコマンドもあるが、これはプロセス起動中に gdb でアタッチしたときに
使用して `/proc` から情報を採るものなので、core からの調査には使えないようだ。
(core に対して実行すると PID:1 の `/sbin/init` の `/proc/1/maps` 情報が出てきてしまい役に立たない。)

### core から文字列等を検索する

リークしている領域の特徴に当たりがついていれば gdb の `find` で見つけるという方法もある。

    (gdb) help find
    Search memory for a sequence of bytes.
    Usage:
    find [/size-char] [/max-count] start-address, end-address, expr1 [, expr2 ...]
    find [/size-char] [/max-count] start-address, +length, expr1 [, expr2 ...]
    size-char is one of b,h,w,g for 8,16,32,64 bit values respectively,
    and if not specified the size is taken from the type of the expression
    in the current language.
    Note that this means for example that in the case of C-like languages
    a search for an untyped 0x42 will search for "(int) 0x42"
    which is typically four bytes.

    The address of the last match is stored as the value of "$_".
    Convenience variable "$numfound" is set to the number of matches.

例えば上述の例で 0x7baf000 から 132Kbytes 分の間に "leaked memory" という文字列を見つける。

    $ gdb
    (gdb) core-file core.16639.after
    (gdb) find 0x7baf000, +(132 * 1024), "leaked memory"
    0x7baf430
    0x7bafc40
    0x7bb0450
    0x7bb0c60
    0x7bb1470
    0x7bb1c80
    0x7bb2490
    0x7bb2ca0
    0x7bb34b0
    0x7bb3cc0
    0x7bb44d0
    0x7bb4ce0
    0x7bb54f0
    0x7bb5d00
    0x7bb6510
    0x7bb6d20
    0x7bb7530
    0x7bb7d40
    0x7bb8550
    0x7bb8d60
    0x7bb9570
    0x7bb9d80
    0x7bba590
    0x7bbada0
    0x7bbb5b0
    0x7bbbdc0
    0x7bbc5d0
    0x7bbcde0
    0x7bbd5f0
    0x7bbde00
    0x7bbe610
    0x7bbee20
    0x7bbf630
    0x7bbfe40
    0x7bc0650
    0x7bc0e60
    36 patterns found.

36 個所で見つかった。
なお、`find` の引数で文字列を渡して探すときは文字列は NULL 終端を含むことに注意。
例えば、上記の例で部分文字列で探そうと "leaked" としても引っかからない。

    (gdb)  find 0x7baf000, +(132 * 1024), "leaked"
    Pattern not found.

部分文字列で探す場合は文字列を16進数にするしかなさそう。
(リトルエンディアンで "leaked" は `0x64656b61656c` になる。)

    (gdb)  find /b  0x7baf000, +(132 * 1024), 0x64656b61656c
    0x7baf430
    0x7bafc40
    <中略>
    0x7bc0650
    0x7bc0e60
    36 patterns found.
