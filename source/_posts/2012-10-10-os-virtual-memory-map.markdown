---
layout: post
title: "[OS] 仮想メモリ空間のメモリマップを調べる"
date: 2012-10-10 23:20
comments: true
categories: [OS, Linux]
---
## 目的

仮想メモリ空間のアドレス等のメモリマップを調べる。

なお、ちゃんと調べたわけではないので誤りがあるかもしれない。

## 環境

* OS: Oracle Enterprise Linux 5.8
* Kernel: 2.6.32-300.10.1.el5uek x86_64

## 仮想メモリ空間のメモリマップ

Unix/Linux における仮想メモリ空間のメモリマップは一般には以下のようになっている。

    +------------------------------+  0x0000000000000000
    :                              :
    +------------------------------+
    |                              |
    |  text                        |  機械命令
    |                              |
    +------------------------------+
    |                              |
    |  data                        |  初期化された static 変数
    |                              |
    +------------------------------+
    |                              |
    |  BSS                         |  初期化されていない static 変数
    |                              |
    +------------------------------+
    |                              |
    |  heap                        |  malloc() で動的に確保される領域(上位アドレスに伸びる)
    |                              |
    +------------------------------+
    |             ||||             |
    |             VVVV             |
    :                              :
    :                              :
    |                              |
    +------------------------------+
    |                              |
    |  shared memory               |  共有メモリ領域
    |                              |
    +------------------------------+
    |                              |
    :                              :
    :                              :
    |             ^^^^             |
    |             ||||             |
    +------------------------------+
    |                              |
    |  stack                       |  関数呼び出しやローカル変数等で使用されるスタック領域(下位アドレスに伸びる)
    |                              |
    +------------------------------+
    |                              |
    |  arguments / environments    |  引数と環境変数
    |                              |
    +------------------------------+
    :                              :
    :                              :
    +------------------------------+  0xffffffffffffffff = 2^64 (64bit の場合)

## 実例

以下のプログラムでメモリマップを確認する。

