---
layout: post
title: "[Debug] LD_PRELOAD, dlsym, GCC拡張機能によって共有ライブラリの関数の呼出し前後で任意の処理を実行する"
date: 2013-06-25 23:23
comments: true
categories: [OS, Linux, GCC, Debug]
---
## 目的

LD_PRELOAD, dlsym, GCC拡張機能によって共有ライブラリの関数の呼出し前後で任意の処理を実行する。

## 環境

* OS: CentOS 5.5
* Kernel: 2.6.18-194.el5 x86_64
* GCC: gcc 4.1.2 20080704

## 使用する機能

### LD_PRELOAD

環境変数 LD_PRELOAD に共有ライブラリを指定すると、そのライブラリがすべてのライブラリに先立ってロードされる。
これを利用して通常ロードしている共有ライブラリ内の関数を置き換えることができる。(参考: `man ld.so`)

### dlsym

dlsym(3) は、シンボル名の文字列を引数に取り、そのシンボルのアドレスを返す。
これを利用して、関数のアドレスを得ることができる。(参考: `man dlsym`)

### GCC 拡張 `__attribute__((constructor))`, `__attribute__((deconstructor))`

GCC 拡張で `__attribute__` キーワードと共に関数の属性(attribute)を指定することができる。(参考: `info gcc` -> "C Extensions" -> "Function Attributes")

constructor 属性が指定された関数は、main() 関数が呼ばれる前に実行される。
deconstructor 属性が指定された関数は、main() 関数が完了するか exit() が呼ばれた後で実行される。

## 具体例

実際に LD_PRELOAD, dlsym, GCC拡張機能により共有ライブラリの関数の呼出し前後で任意の処理を実行してみる。

### 準備

今回使用するのは共有ライブラリを使用する以下のプログラム。

- foo.so 共有ライブラリ foo.h

{% codeblock lang:c %}
int foo(int x, int y, char *z);
{% endcodeblock %}

- foo.so 共有ライブラリ foo.c

{% codeblock lang:c %}
/* $ gcc -g -Wall -fPIC -shared -o libfoo.so foo.c */

#include "foo.h"
#include <stdio.h>

int foo(int x, int y, char *z)
{
  printf("[foo ] hello\n");
  printf("[foo ] request: %s\n", z);
  printf("[foo ] bye\n");
  return x + y;
}
{% endcodeblock %}

- main 関数(foo.so の関数を利用)

{% codeblock lang:c %}
/* $ gcc -g -Wall -L. -lfoo main.c */

#include "foo.h"
#include <stdio.h>

int main(int argc, char *argv[])
{
  int r = 0;

  printf("[main] hello\n");

  r = foo(10, 20, "ten plus twenty");
  printf("[main] return from foo: %d\n", r);

  printf("[main] bye\n");
  return 0;
}
{% endcodeblock %}

コンパイル方法と実行結果は以下。main 関数から共有ライブラリ foo.so 内の関数 foo() を呼んでいる。

    $ gcc -g -Wall -fPIC -shared -o libfoo.so foo.c
    $ gcc -g -Wall -L. -lfoo main.c
    $ ./a.out
    [main] hello
    [foo ] hello
    [foo ] request: ten plus twenty
    [foo ] bye
    [main] return from foo: 30
    [main] bye

### dlsym, GCC 拡張を使用した共有ライブラリの作成

上記の共有ライブラリ foo.so 内の関数 foo() の前後で処理をするために以下の bar.c から共有ライブラリ bar.so を作成する。

{% codeblock lang:c %}
/*
 * $ gcc -g -Wall -D_GNU_SOURCE -fPIC -shared -o libbar.so bar.c -ldl
 * $ env LD_PRELOAD=./libbar.so ./a.out
 */

#include <stdio.h>  /* printf */
#include <dlfcn.h>  /* dlsym RTLD_NEXT */
#include <unistd.h> /* _exit */

static void init() __attribute__((constructor));
static void fini() __attribute__((destructor));

static long (*original_foo)(long arg1, long arg2, void *arg3);

static void init()
{
  printf("[bar ] init\n");

  original_foo = dlsym(RTLD_NEXT, "foo");
  printf("[bar ] orignal_foo: %p\n", original_foo);

  if (original_foo == NULL)
    _exit(1);
}

static void fini()
{
  printf("[bar ] fini\n");
}

