---
layout: post
title: "[Debug] ptrace によるデバック"
date: 2013-08-27 20:21
comments: true
categories: [OS, Linux, Debug]
---
## 目的

システムコール ptrace(2) を使用してデバックする。
ptrace を使用したデバッカ・プログラムを作成することで、GDB などの汎用デバッカより細かい制御を行うことも可能になる。

## 環境

* OS: CentOS 5.5
* Kernel: 2.6.18-194.el5 x86_64
* CPU: Intel(R) Xeon(R) CPU E5345 (Virtual Machine on VMware)

## 前提知識

### システムコール ptrace(2)

別のプロセスにアタッチしたり、別のプロセスのメモリやレジスタを参照、書込み等を行うことができるシステムコール。
デバッカを実装するために使用される。

{% codeblock lang:c %}
#include <sys/ptrace.h>

long ptrace(enum __ptrace_request request, pid_t pid,
            void *addr, void *data);
{% endcodeblock %}

`request` にアタッチしたプロセスに何を行うかを指定する。
例えば、以下が指定できる。(参考: `man ptrace`)

- PTRACE_ATTACH

  `pid` で指定したプロセスにアタッチする。アタッチしたプロセスは子プロセスとしてトレースできるようにする。(引数 `addr` と `data` は無視される。)

- PTRACE_PEEKTEXT, PTRACE_PEEKDATA

  メモリの `addr` の位置を参照する。(引数 `data` は無視される。)

- PTRACE_PEEKUSR

  USER 領域のオフセット `addr` の位置を参照する。(引数 `data` は無視される。)

- PTRACE_POKETEXT, PTRACE_POKEDATA

  `data` をメモリの `addr` の位置に書込む。

- PTRACE_POKEUSR

  `data` を USER 領域のオフセット `addr` の位置に書き込む。

- PTRACE_GETREGS

  レジスタの値を `data` にコピーする。`data` は user_regs_struct 構造体(<linux/user.h>)。(引数 `addr` は無視される。)

- PTRACE_SETREGS

  `data` をレジスタにコピーする。`data` は user_regs_struct 構造体(<linux/user.h>)。(引数 `addr` は無視される。)

### インストラクション INT 3

`INT X`(X は 0～255 の数値) は x86, x86-64 プロセッサのインストラクションで、ソフトウェア割り込みを発生させる。

特に `INT 3`(オペコード: `0xCC`) により SIGTRAP シグナルが発生し、処理が中断される。これは、デバッカがブレイクポイントを設定するために使用される。

### レジスタ IP

x86-64 プロセッサのレジスタ RIP (インストラクションポインタ)には次に実行される命令のアドレスが格納されている。(プログラムカウンタとも呼ばれる。)

## ptrace によるブレイクポイントの設定

ptrace では、アタッチしたプロセスのメモリやレジスタを参照したり、書込むといった原始的なことしかできない。

したがって、ptrace によってブレイクポイントを設定してデバックするには、以下のような手順が必要。

1. PTRACE_ATTACH でターゲットのプロセスにアタッチ。

2. PTRACE_PEEKTEXT でブレイクポイントを設定したいアドレスの命令を一時的に保存しておく。

3. PTRACE_POKETEXT で 2. のアドレスの命令を INT 3 (0xCC)に書換える。

4. PTRACE_CONT でプロセスを再開させる。

5. waitpid(2) でブレイクするのを待つ。

6. プロセスが INT 3 で書換えたアドレスでブレイクする。

7. 次に実行する命令のアドレスを示すレジスタ IP は INT 3 で書換えたアドレス(ブレイクポイントを設定したアドレス)の次の位置(+1バイトの位置)を指しているので、PTRACE_GETREGS/PTRACE_SETREGS で IP をブレイクポイントを設定したアドレスに書換えて戻す。

8. PTRACE_POKETEXT で 3. で書換えた命令を 2. で保存していたオリジナルに戻す。(ブレイクポイントを設定したアドレスのメモリを元の命令に戻す。)

9. これでブレイクしたい位置で中断されているので、レジスタを見たり、メモリを参照したりして任意のデバックを行う。