{% codeblock lang:c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/shm.h>

#define STRSIZE 64

void hello(char *name);
void hello_local_world();

char not_initialized_global_world[STRSIZE];
char initialized_global_world[STRSIZE] = "initialized global world";


int main(int argc, char *argv[])
{
  static char not_initialized_static_world[STRSIZE];
  static char initialized_static_world[STRSIZE] = "initialized static world";
  char *malloc_world;
  int shmid;
  char *shared_memory_world;
  char *big_malloc_world;
  char *medium_malloc_world_1;
  char *medium_malloc_world_2;

  strcpy(not_initialized_global_world, "not initialized global world");

  strcpy(not_initialized_static_world, "not initialized static world");

  malloc_world = (char *) malloc(sizeof(char) * STRSIZE); 
  strcpy(malloc_world, "malloc world");

  shmid = shmget(IPC_PRIVATE, sizeof(char) * STRSIZE, 0666 | IPC_CREAT);
  shared_memory_world = (char *) shmat(shmid, 0, 0);
  strcpy(shared_memory_world, "shared memory world");

  big_malloc_world = (char *) malloc(256 * 1024); 
  strcpy(big_malloc_world, "big malloc world");

  medium_malloc_world_1 = (char *) malloc(131 * 1024 + 905); 
  strcpy(medium_malloc_world_1, "medium malloc world 1");

  medium_malloc_world_2 = (char *) malloc(131 * 1024 + 904); 
  strcpy(medium_malloc_world_2, "medium malloc world 2");

  hello(argv[1]);
  hello(not_initialized_global_world);
  hello(initialized_global_world);
  hello(not_initialized_static_world);
  hello(initialized_static_world);
  hello(malloc_world);
  hello(shared_memory_world);
  hello(big_malloc_world);
  hello(medium_malloc_world_1);
  hello(medium_malloc_world_2);
  hello_local_world();

  getchar();

  free(malloc_world);
  shmdt(shared_memory_world);
  shmctl(shmid, IPC_RMID, 0); 
  free(big_malloc_world);
  free(medium_malloc_world_1);
  free(medium_malloc_world_2);

  return 0;
}

void hello(char *name)
{
  printf("hello, %s: 0x%016lx\n", name, name);
}

void hello_local_world()
{
  char local_world[STRSIZE];

  strcpy(local_world, "local world");
  hello(local_world);
}
{% endcodeblock %}

実行結果は以下。

    $ gcc -o hello hello.c
    $ ./hello ARGworld
    hello, ARGworld: 0x00007ffff7269cb5
    hello, not initialized global world: 0x0000000000600ec0
    hello, initialized global world: 0x0000000000600de0
    hello, not initialized static world: 0x0000000000600e80
    hello, initialized static world: 0x0000000000600e20
    hello, malloc world: 0x0000000001f61010
    hello, shared memory world: 0x00007f2eeea0e000
    hello, big malloc world: 0x00007f2eee9b0010
    hello, medium malloc world 1: 0x00007f2eee98f010
    hello, medium malloc world 2: 0x0000000001f61060
    hello, local world: 0x00007ffff7269680

メモリマップは `pmap` や `cat /proc/<PID>/maps` や `cat /proc/<PID>/smaps` で確認できる。

    $ pmap -x 19671
    19671:   ./hello ARGworld
    Address           Kbytes     RSS   Dirty Mode   Mapping
    0000000000400000       4       4       0 r-x--  hello
    0000000000600000       4       4       4 rw---  hello
    0000000001f61000     132       8       8 rw---    [ anon ]
    00000037be000000     112      96       0 r-x--  ld-2.5.so
    00000037be21c000       4       4       4 r----  ld-2.5.so
    00000037be21d000       4       4       4 rw---  ld-2.5.so
    00000037be400000    1340     248       0 r-x--  libc-2.5.so
    00000037be54f000    2044       0       0 -----  libc-2.5.so
    00000037be74e000      16      12       8 r----  libc-2.5.so
    00000037be752000       4       4       4 rw---  libc-2.5.so
    00000037be753000      20      16      16 rw---    [ anon ]
    00007f2eee98d000     408      20      20 rw---    [ anon ]
    00007f2eeea0e000       4       4       4 rw-s-    [ shmid=0x578006 ]
    00007f2eeea0f000       8       8       8 rw---    [ anon ]
    00007ffff7255000      84       8       8 rw---    [ stack ]
    00007ffff730e000       4       4       0 r-x--    [ anon ]
    ffffffffff600000       4       0       0 r-x--    [ anon ]
    ----------------  ------  ------  ------
    total kB            4196     444      88

    $ cat /proc/19671/maps 
    00400000-00401000 r-xp 00000000 fd:00 5101878                            /tmp/hello
    00600000-00601000 rw-p 00000000 fd:00 5101878                            /tmp/hello
    01f61000-01f82000 rw-p 00000000 00:00 0                                  [heap]
    37be000000-37be01c000 r-xp 00000000 fd:00 1871272                        /lib64/ld-2.5.so
    37be21c000-37be21d000 r--p 0001c000 fd:00 1871272                        /lib64/ld-2.5.so
    37be21d000-37be21e000 rw-p 0001d000 fd:00 1871272                        /lib64/ld-2.5.so
    37be400000-37be54f000 r-xp 00000000 fd:00 1871273                        /lib64/libc-2.5.so
    37be54f000-37be74e000 ---p 0014f000 fd:00 1871273                        /lib64/libc-2.5.so
    37be74e000-37be752000 r--p 0014e000 fd:00 1871273                        /lib64/libc-2.5.so
    37be752000-37be753000 rw-p 00152000 fd:00 1871273                        /lib64/libc-2.5.so
    37be753000-37be758000 rw-p 00000000 00:00 0 
    7f2eee98d000-7f2eee9f3000 rw-p 00000000 00:00 0 
    7f2eeea0e000-7f2eeea0f000 rw-s 00000000 00:04 5734406                    /SYSV00000000 (deleted)
    7f2eeea0f000-7f2eeea11000 rw-p 00000000 00:00 0 
    7ffff7255000-7ffff726a000 rw-p 00000000 00:00 0                          [stack]
    7ffff730e000-7ffff730f000 r-xp 00000000 00:00 0                          [vdso]
    ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

`size` コマンドで text や BSS のサイズを確認できる。

    $ size hello
       text    data     bss     dec     hex filename
       2414     704     160    3278     cce hello

    $ size --format=SysV -x hello
    hello  :
    section           size       addr
    .interp           0x1c   0x400200
    .note.ABI-tag     0x20   0x40021c
    .gnu.hash         0x1c   0x400240
    .dynsym          0x108   0x400260
    .dynstr           0x6d   0x400368
    .gnu.version      0x16   0x4003d6
    .gnu.version_r    0x20   0x4003f0
    .rela.dyn         0x18   0x400410
    .rela.plt         0xd8   0x400428
    .init             0x18   0x400500
    .plt              0xa0   0x400518
    .text            0x488   0x4005c0
    .fini              0xe   0x400a48
    .rodata           0x25   0x400a58
    .eh_frame_hdr     0x34   0x400a80
    .eh_frame         0xd4   0x400ab8
    .ctors            0x10   0x600b90
    .dtors            0x10   0x600ba0
    .jcr               0x8   0x600bb0
    .dynamic         0x190   0x600bb8
    .got               0x8   0x600d48
    .got.plt          0x60   0x600d50
    .data             0xa0   0x600dc0
    .bss              0xa0   0x600e60
    .comment         0x114        0x0
    Total            0xde2

これらから次のようなメモリマップとなっていると考えられる。

    +------------------------------+  0x0000000000000000
    :                              :
    +------------------------------+  0x0000000000400000
    |text                          |  機械命令
    |                              |
    +------------------------------+  0x0000000000401000
    :                              :
    +------------------------------+  0x0000000000600000
    |                              |  0x0000000000600dc0
    |data                          |  初期化された static 変数
    |  initialized global var      |  0x0000000000600de0
    |  initialized static var      |  0x0000000000600e20
    |                              |
    +------------------------------+  0x0000000000600e60
    |BSS                           |  初期化されていない static 変数
    |  not initialized static var  |  0x0000000000600e80
    |  not initialized global var  |  0x0000000000600ec0
    |                              |
    +------------------------------+  0x0000000000601000
    :                              :
    +------------------------------+  0x0000000001f61000
    |heap                          |  malloc() で動的に確保される領域(上位アドレスに伸びる)
    |  malloc var                  |  0x0000000001f61010
    |  malloc var                  |  0x0000000001f61060
    |                              |
    +------------------------------+  0x0000000001f82000
    |             ||||             |
    |             VVVV             |
    :                              :
    :                              :
    |                              |
    +------------------------------+  0x00000037be000000
    |                              |
    |  共有ライブラリ              |
    |  (ld-2.5.so, libc-2.5.so)    |
    +------------------------------+  0x00000037be753000
    | ???                          |
    +------------------------------+  0x00000037be758000
    |                              |
    :                              :
    :                              :
    |             ^^^^ ??          |
    |             |||| ??          |
    +------------------------------+  0x00007f2eee98d000
    |heap??                        |
    |  malloc var                  |  0x00007f2eee98f010
    |  big malloc var              |  0x00007f2eee9b0010
    |                              |
    +------------------------------+  0x00007f2eee9f3000
    :                              :
    +------------------------------+  0x00007f2eeea0e000
    |shared memory                 |  共有メモリ領域
    |  shared memory var           |  0x00007f2eeea0e000
    |                              |
    +------------------------------+  0x00007f2eeea0f000
    |???                           |
    +------------------------------+  0x00007f2eeea11000
    |                              |
    :                              :
    :                              :
    |             ^^^^             |
    |             ||||             |
    +------------------------------+  0x00007ffff7255000
    |stack                         |  関数呼び出しやローカル変数等で使用されるスタック領域(下位アドレスに伸びる)
    |  local var                   |  0x00007ffff7269680
    |  arguments[1]                |  0x00007ffff7269cb5
    |                              |
    +------------------------------+  0x00007ffff726a000
    :                              :
    +------------------------------+  0x00007ffff730e000
    |???                           |
    +------------------------------+  0x00007ffff730f000
    |                              |
    :                              :
    :                              :
    |                              |
    +------------------------------+  0xffffffffff600000
    |arguments / environments??    |
    |                              |
    +------------------------------+  0xffffffffff601000
    :                              :
    :                              :
    +------------------------------+  0xffffffffffffffff = 2^64 (64bit の場合)

heap としては 0x0000000001f61000 〜 0x0000000001f82000 の 132Kbytes が割り当てられているようだが、
約132Kbytes より大きく malloc() で動的にメモリを割り当てるとアドレスが飛んで 0x00007f2eee98d000 付近に
メモリが割り当てられ、しかも下位にメモリが伸びているようだ。


## 参考

- [メモリ管理、アドレス空間、ページテーブル](http://www.coins.tsukuba.ac.jp/~yas/coins/os2-2011/2012-01-24/index.html)
- [おじさんＯＳ講座（１）：pmapを使ってみよう](http://blog.gakitama.com/?e=490)
- [mallocライブラリのメモリ管理構造](http://www.valinux.co.jp/contents/tech/techlib/eos/malloc/malloc_001.html)
