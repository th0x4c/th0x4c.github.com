---
layout: post
title: "[GDB] Linux x86-64 の呼出規約(calling convention)を gdb で確認する"
date: 2013-04-10 21:01
comments: true
categories: [GDB, OS, Linux]
---
## 目的

Linux x86-64 の呼出規約(calling convention)を gdb で確認する。

## 環境

* OS: CentOS 5.5
* Kernel: 2.6.18-194.el5 x86_64
* GCC: gcc 4.1.2 20080704
* GDB: GNU gdb 7.0.1-23.el5

## 呼出規約(calling convention)

プログラムで関数を呼び出す際に、レジスタやスタックを使いどのように引数を渡すか、戻り値をどのように受け取るかは呼出規約(calling convention)で決められている。

Linux x86-64 では、以下のような呼出規約になっている。
(Wikepedia [x86 calling conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions), [呼出規約](http://ja.wikipedia.org/wiki/%E5%91%BC%E5%87%BA%E8%A6%8F%E7%B4%84#System_V_AMD64_ABI_.E5.91.BC.E5.87.BA.E8.A6.8F.E7.B4.84) から抜粋)

> The calling convention of the System V AMD64 ABI is followed on Solaris,
> GNU/Linux, FreeBSD, and other non-Microsoft operating systems.
> The first six integer or pointer arguments are passed in registers RDI, RSI,
> RDX, RCX, R8, and R9, while XMM0, XMM1, XMM2, XMM3, XMM4, XMM5, XMM6 and XMM7
> are used for floating point arguments.
> For system calls, R10 is used instead of RCX.
> As in the Microsoft x64 calling convention, additional arguments are passed
> on the stack and the return value is stored in RAX.

* 整数・ポインタ引数: RDI, RSI, RDX, RCX, R8, R9
* 浮動小数点引数: XMM0, XMM1, XMM2, XMM3, XMM4, XMM5, XMM6, XMM7
* 戻り値: RAX
* システムコールでは RCX の代わりに R10 を使用
* レジスタだけでは引数の数が不足する場合はスタックを利用

より詳細には [System V Application Binary Interface AMD64 Architecture Processor Supplement](http://x86-64.org/documentation/abi.pdf) を参照。

## gdb による確認

上記の呼出規約を実際のプログラムで確認してみる。

使用するサンプルプログラムは以下。

{% codeblock lang:c %}
#include <stdio.h>

int sum(int a1, int a2, int a3, int a4, int a5,
        int a6, int a7, int a8, int a9);
void func();

int sum(int a1, int a2, int a3, int a4, int a5,
        int a6, int a7, int a8, int a9)
{
  int s = -9999;

  s = a1 + a2 + a3 + a4 + a5 + a6 + a7 + a8 + a9;

  return s;
}

void func()
{
  int ret = -1;

  ret = sum(11, 22, 33, 44, 55, 66, 77, 88, 99);

  printf("sum: %d\n", ret);
}

int main(int argc, char *argv[])
{
  func();
  return 0;
}
{% endcodeblock %}

実行結果は以下。

    $ gcc -g -o sample1 sample1.c
    $ ./sample1
    sum: 495

gdb から、関数 func から関数 sum を呼び出している個所を確認する。
`disas/m` というふうに修飾子 /m を指定するとソースと共に表示されて分かりやすくなる。
(ちなみに `objdump -d -S ./sample1` としてもソースと関数の逆アセンブル結果が出力される)

    $ gdb ./sample1
    (gdb) set height 0
    (gdb) disas/m func
    Dump of assembler code for function func:
    18      {
    0x00000000004004da <func+0>:    push   %rbp
    0x00000000004004db <func+1>:    mov    %rsp,%rbp
    0x00000000004004de <func+4>:    sub    $0x30,%rsp

    19        int ret = -1;
    0x00000000004004e2 <func+8>:    movl   $0xffffffff,-0x4(%rbp)

    20
    21        ret = sum(11, 22, 33, 44, 55, 66, 77, 88, 99);
    0x00000000004004e9 <func+15>:   movl   $0x63,0x10(%rsp)  <== 第9引数(0x63 = 99) を スタック+0x10 の位置に格納
    0x00000000004004f1 <func+23>:   movl   $0x58,0x8(%rsp)   <== 第8引数(0x58 = 88) を スタック+0x8  の位置に格納
    0x00000000004004f9 <func+31>:   movl   $0x4d,(%rsp)      <== 第7引数(0x4d = 77) を スタック      の位置に格納
    0x0000000000400500 <func+38>:   mov    $0x42,%r9d        <== 第6引数(0x42 = 66) を R9  に格納
    0x0000000000400506 <func+44>:   mov    $0x37,%r8d        <== 第5引数(0x37 = 55) を R8  に格納
    0x000000000040050c <func+50>:   mov    $0x2c,%ecx        <== 第4引数(0x2c = 44) を ECX に格納
    0x0000000000400511 <func+55>:   mov    $0x21,%edx        <== 第3引数(0x21 = 33) を EDX に格納
    0x0000000000400516 <func+60>:   mov    $0x16,%esi        <== 第2引数(0x16 = 22) を RSI に格納
    0x000000000040051b <func+65>:   mov    $0xb,%edi         <== 第1引数( 0xb = 11) を RDI に格納
    0x0000000000400520 <func+70>:   callq  0x400498 <sum>    <== 関数 sum を呼出
    0x0000000000400525 <func+75>:   mov    %eax,-0x4(%rbp)   <== 戻り値を RAX から受け取る

    22
    23        printf("sum: %d\n", ret);
    0x0000000000400528 <func+78>:   mov    -0x4(%rbp),%esi
    0x000000000040052b <func+81>:   mov    $0x400658,%edi
    0x0000000000400530 <func+86>:   mov    $0x0,%eax
    0x0000000000400535 <func+91>:   callq  0x400398 <printf@plt>

    24      }
    0x000000000040053a <func+96>:   leaveq
    0x000000000040053b <func+97>:   retq

    End of assembler dump.

確かに呼出規約通り、第1引数～第6引数まで RDI, RSI, RDX, RCX, R8, R9 を順に使用して、
残りの第7引数～第9引数はスタックを利用している。戻り値は RAX に格納されている。

ちなみに呼び出される関数 sum の disas の結果は以下

    (gdb) disas/m sum
    Dump of assembler code for function sum:
    9       {
    0x0000000000400498 <sum+0>:     push   %rbp
    0x0000000000400499 <sum+1>:     mov    %rsp,%rbp
    0x000000000040049c <sum+4>:     mov    %edi,-0x14(%rbp)
    0x000000000040049f <sum+7>:     mov    %esi,-0x18(%rbp)
    0x00000000004004a2 <sum+10>:    mov    %edx,-0x1c(%rbp)
    0x00000000004004a5 <sum+13>:    mov    %ecx,-0x20(%rbp)
    0x00000000004004a8 <sum+16>:    mov    %r8d,-0x24(%rbp)
    0x00000000004004ac <sum+20>:    mov    %r9d,-0x28(%rbp)

    10        int s = -9999;
    0x00000000004004b0 <sum+24>:    movl   $0xffffd8f1,-0x4(%rbp)

    11
    12        s = a1 + a2 + a3 + a4 + a5 + a6 + a7 + a8 + a9;
    0x00000000004004b7 <sum+31>:    mov    -0x18(%rbp),%eax
    0x00000000004004ba <sum+34>:    add    -0x14(%rbp),%eax
    0x00000000004004bd <sum+37>:    add    -0x1c(%rbp),%eax
    0x00000000004004c0 <sum+40>:    add    -0x20(%rbp),%eax
    0x00000000004004c3 <sum+43>:    add    -0x24(%rbp),%eax
    0x00000000004004c6 <sum+46>:    add    -0x28(%rbp),%eax
    0x00000000004004c9 <sum+49>:    add    0x10(%rbp),%eax
    0x00000000004004cc <sum+52>:    add    0x18(%rbp),%eax
    0x00000000004004cf <sum+55>:    add    0x20(%rbp),%eax
    0x00000000004004d2 <sum+58>:    mov    %eax,-0x4(%rbp)

    13
    14        return s;
    0x00000000004004d5 <sum+61>:    mov    -0x4(%rbp),%eax

    15      }
    0x00000000004004d8 <sum+64>:    leaveq
    0x00000000004004d9 <sum+65>:    retq

    End of assembler dump.

関数 sum で break させて、レジスタの状態やスタックを確認すれば引数の値が分かる。

    (gdb) b sum
    Breakpoint 1 at 0x4004b0: file sample1.c, line 10.
    (gdb) run
    Starting program: /home/hashi/tmp/sample1

    Breakpoint 1, sum (a1=11, a2=22, a3=33, a4=44, a5=55, a6=66, a7=77, a8=88, a9=99) at sample1.c:10
    10        int s = -9999;
    (gdb) info reg
    rax            0x0      0
    rbx            0x3e1c81bbc0     266766236608
    rcx            0x2c     44                        <== 第4引数
    rdx            0x21     33                        <== 第3引数
    rsi            0x16     22                        <== 第2引数
    rdi            0xb      11                        <== 第1引数
    rbp            0x7fffffffe270   0x7fffffffe270
    rsp            0x7fffffffe270   0x7fffffffe270
    r8             0x37     55                        <== 第5引数
    r9             0x42     66                        <== 第6引数
    r10            0x0      0
    r11            0x3e1ca1d8a0     266768341152
    r12            0x0      0
    r13            0x7fffffffe3b0   140737488348080
    r14            0x0      0
    r15            0x0      0
    rip            0x4004b0 0x4004b0 <sum+24>         <== sum+20 までの処理が終わっていて次の処理は sum+24
    eflags         0x202    [ IF ]
    cs             0x33     51
    ss             0x2b     43
    ds             0x0      0
    es             0x0      0
    fs             0x0      0
    gs             0x0      0
    fctrl          0x37f    895
    fstat          0x0      0
    ftag           0xffff   65535
    fiseg          0x0      0
    fioff          0x0      0
    foseg          0x0      0
    fooff          0x0      0
    fop            0x0      0
    mxcsr          0x1f80   [ IM DM ZM OM UM PM ]
    (gdb) x/6gx $rbp
    0x7fffffffe270: 0x00007fffffffe2b0      0x0000000000400525
    0x7fffffffe280: 0x000000000000004d      0x0000000000000058
                              ~~~~~~~~第7引数         ~~~~~~~~第8引数
    0x7fffffffe290: 0x2cb4304900000063      0x00000000004005a7
                              ~~~~~~~~第9引数

スタック上の値を確認するのに `x/6gx $rbp` としているのは、関数で break したときは関数の最初の方の処理(サブルーチンプロローグ)まで
終わっているが、普通はここまでで引数をスタックに積んだ後、

* `call <function>` : スタックに戻り先の命令アドレスを積んで関数を呼ぶ(rsp が -0x8 される)
* `push %rbp` : ベースポインタ(rbp)の値が push され、スタックに保存される(rsp が -0x8 される)
* `mov %rsp,%rbp` : スタックポインタ(rsp)の値をベースポインタ(rbp)に保存する
* (`sub $0xXX,%rsp` : ローカル変数を保持するためにスタックを上げる)

という処理が一般的に行われるためである。結果としてスタックに積まれた引数は `$rbp + 0x10` の位置から、第7引数が `$rbp + 0x10`, 第8引数が `$rbp + 0x18`, 第9引数が `$rbp + 0x20`,... というふうに存在することになる。

関数 sum から返った後に レジスタ RAX にて戻り値が分かる。

    (gdb) finish
    Run till exit from #0  sum (a1=11, a2=22, a3=33, a4=44, a5=55, a6=66, a7=77, a8=88, a9=99) at sample1.c:12
    0x0000000000400525 in func () at sample1.c:21
    21        ret = sum(11, 22, 33, 44, 55, 66, 77, 88, 99);
    Value returned is $1 = 495
    (gdb) p $rax
    $2 = 495

## 詳細確認

せっかくなので、関数 func から関数 sum を呼び出しているところを逆アセンブルの結果から詳しく追ってみる。

初期状態(関数 func が呼ばれる直前)

    rbp            0x7fffffffe2d0   0x7fffffffe2d0
    rsp            0x7fffffffe2c0   0x7fffffffe2c0

            0x7fffffffe2b8 +------------------+
                           |                  |
    rsp --> 0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
    rbp --> 0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


関数 main から関数 func を呼ぶ

    0x0000000000400550 <main+20>:   callq  0x4004da <func>  <== 関数 func を呼出

この命令後のレジスタの状態

    rbp            0x7fffffffe2d0   0x7fffffffe2d0
    rsp            0x7fffffffe2b8   0x7fffffffe2b8

    rip            0x4004da 0x4004da <func>

    rsp --> 0x7fffffffe2b8 +------------------+
                           |0x0000000000400555| <== 関数 main の戻り先命令アドレス
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
    rbp --> 0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


関数 func が呼び出され、ここから関数 func に入る。

    Dump of assembler code for function func:
    18      {
    0x00000000004004da <func+0>:    push   %rbp
    0x00000000004004db <func+1>:    mov    %rsp,%rbp
    0x00000000004004de <func+4>:    sub    $0x30,%rsp

この命令後のレジスタの状態

    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe280   0x7fffffffe280

    rsp --> 0x7fffffffe280 +------------------+ <== ローカル変数を保持するためにスタックを上げる
                           |                  |
            0x7fffffffe288 +------------------+
                           |                  |
            0x7fffffffe290 +------------------+
                           |                  |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |                  |
    rbp --> 0x7fffffffe2b0 +------------------+ <== 元のスタックの位置がベースポインタに保存される
                           |0x00007fffffffe2d0| <== ベースポインタの値がスタックに保存される
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+

進める

    19        int ret = -1;
    0x00000000004004e2 <func+8>:    movl   $0xffffffff,-0x4(%rbp)


この命令後のレジスタの状態

    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe280   0x7fffffffe280

    rsp --> 0x7fffffffe280 +------------------+
                           |                  |
            0x7fffffffe288 +------------------+
                           |                  |
            0x7fffffffe290 +------------------+
                           |                  |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
    rbp --> 0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+

進める

    20
    21        ret = sum(11, 22, 33, 44, 55, 66, 77, 88, 99);
    0x00000000004004e9 <func+15>:   movl   $0x63,0x10(%rsp)  <== 第9引数(0x63 = 99) を スタック+0x10 の位置に格納
    0x00000000004004f1 <func+23>:   movl   $0x58,0x8(%rsp)   <== 第8引数(0x58 = 88) を スタック+0x8  の位置に格納
    0x00000000004004f9 <func+31>:   movl   $0x4d,(%rsp)      <== 第7引数(0x4d = 77) を スタック      の位置に格納
    0x0000000000400500 <func+38>:   mov    $0x42,%r9d        <== 第6引数(0x42 = 66) を R9  に格納
    0x0000000000400506 <func+44>:   mov    $0x37,%r8d        <== 第5引数(0x37 = 55) を R8  に格納
    0x000000000040050c <func+50>:   mov    $0x2c,%ecx        <== 第4引数(0x2c = 44) を ECX に格納
    0x0000000000400511 <func+55>:   mov    $0x21,%edx        <== 第3引数(0x21 = 33) を EDX に格納
    0x0000000000400516 <func+60>:   mov    $0x16,%esi        <== 第2引数(0x16 = 22) を RSI に格納
    0x000000000040051b <func+65>:   mov    $0xb,%edi         <== 第1引数( 0xb = 11) を RDI に格納

この命令後のレジスタの状態

    rcx            0x2c     44                  <== 第4引数
    rdx            0x21     33                  <== 第3引数
    rsi            0x16     22                  <== 第2引数
    rdi            0xb      11                  <== 第1引数
    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe280   0x7fffffffe280
    r8             0x37     55                  <== 第5引数
    r9             0x42     66                  <== 第6引数

    rsp --> 0x7fffffffe280 +------------------+
                           |0x0000004d        | <== 第7引数
            0x7fffffffe288 +------------------+
                           |0x00000058        | <== 第8引数
            0x7fffffffe290 +------------------+
                           |0x00000063        | <== 第9引数
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
    rbp --> 0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


進める

    0x0000000000400520 <func+70>:   callq  0x400498 <sum>    <== 関数 sum を呼出

この命令後のレジスタの状態

    rcx            0x2c     44
    rdx            0x21     33
    rsi            0x16     22
    rdi            0xb      11
    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe278   0x7fffffffe278
    r8             0x37     55
    r9             0x42     66

    rip            0x400498 0x400498 <sum>  <== 次は関数 sum の命令を呼ぶ

    rsp --> 0x7fffffffe278 +------------------+
                           |0x0000000000400525| <== 関数 func の戻り先命令アドレス
            0x7fffffffe280 +------------------+
                           | 0x0000004d       |
            0x7fffffffe288 +------------------+
                           | 0x00000058       |
            0x7fffffffe290 +------------------+
                           | 0x00000063       |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |       0xffffffff |
    rbp --> 0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+

ここから関数 sum に入る

    Dump of assembler code for function sum:
    9       {
    0x0000000000400498 <sum+0>:     push   %rbp
    0x0000000000400499 <sum+1>:     mov    %rsp,%rbp
    0x000000000040049c <sum+4>:     mov    %edi,-0x14(%rbp)
    0x000000000040049f <sum+7>:     mov    %esi,-0x18(%rbp)
    0x00000000004004a2 <sum+10>:    mov    %edx,-0x1c(%rbp)
    0x00000000004004a5 <sum+13>:    mov    %ecx,-0x20(%rbp)
    0x00000000004004a8 <sum+16>:    mov    %r8d,-0x24(%rbp)
    0x00000000004004ac <sum+20>:    mov    %r9d,-0x28(%rbp)

この命令後のレジスタの状態

    rcx            0x2c     44
    rdx            0x21     33
    rsi            0x16     22
    rdi            0xb      11
    rbp            0x7fffffffe270   0x7fffffffe270
    rsp            0x7fffffffe270   0x7fffffffe270
    r8             0x37     55
    r9             0x42     66

            0x7fffffffe248 +------------------+
                           |0x00000042 0x00000037| <== 66 55
            0x7fffffffe250 +------------------+
                           |0x0000002c 0x00000021| <== 44 33
            0x7fffffffe258 +------------------+
                           |0x00000016 0x0000000b| <== 22 11
            0x7fffffffe260 +------------------+
                           |                  |
            0x7fffffffe268 +------------------+
                           |                  |
    rsp +-->0x7fffffffe270 +------------------+ <== 元のスタックの位置がベースポインタに保存される
    rbp +                  |0x00007fffffffe2b0| <== ベースポインタの値がスタックに保存される
            0x7fffffffe278 +------------------+
                           |0x0000000000400525|
            0x7fffffffe280 +------------------+
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
            0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+

進める

    10        int s = -9999;
    0x00000000004004b0 <sum+24>:    movl   $0xffffd8f1,-0x4(%rbp)

この命令後のレジスタの状態

    rcx            0x2c     44
    rdx            0x21     33
    rsi            0x16     22
    rdi            0xb      11
    rbp            0x7fffffffe270   0x7fffffffe270
    rsp            0x7fffffffe270   0x7fffffffe270
    r8             0x37     55
    r9             0x42     66

            0x7fffffffe248 +------------------+
                           |0x00000042 0x00000037|
            0x7fffffffe250 +------------------+
                           |0x0000002c 0x00000021|
            0x7fffffffe258 +------------------+
                           |0x00000016 0x0000000b|
            0x7fffffffe260 +------------------+
                           |                  |
            0x7fffffffe268 +------------------+
                           |        0xffffd8f1| <== -9999
    rsp +-->0x7fffffffe270 +------------------+
    rbp +                  |0x00007fffffffe2b0|
            0x7fffffffe278 +------------------+
                           |0x0000000000400525|
            0x7fffffffe280 +------------------+
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
            0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+

進める

    11
    12        s = a1 + a2 + a3 + a4 + a5 + a6 + a7 + a8 + a9;
    0x00000000004004b7 <sum+31>:    mov    -0x18(%rbp),%eax
    0x00000000004004ba <sum+34>:    add    -0x14(%rbp),%eax
    0x00000000004004bd <sum+37>:    add    -0x1c(%rbp),%eax
    0x00000000004004c0 <sum+40>:    add    -0x20(%rbp),%eax
    0x00000000004004c3 <sum+43>:    add    -0x24(%rbp),%eax
    0x00000000004004c6 <sum+46>:    add    -0x28(%rbp),%eax
    0x00000000004004c9 <sum+49>:    add    0x10(%rbp),%eax
    0x00000000004004cc <sum+52>:    add    0x18(%rbp),%eax
    0x00000000004004cf <sum+55>:    add    0x20(%rbp),%eax
    0x00000000004004d2 <sum+58>:    mov    %eax,-0x4(%rbp)

この命令後のレジスタの状態

    rax            0x1ef    495

    rcx            0x2c     44
    rdx            0x21     33
    rsi            0x16     22
    rdi            0xb      11
    rbp            0x7fffffffe270   0x7fffffffe270
    rsp            0x7fffffffe270   0x7fffffffe270
    r8             0x37     55
    r9             0x42     66

            0x7fffffffe248 +------------------+
                           |0x00000042 0x00000037|
            0x7fffffffe250 +------------------+
                           |0x0000002c 0x00000021|
            0x7fffffffe258 +------------------+
                           |0x00000016 0x0000000b|
            0x7fffffffe260 +------------------+
                           |                  |
            0x7fffffffe268 +------------------+
                           |        0x000001ef| <== 495
    rsp +-->0x7fffffffe270 +------------------+
    rbp +                  |0x00007fffffffe2b0|
            0x7fffffffe278 +------------------+
                           |0x0000000000400525|
            0x7fffffffe280 +------------------+
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
            0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


進める

    13
    14        return s;
    0x00000000004004d5 <sum+61>:    mov    -0x4(%rbp),%eax

この命令後のレジスタの状態

    rax            0x1ef    495 <== 戻り値が RAX に入っている

    rcx            0x2c     44
    rdx            0x21     33
    rsi            0x16     22
    rdi            0xb      11
    rbp            0x7fffffffe270   0x7fffffffe270
    rsp            0x7fffffffe270   0x7fffffffe270
    r8             0x37     55
    r9             0x42     66

            0x7fffffffe248 +------------------+
                           |0x00000042 0x00000037|
            0x7fffffffe250 +------------------+
                           |0x0000002c 0x00000021|
            0x7fffffffe258 +------------------+
                           |0x00000016 0x0000000b|
            0x7fffffffe260 +------------------+
                           |                  |
            0x7fffffffe268 +------------------+
                           |        0x000001ef|
    rsp +-->0x7fffffffe270 +------------------+
    rbp +                  |0x00007fffffffe2b0|
            0x7fffffffe278 +------------------+
                           |0x0000000000400525|
            0x7fffffffe280 +------------------+
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
            0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


進める

    15      }
    0x00000000004004d8 <sum+64>:    leaveq

この命令後のレジスタの状態

    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe278   0x7fffffffe278

            0x7fffffffe248 +------------------+
                           |0x00000042 0x00000037|
            0x7fffffffe250 +------------------+
                           |0x0000002c 0x00000021|
            0x7fffffffe258 +------------------+
                           |0x00000016 0x0000000b|
            0x7fffffffe260 +------------------+
                           |                  |
            0x7fffffffe268 +------------------+
                           |        0x000001ef|
            0x7fffffffe270 +------------------+
                           |0x00007fffffffe2b0|
    rsp --->0x7fffffffe278 +------------------+
                           |0x0000000000400525|
            0x7fffffffe280 +------------------+
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
    rbp --->0x7fffffffe2b0 +------------------+  <== ベースポインタが関数 sum 呼出直前に戻る
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


進める

    0x00000000004004d9 <sum+65>:    retq

この命令後のレジスタの状態

    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe280   0x7fffffffe280

    rip            0x400525 0x400525 <func+75>      <== 関数 func の戻り先アドレス

            0x7fffffffe248 +------------------+
                           |0x00000042 0x00000037|
            0x7fffffffe250 +------------------+
                           |0x0000002c 0x00000021|
            0x7fffffffe258 +------------------+
                           |0x00000016 0x0000000b|
            0x7fffffffe260 +------------------+
                           |                  |
            0x7fffffffe268 +------------------+
                           |        0x000001ef|
            0x7fffffffe270 +------------------+
                           |0x00007fffffffe2b0|
            0x7fffffffe278 +------------------+
                           |0x0000000000400525|
    rsp --> 0x7fffffffe280 +------------------+  <== スタックポインタが関数 sum 呼出直前に戻る
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0xffffffff|
    rbp --->0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


関数 func に戻ってきた

    0x0000000000400525 <func+75>:   mov    %eax,-0x4(%rbp)   <== 戻り値を RAX から受け取る

この命令後のレジスタの状態

    rax            0x1ef    495

    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe280   0x7fffffffe280

            0x7fffffffe248 +------------------+
                           |0x00000042 0x00000037|
            0x7fffffffe250 +------------------+
                           |0x0000002c 0x00000021|
            0x7fffffffe258 +------------------+
                           |0x00000016 0x0000000b|
            0x7fffffffe260 +------------------+
                           |                  |
            0x7fffffffe268 +------------------+
                           |        0x000001ef|
            0x7fffffffe270 +------------------+
                           |0x00007fffffffe2b0|
            0x7fffffffe278 +------------------+
                           |0x0000000000400525|
    rsp --> 0x7fffffffe280 +------------------+
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0x000001ef|  <== 戻り値を RAX から受け取って格納された
    rbp --->0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+

進める

    22
    23        printf("sum: %d\n", ret);
    0x0000000000400528 <func+78>:   mov    -0x4(%rbp),%esi
    0x000000000040052b <func+81>:   mov    $0x400658,%edi
    0x0000000000400530 <func+86>:   mov    $0x0,%eax
    0x0000000000400535 <func+91>:   callq  0x400398 <printf@plt>

この命令後のレジスタの状態(printf 関数から返った直後)

    rax            0x9      9      <=== printf からの戻り値(出力バイト数なので9バイト, "sum: 495\n")

    rsi            0x2aaaaaaac000   46912496123904  <== printf の中で使用されたので変わっている
    rdi            0x1      1                       <== printf の中で使用されたので変わっている
    rbp            0x7fffffffe2b0   0x7fffffffe2b0
    rsp            0x7fffffffe280   0x7fffffffe280

    rsp --> 0x7fffffffe280 +------------------+  <== スタックポインタより上位アドレス(この絵では↓)は printf 呼出前と同じ
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0x000001ef|
    rbp --->0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
            0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
            0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+

進める

    24      }
    0x000000000040053a <func+96>:   leaveq
    0x000000000040053b <func+97>:   retq

この命令後のレジスタの状態

    rbp            0x7fffffffe2d0   0x7fffffffe2d0  <== 関数 func 呼出直前に戻っている
    rsp            0x7fffffffe2c0   0x7fffffffe2c0  <== 関数 func 呼出直前に戻っている

    rip            0x400555 0x400555 <main+25>      <== 関数 main の戻り先アドレス

            0x7fffffffe280 +------------------+ 
                           |0x0000004d        |
            0x7fffffffe288 +------------------+
                           |0x00000058        |
            0x7fffffffe290 +------------------+
                           |0x00000063        |
            0x7fffffffe298 +------------------+
                           |                  |
            0x7fffffffe2a0 +------------------+
                           |                  |
            0x7fffffffe2a8 +------------------+
                           |        0x000001ef|
            0x7fffffffe2b0 +------------------+
                           |0x00007fffffffe2d0|
            0x7fffffffe2b8 +------------------+
                           |0x0000000000400555|
    rsp --> 0x7fffffffe2c0 +------------------+
                           |                  |
            0x7fffffffe2c8 +------------------+
                           |                  |
    rbp --> 0x7fffffffe2d0 +------------------+
                           |                  |
            0x7fffffffe2d8 +------------------+


## 参考

* [x86 calling conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions)
* [呼出規約](http://ja.wikipedia.org/wiki/%E5%91%BC%E5%87%BA%E8%A6%8F%E7%B4%84#System_V_AMD64_ABI_.E5.91.BC.E5.87.BA.E8.A6.8F.E7.B4.84)
* [System V Application Binary Interface AMD64 Architecture Processor Supplement](http://x86-64.org/documentation/abi.pdf)