10. 再度ブレイクさせたかったら 2. に戻る。
    ただし、同じ命令のアドレスでブレイクさせたかったら、PTRACE_SINGLESTEP で 1 命令だけ進めてから、2. に戻る。(そうでないと現在の IP 位置を INT 3 で書換えてしまい、すぐにブレイクして処理が進まなくなってしまう。)

11. デバックが終わったら、PTRACE_DETACH でプロセスからデタッチする。

## 具体例

上述のような方法でブレイクポイントを設定してデバックするデバッカ・プログラムを作成する。

### デバッカ・プログラム

引数としてアタッチしたいプロセスの PID と、ブレイクするアドレスを指定して、ブレイクポイントでレジスタ RDI の値を表示するデバッカ・プログラム。

ブレイクするアドレスとして関数の先頭アドレスを指定すれば、レジスタ RDI には第 1 引数が入っているはずなので、このプログラムを使って第 1 引数を表示するデバックができる。

{% codeblock lang:c %}
#include <stdio.h>      /* printf */
#include <string.h>     /* memset */
#include <sys/ptrace.h> /* ptrace */
#include <sys/reg.h>    /* RIP */
#include <sys/types.h>  /* waitpid, stat */
#include <sys/wait.h>   /* waitpid */
#include <stdlib.h>     /* atoi, strtol, exit */
#include <linux/user.h> /* user_regs_struct */
#include <sys/stat.h>   /* stat */
#include <unistd.h>     /* stat */

void my_command(pid_t pid)
{
  struct user_regs_struct regs;

  memset(&regs, 0, sizeof(regs));
  
  ptrace(PTRACE_GETREGS, pid, 0, &regs);
  printf("arg 1: %ld\n", regs.rdi);
}

void p_attach(pid_t pid)
{
  int status;

  ptrace(PTRACE_ATTACH, pid, NULL, NULL);
  waitpid(pid, &status, 0);
}

long p_break(pid_t pid, void *addr)
{
  long original_text;

  original_text = ptrace(PTRACE_PEEKTEXT, pid, addr, NULL);
  ptrace(PTRACE_POKETEXT, pid, addr, ((original_text & 0xFFFFFFFFFFFFFF00) | 0xCC));
  printf("Breakpoint at %p.\n", addr);

  return original_text;
}

void p_delete(pid_t pid, void *addr, long original_text)
{
  ptrace(PTRACE_POKEUSER, pid, sizeof(long) * RIP, addr);
  ptrace(PTRACE_POKETEXT, pid, addr, original_text);
}

void p_continue(pid_t pid)
{
  int status;

  ptrace(PTRACE_CONT, pid, NULL, NULL);
  printf("Continuing.\n");

  waitpid(pid, &status, 0);

  if (WIFEXITED(status))
  {
    printf("Program exited normally.\n");
    exit(0);
  }

  if (WIFSTOPPED(status))
    printf("Breakpoint.\n");
  else
    exit(1);
}

void p_stepi(pid_t pid)
{
  int status;

  ptrace(PTRACE_SINGLESTEP, pid, NULL, NULL);
  waitpid(pid, &status, 0);
}

void p_quit(pid_t pid)
{
  ptrace(PTRACE_DETACH, pid, NULL, NULL);
}

int main(int argc, char *argv[])
{
  pid_t pid;
  void *addr;
  long original_text;
  struct stat buf;

  if (argc < 2)
  {
    printf("Usage: ptrace_demo <pid> <addr>\n");
    printf("Example: ptrace_demo 1234 0xabcdef0123456789\n");
    exit(1);
  }

  pid = atoi(argv[1]);
  addr = (void *)strtol(argv[2], NULL, 0);

  p_attach(pid);
  original_text = p_break(pid, addr);
  while (1)
  {
    p_continue(pid);
    p_delete(pid, addr, original_text);

    if (stat("./quit", &buf) == 0)
      break;
        
    /* do stuff */
    my_command(pid);

    p_stepi(pid);
    original_text = p_break(pid, addr);
  }

  p_quit(pid);

  return 0;
}
{% endcodeblock %}