long foo(long arg1, long arg2, void *arg3)
{
  long ret = 0;

  printf("[bar ] before foo arg1: %ld, arg2: %ld, arg3: %p(\"%s\")\n",
         arg1, arg2, arg3, (char *)arg3);

  ret = (*original_foo)(arg1, arg2, arg3);

  printf("[bar ] after foo ret: %ld\n", ret);

  return ret;
}
{% endcodeblock %}

以下、コードの解説。

{% codeblock lang:c %}
static void init() __attribute__((constructor));
static void fini() __attribute__((destructor));
{% endcodeblock %}

GCC 拡張機能により、関数 init() に constructor 属性を設定し、main() 関数が呼ばれる前に実行されるようにしている。
また、関数 fini() に destructor 属性を設定し、プログラム終了直前に実行されるようにしている。

{% codeblock lang:c %}
static long (*original_foo)(long arg1, long arg2, void *arg3);

static void init()
{
  printf("[bar ] init\n");

  original_foo = dlsym(RTLD_NEXT, "foo");
  printf("[bar ] orignal_foo: %p\n", original_foo);

  if (original_foo == NULL)
    _exit(1);
}
{% endcodeblock %}

main() 関数が呼ばれる前に実行される init() 関数内で、オリジナルの foo() 関数(のアドレス)をグローバル変数 original_foo に格納している。dlsym() の引数に RTLD_NEXT を指定することで現在のライブラリ(この例では bar.so)以降で最初に関数が現れるところを探す。この機能により別の共有ライブラリ(この例では foo.so)の関数へのラッパーを提供することができる。

{% codeblock lang:c %}
long foo(long arg1, long arg2, void *arg3)
{
  long ret = 0;

  printf("[bar ] before foo arg1: %ld, arg2: %ld, arg3: %p(\"%s\")\n",
         arg1, arg2, arg3, (char *)arg3);

  ret = (*original_foo)(arg1, arg2, arg3);

  printf("[bar ] after foo ret: %ld\n", ret);

  return ret;
}
{% endcodeblock %}

オリジナルの foo() のラッパー関数として同一関数名を定義し、先ほど格納したオリジナルの関数を呼んでいる。その前後で任意の処理を実行している。(この例では引数や返値を出力している)

なお、この例のようにオリジナルの関数の引数の数は合わせたが、引数や返値の型が一致していなくても `long` や `void *` などオリジナルの型が入るような型であれば暗黙的に型変換してうまく動いてくれるようだ。(クローズドソースの共有ライブラリとかオリジナルの関数の引数の型が分からないようなケースでも推測してある程度合わせればOKということ。)

コンパイルは以下のように実施して libbar.so を作成する。`RTLD_NEXT` マクロを使用するため、`-D_GNU_SOURCE` を加えていることと、dlsym() を使用するために `-ldl` を加えていることに注意。

    $ gcc -g -Wall -D_GNU_SOURCE -fPIC -shared -o libbar.so bar.c -ldl
  
### LD_PRELOAD の使用

あとは、作成した libbar.so を LD_PRELOAD で指定してロードされるようにすればよい。
元の実行ファイルや共有ライブラリは再コンパイル、リリンクすることなしに foo() 関数を置き換えて、前後に処理を実行できている。
これを利用すれば、既存の共有ライブラリの関数の引数を出力するなど、デバックが容易にできる。

    $ env LD_PRELOAD=./libbar.so ./a.out
    [bar ] init
    [bar ] orignal_foo: 0x2b5fdd50d55c
    [main] hello
    [bar ] before foo arg1: 10, arg2: 20, arg3: 0x400785("ten plus twenty")
    [foo ] hello
    [foo ] request: ten plus twenty
    [foo ] bye
    [bar ] after foo ret: 30
    [main] return from foo: 30
    [main] bye
    [bar ] fini

LD_PRELOAD しなかったときの出力と比較して、foo() 関数の前後で引数情報などが出力できている。
また、main 関数前後でも処理が実行されていることが分かる。

## 参考

* [WATARU'S MEMO: [UNIX] malloc failure (その４)](http://memo.wnishida.com/?date=20060730)
* [Garbage In Garbage Out: Linux 共有ライブラリ(.so)の動作を変える。LD_PRELOADで楽しく遊ぶ](http://g1g0.com/2012/04/1790/)