なお、レジスタの書換えは以下のように PTRACE_GETREGS/PTRACE_SETREGS でなくて、PTRACE_POKEUSER でも行けた。(43行目)

{% codeblock lang:c %}
  ptrace(PTRACE_POKEUSER, pid, sizeof(long) * RIP, addr);
{% endcodeblock %}

もちろん、PTRACE_GETREGS/PTRACE_SETREGS を使って以下のようにしてもよい。

{% codeblock lang:c %}
  struct user_regs_struct regs;

  ptrace(PTRACE_GETREGS, pid, 0, &regs);
  regs.rip = (unsigned long) addr;
  ptrace(PTRACE_SETREGS, pid, 0, &regs);
{% endcodeblock %}

あと、デバックを止めるときはカレントディレクトリに `quit` というファイルを置くことにした。(`$ touch ./quit` とかで作ればよい。)

コンパイルしておく。

    $ gcc -o ptrace_demo ptrace_demo.c

### デバックするプログラム

デバック対象として以下の簡単なプログラムを用意。1秒毎に func 関数が呼ばれている。

{% codeblock lang:c %}
#include <stdio.h>
#include <unistd.h>

void func(int x);

void func(int x)
{
  printf("hello, world: %d\n", x);
}

int main(int argc, char *argv[])
{
  int i = 0;
  
  for (i = 1; i <= 100; i++)
  {
    sleep(1);
    func(i);
  }
  return 0;
}
{% endcodeblock %}

コンパイルしておく

    $ gcc -o sample sample.c

## デバック例

上記の func 関数が呼ばれる度に第 1 引数(レジスタ RDI) の値を出力するデバックを行う。

まず、func のアドレスを確認。

    $ nm sample | grep func
    00000000004004d8 T func

func のアドレスは 0x00000000004004d8 。

プログラムを実行。

    $ ./sample
    hello, world: 1
    hello, world: 2
    hello, world: 3
    hello, world: 4
    hello, world: 5
    hello, world: 6
    ...<略>

PID, アドレスを指定してデバッカ・プログラムを実行してみる。

    $ ps -ef | grep sample
    hashi    21250 32254  0 17:33 pts/2    00:00:00 ./sample

    $ ./ptrace_demo 21250 0x00000000004004d8
    Breakpoint at 0x4004d8.
    Continuing.
    Breakpoint.
    arg 1: 27
    Breakpoint at 0x4004d8.
    Continuing.
    Breakpoint.
    arg 1: 28
    Breakpoint at 0x4004d8.
    Continuing.
    Breakpoint.
    arg 1: 29
    Breakpoint at 0x4004d8.
    Continuing.
    Breakpoint.
    ...<略>

func でブレイクして、第 1 引数(レジスタ RDI) の値が出力できている。

止めるときは同じディレクトリに quit という名前のファイルを作れば、デタッチしてデバック終了する。

    $ touch quit

### GDB で実行した場合

上述の方法で GDB で以下のようにしたときと同じようにデバックすることができた。

    $ gdb ./sample 21250
    (gdb) b func
    Breakpoint 1 at 0x4004dc
    (gdb) command
    Type commands for when breakpoint 1 is hit, one per line.
    End with a line saying just "end".
    >p $rdi
    >c
    >end
    (gdb) c
    Continuing.

    Breakpoint 1, 0x00000000004004dc in func ()
    $1 = 27

    Breakpoint 1, 0x00000000004004dc in func ()
    $2 = 28

    Breakpoint 1, 0x00000000004004dc in func ()
    $3 = 29

    ...<略>

ptrace を使用したデバッカ・プログラムを作るとで GDB より柔軟に情報採取をすることも可能になる。

## 参考

* [Eli Bendersky's website: How debuggers work: Part 2 - Breakpoints](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/)
* [main is usually a function: Implementing breakpoints on x86 Linux](http://mainisusuallyafunction.blogspot.jp/2011/01/implementing-breakpoints-on-x86-linux.html)
* [memologue: hogetrace - 関数コールトレーサ](http://d.hatena.ne.jp/yupo5656/20071008/p1)
