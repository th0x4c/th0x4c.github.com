---
layout: post
title: "[OS] OS コマンドによるボトルネック調査"
date: 2013-04-09 21:47
comments: true
categories: [OS, Linux]
---
## 目的

OS コマンドによるボトルネック調査方法をまとめる。

* CPU
* メモリ
* I/O
* ネットワーク

## 環境

* OS: CentOS 5.5
* Kernel: 2.6.18-194.el5 x86_64

## CPU

### サーバ全体の CPU 使用率

CPU 使用率を確認する。使用率が 100% に近くなっている(= idle が 0% に近くなっている)とボトルネック。

#### `top`

`top` では複数の論理 CPU がある場合もサーバ全体として 1 つに集約されて出力される。

    $ top
    top - 07:23:36 up 45 days, 17:41,  2 users,  load average: 7.22, 9.43, 8.03
    Tasks: 223 total,   1 running, 222 sleeping,   0 stopped,   0 zombie
    Cpu(s):  7.1%us,  8.0%sy,  0.0%ni, 70.8%id,  7.1%wa,  2.7%hi,  4.4%si,  0.0%st
    Mem:   4044532k total,  3735152k used,   309380k free,   180688k buffers
    Swap:  8159224k total,   629880k used,  7529344k free,  1822664k cached

      PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
     1314 oracle    15   0 1800m  83m  63m S  1.3  2.1  16:38.42 oracle
    32392 root      15   0  362m  39m  15m S  1.0  1.0  18:24.31 orarootagent.bi
      667 grid      15   0 1244m  41m  16m S  0.7  1.0  11:23.32 oraagent.bin
    32450 grid      RT   0  334m 121m  53m S  0.7  3.1  28:04.95 ocssd.bin
      442 grid      15   0  729m  32m  19m S  0.3  0.8   5:04.54 oracle
      676 root      15   0 1681m  27m  13m S  0.3  0.7  22:21.63 orarootagent.bi
     1016 oracle    15   0  826m  36m  16m S  0.3  0.9   7:57.93 oraagent.bin
     1300 oracle    -2   0 1781m  16m  14m S  0.3  0.4   0:28.27 oracle
     1306 oracle    15   0 1787m  24m  17m S  0.3  0.6   0:39.76 oracle

`Cpu(s):` で始まる行が CPU 使用率

    Cpu(s):  7.1%us,  8.0%sy,  0.0%ni, 70.8%id,  7.1%wa,  2.7%hi,  4.4%si,  0.0%st

(-b オプションによるバッチモードでなく)対話的に起動した場合は `1` を押すと個々の CPU 毎の
CPU 使用率が出力される。

    $ top # 起動後 1 を押下
    top - 07:25:32 up 45 days, 17:43,  2 users,  load average: 5.02, 8.37, 7.84
    Tasks: 223 total,   1 running, 222 sleeping,   0 stopped,   0 zombie
    Cpu0  :  5.6%us, 16.7%sy,  0.0%ni, 33.3%id, 41.7%wa,  0.0%hi,  2.8%si,  0.0%st
    Cpu1  :  8.1%us,  5.4%sy,  0.0%ni, 70.3%id, 13.5%wa,  0.0%hi,  2.7%si,  0.0%st
    Mem:   4044532k total,  3735368k used,   309164k free,   180704k buffers
    Swap:  8159224k total,   629880k used,  7529344k free,  1822916k cached

      PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
    32392 root      15   0  362m  39m  15m S  1.0  1.0  18:24.71 orarootagent.bi
    32450 grid      RT   0  334m 121m  53m S  1.0  3.1  28:05.51 ocssd.bin
      676 root      15   0 1681m  27m  13m S  0.7  0.7  22:22.10 orarootagent.bi
     1323 oracle    -2   0 1795m 280m 265m S  0.7  7.1  10:14.08 oracle
    32382 grid      15   0  172m  25m  11m S  0.7  0.6   1:30.89 gpnpd.bin
      442 grid      15   0  729m  32m  19m S  0.3  0.8   5:04.61 oracle
      477 grid      15   0  718m  21m  16m S  0.3  0.6   1:58.51 oracle
      667 grid      15   0 1244m  41m  16m S  0.3  1.0  11:23.55 oraagent.bin
     1016 oracle    15   0  826m  36m  16m S  0.3  0.9   7:58.08 oraagent.bin

本環境は CPU 数が2つのため、以下のようにそれぞれの CPU 使用率が出力されている。

    Cpu0  :  5.6%us, 16.7%sy,  0.0%ni, 33.3%id, 41.7%wa,  0.0%hi,  2.8%si,  0.0%st
    Cpu1  :  8.1%us,  5.4%sy,  0.0%ni, 70.3%id, 13.5%wa,  0.0%hi,  2.7%si,  0.0%st

#### `mpstat`

`mpstat` でも CPU 使用率が確認できる。デフォルトではすべての CPU が集約されて出力される。

    $ mpstat 2 3  # 2秒毎に3回出力
    Linux 2.6.18-194.el5 (sv1.local)     04/08/13

    07:33:04     CPU   %user   %nice    %sys %iowait    %irq   %soft  %steal   %idle    intr/s
    07:33:06     all    4.41    0.00    4.41   13.24    1.47    0.00    0.00   76.47   1409.09
    07:33:08     all    5.06    0.00    3.80    6.33    0.00    2.53    0.00   82.28   1481.40
    07:33:10     all   10.61    0.00   10.61   12.12    0.00    3.03    0.00   63.64   1468.75
    Average:     all    6.57    0.00    6.10   10.33    0.47    1.88    0.00   74.65   1455.56

個々の CPU の使用率を確認したい場合は、`-P ALL` オプションを付与する。

    $ mpstat -P ALL 2 3
    Linux 2.6.18-194.el5 (sv1.local)     04/08/13

    07:35:03     CPU   %user   %nice    %sys %iowait    %irq   %soft  %steal   %idle    intr/s
    07:35:05     all    9.09    0.00    5.19    7.79    0.00    2.60    0.00   75.32   1353.85
    07:35:05       0   10.26    0.00    5.13   15.38    2.56    2.56    0.00   64.10   1353.85
    07:35:05       1    7.69    0.00    2.56    2.56    0.00    2.56    0.00   84.62      0.00

    07:35:05     CPU   %user   %nice    %sys %iowait    %irq   %soft  %steal   %idle    intr/s
    07:35:07     all    5.80    0.00    4.35   10.14    0.00    2.90    0.00   76.81   1412.12
    07:35:07       0    3.03    0.00    6.06   12.12    0.00    3.03    0.00   75.76   1412.12
    07:35:07       1    5.88    0.00    2.94    8.82    0.00    0.00    0.00   82.35      0.00

    07:35:07     CPU   %user   %nice    %sys %iowait    %irq   %soft  %steal   %idle    intr/s
    07:35:09     all   10.00    0.00   15.00   13.33    0.00    1.67    0.00   60.00   1393.55
    07:35:09       0    6.45    0.00   19.35   25.81    0.00    3.23    0.00   45.16   1393.55
    07:35:09       1   13.33    0.00   13.33    0.00    0.00    0.00    0.00   73.33      0.00

    Average:     CPU   %user   %nice    %sys %iowait    %irq   %soft  %steal   %idle    intr/s
    Average:     all    8.25    0.00    7.77   10.19    0.00    2.43    0.00   71.36   1384.47
    Average:       0    6.80    0.00    9.71   17.48    0.97    2.91    0.00   62.14   1384.47
    Average:       1    8.74    0.00    5.83    3.88    0.00    0.97    0.00   80.58      0.00

各項目の意味は以下の通り。

| 項目    | 説明                                                 |
|:--------|:-----------------------------------------------------|
| CPU     | CPU番号。ALLの場合は、全CPUの平均値であることを示す。
| %user   | ユーザレベル（アプリケーション）のCPU使用率
| %nice   | 優先度(ナイス値)によるユーザーレベルのCPU使用率
| %sys    | システムレベル(kernel)のCPU使用率
| %iowait | ディスクi/o競合によるCPU待機時間割合
| %irq    | CPUの割り込み実行時間割合
| %soft   | CPUのソフトウェア割り込み実行時間割合
| %idle   | CPUのアイドル時間割合(ディスクi/o待機時間はのぞく)
| intr/s  | 1秒あたりの平均割り込み数

#### `vmstat`

`vmstat` からも CPU 使用率が確認できる。

    $ vmstat 2 3
    procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
     r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
     1  0 629504 306172 181788 1825200    0    1   105   236   14   12  4  5 85  6  0
    10  1 629504 306048 181788 1825200    0    0    35   732  300 1425  7  5 77 11  0
     4  0 629504 306040 181788 1825204    0    0    27     2  236 1552  9 11 75  5  0

| 項目 | 説明                                                                                  |
|:-----|:--------------------------------------------------------------------------------------|
| r    | CPUを割り当て中もしくは割り当て可能なプロセスの数。CPUの個数以下であることが望ましい。
| b    | 割り込みを禁止しているプロセスの数。I/O待ちなどで割り込み不可能なときに発生。ゼロであることが望ましい。
| us   | ユーザー時間の CPU 使用率(nice 時間を含む) 
| sy   | システム時間の CPU 使用率
| id   | アイドル時間の割合
| wa   | IO 待ち時間の割合

CPU の割り当て状況を示す r, b の値も重要。r が CPU の個数と同じ場合は、システムの CPU がフルで使われており、
r が CPU 数より多い場合は、CPU の割り当てを待っていて、ボトルネックとなっている状況。
また、b が 0 より大きい場合は、I/O 等 CPU 以外のボトルネックが発生している可能性がある。


### プロセス単位の CPU 使用率

プロセス単位の CPU 使用率を確認し、CPU 使用率が 100% に近くなっているプロセスが
無いか確認する。

#### `top`

`top` によりプロセス単位の CPU 使用率が確認できる。

出力結果の下部にプロセス毎の情報があり、`%CPU` で CPU 使用率が確認できる。

      PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
     1314 oracle    15   0 1800m  83m  63m S  1.3  2.1  16:38.42 oracle
    32392 root      15   0  362m  39m  15m S  1.0  1.0  18:24.31 orarootagent.bi
      667 grid      15   0 1244m  41m  16m S  0.7  1.0  11:23.32 oraagent.bin
    32450 grid      RT   0  334m 121m  53m S  0.7  3.1  28:04.95 ocssd.bin
      442 grid      15   0  729m  32m  19m S  0.3  0.8   5:04.54 oracle
      676 root      15   0 1681m  27m  13m S  0.3  0.7  22:21.63 orarootagent.bi
     1016 oracle    15   0  826m  36m  16m S  0.3  0.9   7:57.93 oraagent.bin
     1300 oracle    -2   0 1781m  16m  14m S  0.3  0.4   0:28.27 oracle
     1306 oracle    15   0 1787m  24m  17m S  0.3  0.6   0:39.76 oracle

`top` を対話的に起動すると画面サイズ分しかプロセスが出力されず、すべてのプロセス
が確認できるわけではない。その場合は -b オプションでバッチモードで起動する。
例えば、バッチモードで 2 秒毎に 3 回出力する場合は、`top -b -d 2 -n 3` とする。

#### `ps aux`

`ps aux` の %CPU の項目でプロセス単位の CPU 使用率が確認できる。こちらはすべてのプロセスが
確認できる。

    $ ps aux
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root         1  0.0  0.0  10348   680 ?        Ss   Feb21   0:15 init [5]
    root         2  0.0  0.0      0     0 ?        S<   Feb21   5:34 [migration/0]
    root         3  0.0  0.0      0     0 ?        SN   Feb21   0:10 [ksoftirqd/0]
    root         4  0.0  0.0      0     0 ?        S<   Feb21   3:58 [migration/1]
    root         5  0.0  0.0      0     0 ?        SN   Feb21   0:18 [ksoftirqd/1]
    root         6  0.0  0.0      0     0 ?        S<   Feb21  33:22 [events/0]
    root         7  0.0  0.0      0     0 ?        S<   Feb21   0:06 [events/1]
    ...

### プロセス単位で CPU を使用している原因の特定

CPU を消費しているプロセスを特定したら、プロファイラ(OProfile, Valgrind(Callgrind)など)や
動的トレーサ(strace, ltrace など)で、どの関数で CPU を消費しているか特定していく。
簡易的には `pstack` を定期的に採取して、どの関数を通っている割合が多そうか確認する。


## メモリ

### 物理メモリ、スワップの確認

物理メモリのサイズを確認するには次を実行。

    $ grep MemTotal /proc/meminfo
    MemTotal:      4044532 kB

スワップのサイズを確認するには次を実行。

    $ grep SwapTotal /proc/meminfo
    SwapTotal:     8159224 kB

### サーバ全体の メモリ 使用量

メモリ使用量を確認して、物理メモリ以上使用されてスワップが多発していないか確認する。

#### `vmstat`, `free`, `cat /proc/meminfo`

同じタイミングで取得した `vmstat`, `free`, `cat /proc/meminfo` の出力結果は以下

`vmstat` の出力。

    $ vmstat 2 3
    procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
     r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
     1  0 629504 306172 181788 1825200    0    1   105   236   14   12  4  5 85  6  0
    10  1 629504 306048 181788 1825200    0    0    35   732  300 1425  7  5 77 11  0
     4  0 629504 306040 181788 1825204    0    0    27     2  236 1552  9 11 75  5  0

| 項目  | 説明                                                           |
|:------|:---------------------------------------------------------------|
| swapd | 仮想メモリの量(KB)
| free  | 空きメモリの量(KB)
| buff  | バッファに用いられているメモリの量(KB)
| cache | キャッシュに用いられているメモリの量(KB)

`free` の出力

    $ free
                 total       used       free     shared    buffers     cached
    Mem:       4044532    3738376     306156          0     181788    1825204
    -/+ buffers/cache:    1731384    2313148
    Swap:      8159224     629504    7529720

`cat /proc/meminfo` の出力

    $ cat /proc/meminfo
    MemTotal:      4044532 kB
    MemFree:        306148 kB
    Buffers:        181788 kB
    Cached:        1825204 kB
    SwapCached:     358460 kB
    Active:        2598312 kB
    Inactive:       867084 kB
    HighTotal:           0 kB
    HighFree:            0 kB
    LowTotal:      4044532 kB
    LowFree:        306148 kB
    SwapTotal:     8159224 kB
    SwapFree:      7529720 kB
    Dirty:             588 kB
    Writeback:           0 kB
    AnonPages:     1099912 kB
    Mapped:         687964 kB
    Slab:           153028 kB
    PageTables:      63084 kB
    NFS_Unstable:        0 kB
    Bounce:              0 kB
    CommitLimit:  10181488 kB
    Committed_AS:  6503324 kB
    VmallocTotal: 34359738367 kB
    VmallocUsed:    286400 kB
    VmallocChunk: 34359451067 kB
    HugePages_Total:     0
    HugePages_Free:      0
    HugePages_Rsvd:      0
    Hugepagesize:     2048 kB

| 項目        | 説明                                                            |
|:------------|:---------------------------------------------------------------|
| MemTotal    | システム全体で利用できる物理メモリの総容量。システム起動時に計算される。その後、この値が変化することはない。
| MemFree     | システム全体で利用できる物理メモリの空き容量
| Buffers     | ファイルなどのメタデータとして使用している物理メモリの総容量
| Cached      | ファイルデータのキャッシュなどに使用している物理メモリの総容量。共有メモリは Cached に加算される。SwapCachedは含まない。
| SwapCached  | 物理メモリ上にキャッシュされたスワップページの総容量
| Active      | 最近アクセスした（とカーネルが思っている）物理メモリの容量
| Inactive    | 最近アクセスしていない（とカーネルが思っている）、解放してよい物理メモリの容量
| Slab        | スラブアロケータで使用されている物理メモリの総容量
| VmallocUsed | vmalloc()で確保された物理メモリ領域とMMCONFIGで確保しているメモリ領域の総容量
| AnonPages   | 無名ページ（Anonymous Page）の領域。無名ページとは、ユーザープロセスがmalloc()などで確保したり、プログラム本体用に利用するメモリ領域。

出力を確認すると以下が分かる。

* 「`vmstat` の free」 = 「`free` の Mem: の free」 = 「`cat /proc/meminfo` の MemFree」
* 「`vmstat` の buff」 = 「`free` の Mem: の buffers」 = 「`cat /proc/meminfo` の Buffers」
* 「`vmstat` の cache」 = 「`free` の Mem: の cached」 = 「`cat /proc/meminfo` の Cached」

Linux では、空いているメモリはファイル I/O を効率化させるためにページキャッシュ
として利用する。
buffers と cached はページキャッシュ(の一部)であり、実際はストレージと同期がとれ
ていれば再利用可能なメモリである。

したがって利用可能な物理メモリ量は実際は、free + buffers + cached (`free` の free+)となる。

    |================= total =================|
    |= free =|============= used =============|

    +--------+----------+-----------+---------+
    |        |          |           |         |
    |        |          |           |         |
    |        |          |           |         |
    +--------+----------+-----------+---------+

             |= cached =|= buffers =|
    |============ free+ ============|= used- =|


厳密には、使用可能な物理メモリは「ストレージと同期されていない」ページキャッシュを除かないといけない。
ストレージと同期されており、すぐに再利用可能なメモリは「`cat /proc/meminfo` の Inactive 」
(もしくは `vmstat -a`)にて確認ができる。したがって、実際に再利用可能なメモリは厳密には「`cat /proc/meminfo` の MemFree + Inactive」
となる。(概算としては、free + buffers + cached でよいと思う。)

ちなみに /proc/meminfo に関して原則として、以下のような計算式が成り立つ。

* MemTotal = MemFree + Active + Inactive + Slab + VmallocUsed + PageTables
* Active + Inactive = AnonPages + Cached + Buffers + SwapCached
* 利用可能なメモリ = MemFree + Inactive, 解放できないメモリ = Active + Slab + VmallocUsed + PageTables


#### スワップ状況の確認

サーバ全体のスワップ状況は、`vmstat` の swap 欄の si(ディスクからページインされるメモリの量 KB/秒), 
so(ディスクにページアウトしているメモリ量 KB/秒)から確認する。
si, so が定常的に 0 より大きい場合は、スワップが発生しているのでメモリ不足に陥っている。

    $ vmstat 2    # メモリを使用するプログラムを実行中

    procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
     r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
    20  1 731484  36160 180152 1722248    0    1   105   238    7    9  4  5 84  6  0
    16  1 731484  31292 180152 1722248    0    0    69    80  151  788 22 78  0  0  0
    13  3 731484  26836 180152 1722248    0    0    19   168  149  807 23 69  8  0  0
     2  0 733040  26528 179984 1720100    0    0    34    55  134  850 22 53 13 13  0
    ...<中略>
    19  2 797140  24992   4276 787872    0 2804    56  2859  254 1024 38 62  0  0  0
    19  1 814564  26708   2404 761484    0 5008   131  5191  246  912 37 63  0  0  0
     6  1 816372  25500   2236 755376    0  626   113   882  156  599 35 65  0  0  0
    28  4 824480  26208   1516 739372    0 3912   871  3934  265 1038 28 61  0 10  0
    16  2 840912  24156   1232 725428    0 3166   963  3216  196  834 30 66  0  5  0
    19  4 861804  25324   1268 718716    0 6790   323  7015  222  839 34 66  0  0  0
     7  2 890208  28728   1296 709336    0 8648   878  8650  282 1061 30 66  2  3  0
    19  3 919964  34312   1376 707076    0 10578   399 10820  293 1106 29 66  0  5  0
    24  6 987308  55048   1384 707584    0 24256   293 24326  425 1415 36 64  0  0  0
     2  9 987292  38232   1444 707816   32    0   207   158  189  738 48 50  0  2  0
     6  2 1009184  33604   1504 709208    0 10228   780 10291  342 1047 30 58  5  7  0
     5  2 1014212  26284   1536 709544    0 2078   259  2086  151  798 37 55  0  8  0
    21  1 1030672  27936   1556 710044   16 7522   323  7758  225  944 37 58  2  3  0
    10  1 1041180  26028   1604 710244   18 5318   239  5320  207 1013 38 62  0  0  0
    24  5 1049788  28312   1632 710332    0 3618    31  3681  117  494 44 56  0  0  0

| 項目 | 説明                                                                |
|:-----|:--------------------------------------------------------------------| 
| si   | ディスクからスワップインされているメモリの量 (KB/s)
| so   | ディスクにスワップしているメモリの量 (KB/s)

また、スワップ時は `kswapd0` というカーネルスレッドが動作するので、これが `top` などで確認して
CPU 使用率の上位に出現しているとスワップが多発している状況と判断できる。

### プロセス単位の メモリ 使用量

プロセス単位のメモリ使用量を確認して、物理メモリを多く消費しているプロセスが無いか確認する。

#### `top`

VIRT, RES, SHR, %MEM から確認する。RES が物理メモリ使用量。

    $ top
    top - 11:41:10 up 46 days, 21:59,  3 users,  load average: 9.69, 8.95, 7.96
    Tasks: 226 total,   9 running, 217 sleeping,   0 stopped,   0 zombie
    Cpu(s):  3.8%us,  4.4%sy,  0.1%ni, 84.5%id,  6.1%wa,  0.3%hi,  0.8%si,  0.0%st
    Mem:   4044532k total,  2077900k used,  1966632k free,     7208k buffers
    Swap:  8159224k total,   938488k used,  7220736k free,   745852k cached

      PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
      477 grid      15   0  718m  21m  16m S  1.0  0.6   2:35.70 oracle
      667 grid      15   0 1247m  44m  16m S  1.0  1.1  15:30.20 oraagent.bin
     1310 oracle    15   0 1781m  17m  15m S  1.0  0.4   1:34.40 oracle
     1314 oracle    15   0 1800m  84m  64m S  1.0  2.1  21:53.70 oracle
    32241 root      15   0  319m  51m  21m S  1.0  1.3   7:56.18 ohasd.bin
    32360 grid      15   0  310m  35m  15m S  1.0  0.9   7:53.41 oraagent.bin
    32582 root      16   0  240m  21m  10m S  1.0  0.6   8:48.50 octssd.bin
        1 root      15   0 10348  672  568 S  0.0  0.0   0:15.40 init

| 項目 | 説明                                                                |
|:-----|:--------------------------------------------------------------------| 
| VIRT | 使用している仮想メモリの総量
| RES  | 使用しているスワップされていない物理メモリの総量
| SHR  | 利用している共有メモリの総量。他のプロセスと共有される可能性がある。
| %MEM | 現在使用している利用可能な物理メモリの占有率


#### `ps aux`

%MEM, VSZ, RSS から確認する。RSS が物理メモリ使用量。

    $ ps aux
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root         1  0.0  0.0  10348   672 ?        Ss   Feb21   0:15 init [5]
    root         2  0.0  0.0      0     0 ?        S<   Feb21   5:40 [migration/0]
    ...<中略>
    grid       477  0.0  0.5 735924 22424 ?        Ss   Apr04   2:35 asm_gmon_+ASM1
    ...<中略>
    grid       667  0.2  1.1 1277500 45740 ?       Ssl  Apr04  15:30 /u01/app/11.2.0.3/grid/bin/oraagent.bin
    ...<中略>
    oracle    1310  0.0  0.4 1823952 17752 ?       Ss   Apr04   1:34 ora_ping_rac1
    oracle    1312  0.0  0.4 1823952 17188 ?       Ss   Apr04   0:04 ora_acms_rac1
    oracle    1314  0.3  2.1 1843460 86036 ?       Ss   Apr04  21:53 ora_dia0_rac1
    ...

| 項目 | 説明                                                                |
|:-----|:--------------------------------------------------------------------| 
| %MEM | 現在使用している利用可能な物理メモリの占有率
| VSZ  | 使用している仮想メモリの総量(KB)
| RSS  | 使用しているスワップされていない物理メモリの総量(KB)


## I/O

### デバイス毎の I/O 状況

デバイス毎の I/O 状況を確認する。ビジー率が 100% に近かったり、IOPS, スループット(MB/s) が
カタログ・スペックと比較して限界性能に近かったり、定常的に I/O キューが溜まっていると
ボトルネックとなっている。

#### `iostat -x`

`iostat -x` によりデバイス毎の I/O 状況が確認できる。
最初の1回目はシステムがブートしてからその時点までの統計情報であるので注意。

    $ iostat -x 2 3
    Linux 2.6.18-194.el5 (sv1.local)       04/09/2013

    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
               1.63    0.01    1.58    1.91    0.00   94.88

    Device:         rrqm/s   wrqm/s   r/s   w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
    sda               0.22    17.31  0.43  8.47    19.48   206.31    25.34     0.32   36.46   3.53   3.14
    sda1              0.00     0.00  0.00  0.00     0.05     0.00    33.95     0.00    4.87   3.33   0.00
    sda2              0.21    17.31  0.42  8.47    19.41   206.31    25.38     0.32   36.53   3.53   3.14
    sdb               0.01     1.34  0.10  0.20     4.00    12.34    54.49     0.01   24.80   4.21   0.13
    sdb1              0.01     1.34  0.10  0.20     4.00    12.34    54.60     0.01   24.85   4.22   0.13
    sdc               0.07    13.21  0.52  4.53    49.09   141.94    37.83     0.16   31.58   8.10   4.09
    sdc1              0.07    13.21  0.52  4.53    49.09   141.94    37.83     0.16   31.58   8.10   4.09
    sdd               0.05     3.10  0.13  0.09    29.64    25.52   244.66     0.01   35.69   5.43   0.12
    sdd1              0.05     3.10  0.13  0.09    29.63    25.52   245.32     0.01   35.79   5.45   0.12
    dm-0              0.00     0.00  1.01 43.09    66.84   344.69     9.33     1.30   29.41   1.42   6.28
    dm-1              0.00     0.00  0.21  0.45     1.65     3.57     8.00     0.03   40.23   0.35   0.02
    dm-2              0.00     0.00  0.28  4.73    33.61    37.87    14.26     0.69  137.64   0.42   0.21

    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
               1.43    0.00    0.00    2.86    0.00   95.71

    Device:         rrqm/s   wrqm/s   r/s   w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
    sda               0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sda1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sda2              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdb               0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdb1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdc               0.00     7.14  0.00  4.29     0.00    91.43    21.33     0.49  114.33 107.00  45.86
    sdc1              0.00     7.14  0.00  4.29     0.00    91.43    21.33     0.49  114.33 107.00  45.86
    sdd               0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdd1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    dm-0              0.00     0.00  0.00 11.43     0.00    91.43     8.00     1.83  159.88  40.12  45.86
    dm-1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    dm-2              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00

    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
               1.19    0.00    0.00    1.19    0.00   97.62

    Device:         rrqm/s   wrqm/s   r/s   w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
    sda               0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sda1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sda2              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdb               0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdb1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdc               0.00     5.95  0.00  2.38     0.00    47.62    20.00     0.03    6.50   8.00   1.90
    sdc1              0.00     5.95  0.00  2.38     0.00    47.62    20.00     0.03    6.50   8.00   1.90
    sdd               0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    sdd1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    dm-0              0.00     0.00  0.00  9.52     0.00    76.19     8.00     0.07    4.62   2.00   1.90
    dm-1              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
    dm-2              0.00     0.00  0.00  0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00

| 項目     | 説明                                                                                         |
|:---------|:---------------------------------------------------------------------------------------------| 
| r/s      | 読み込みリクエスト数(回/秒)
| w/s      | 書き込みリクエスト数(回/秒)
| rsec/s   | 読み込みセクタ数(個/秒)。1 セクタ = 512bytes なので、512 を掛ければ読み込み byte 量が分かる。
| wsec/s   | 書き込みセクタ数(個/秒)。1 セクタ = 512bytes なので、512 を掛ければ書き込み byte 量が分かる。
| avgqu-sz | IOリクエストのキュー（待ち行列）の平均サイズ
| await    | IOリクエストの平均待ち時間（ミリ秒）。キューにいる時間＋処理時間。
| svctm    | IOリクエストの平均処理時間（ミリ秒）
| %util    | IOリクエスト実行中の CPU 時間の割合。この値が 100% に近いとビジーであり、ボトルネックとなる。

IOPS は `r/s + w/s` で算出できる。
スループットについては、`rsec/s`, `wsec/s` に 512 を掛ければ byte 単位のスループット(bytes/s)が算出できる。
`iostat -x -k` というように -k オプションを付与すると書込み量が kB/s で出力されるので見やすくなる。

ちなみに `dm-*` で表されているデバイスは、デバイス・マッパーと呼ばれるもので Logical Volume に対応している。
`lvdisplay` で出力される `Block device` の項目の右の数字や、`ls -l /dev/mapper` で LV との対応が分かる。

    $ sudo /usr/sbin/lvdisplay
      --- Logical volume ---
      LV Name                /dev/VolGroup01/LogVol00
      VG Name                VolGroup01
      LV UUID                hDNPM9-jBMV-7xto-mChi-ubR9-iN3i-styC8L
      LV Write Access        read/write
      LV Status              available
      # open                 1
      LV Size                39.99 GB
      Current LE             10237
      Segments               2
      Allocation             inherit
      Read ahead sectors     auto
      - currently set to     256
      Block device           253:2

      --- Logical volume ---
      LV Name                /dev/VolGroup00/LogVol00
      VG Name                VolGroup00
      LV UUID                2V6IUo-8Eui-DHq1-Nnzl-g0Pf-Z8HN-FIKz2E
      LV Write Access        read/write
      LV Status              available
      # open                 1
      LV Size                35.94 GB
      Current LE             1150
      Segments               2
      Allocation             inherit
      Read ahead sectors     auto
      - currently set to     256
      Block device           253:0

      --- Logical volume ---
      LV Name                /dev/VolGroup00/LogVol01
      VG Name                VolGroup00
      LV UUID                sotwax-PxcG-hwzh-dCG7-tkO1-Vy2m-W5qk5M
      LV Write Access        read/write
      LV Status              available
      # open                 1
      LV Size                3.91 GB
      Current LE             125
      Segments               1
      Allocation             inherit
      Read ahead sectors     auto
      - currently set to     256
      Block device           253:1

    $ ls -l /dev/mapper
    total 0
    crw------- 1 root root  10, 63 Apr  9 16:50 control
    brw-rw---- 1 root disk 253,  0 Apr  9 16:50 VolGroup00-LogVol00
    brw-rw---- 1 root disk 253,  1 Apr  9 16:50 VolGroup00-LogVol01
    brw-rw---- 1 root disk 253,  2 Apr  9 16:55 VolGroup01-LogVol00


## ネットワーク

### ネットワークインターフェイス毎の ネットワーク使用状況

ネットワークインターフェイス毎の ネットワーク使用状況を確認する。
ネットワーク帯域 bps (bit per second), 処理パケット数 pps (packet per second) といったスループットが
カタログ・スペックと比較して限界性能に近いとボトルネックとなっている。
bps は (byte でなく) bit 単位であることに注意。

また、処理するパケット数が多い(chatty な処理)と、CPU のソフトウェア割り込みが多くなるので、CPU 使用率で
ソフトウェア割り込みが多くなっていないかも確認する。(`mpstat` の `%soft` の項目から確認できる。)

#### `netstat -e -a -i -n`

`netstat -e -a -i -n` によりネットワークインターフェイス毎の ネットワーク使用状況が確認できる。
インターフェイス起動後からの累積値で表されるため、定期的に採取して差分を採る必要がある。

    $ netstat -e -a -i -n                                                                                                   vm13 /home/oracle 14:23
    Kernel Interface table
    eth0      Link encap:Ethernet  HWaddr 00:0C:29:1C:4C:61
              inet addr:192.168.238.138  Bcast:192.168.238.255  Mask:255.255.255.0
              inet6 addr: fe80::20c:29ff:fe1c:4c61/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:8836237 errors:0 dropped:0 overruns:0 frame:0
              TX packets:9148333 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:1491856405 (1.3 GiB)  TX bytes:6627509379 (6.1 GiB)

    eth0:1    Link encap:Ethernet  HWaddr 00:0C:29:1C:4C:61
              inet addr:192.168.238.141  Bcast:192.168.238.255  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

    eth0:3    Link encap:Ethernet  HWaddr 00:0C:29:1C:4C:61
              inet addr:192.168.238.140  Bcast:192.168.238.255  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

    eth0:5    Link encap:Ethernet  HWaddr 00:0C:29:1C:4C:61
              inet addr:192.168.238.143  Bcast:192.168.238.255  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

    eth1      Link encap:Ethernet  HWaddr 00:0C:29:1C:4C:6B
              inet addr:192.168.13.14  Bcast:192.168.13.255  Mask:255.255.255.0
              inet6 addr: fe80::20c:29ff:fe1c:4c6b/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:160311137 errors:0 dropped:0 overruns:0 frame:0
              TX packets:139265311 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:118804988857 (110.6 GiB)  TX bytes:84439638737 (78.6 GiB)

    eth1:1    Link encap:Ethernet  HWaddr 00:0C:29:1C:4C:6B
              inet addr:169.254.94.158  Bcast:169.254.255.255  Mask:255.255.0.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

    lo        Link encap:Local Loopback
              inet addr:127.0.0.1  Mask:255.0.0.0
              inet6 addr: ::1/128 Scope:Host
              UP LOOPBACK RUNNING  MTU:16436  Metric:1
              RX packets:23503874 errors:0 dropped:0 overruns:0 frame:0
              TX packets:23503874 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:20946402718 (19.5 GiB)  TX bytes:20946402718 (19.5 GiB)

    sit0      Link encap:IPv6-in-IPv4
              NOARP  MTU:1480  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)

| 項目       | 説明           |
|:-----------|:---------------|
| MTU        | MTU 値
| RX packets | 受信パケット数
| TX packets | 送信パケット数
| RX bytes   | 受信バイト数
| TX bytes   | 送信バイト数

## まとめ

まとめると、以下のような点を確認してボトルネックとなっていないか特定する。

* CPU: CPU 使用率が 100% に近くないか。
* メモリ: 物理メモリに空きがあるか(利用可能な物理メモリ量は十分か)、スワップが発生していないか。
* I/O: デバイスがビジーでないか、IOPS, スループットがスペック限界に達していないか。
* ネットワーク: ネットワークの帯域限界、パケット数の限界に達していないか。

## ボトルネック特定のフロー

[4.メモリ使用率(第5章 パフォーマンス管理～上級:基本管理コースII)](https://users.miraclelinux.com/technet/document/linux/training/2_5_4.html) のフローが参考になる。


### システム全体の調査

    +---------+
    | vmstat  |
    +----+----+
         |
         V
    +---------+  Yes   +------------------+
    | id < 10 +------->| CPU 使用率の評価 |
    +----+----+        +------------------+
         |
         | No
         V
    +---------+  No    +------------------+
    | so >  0 +------->| Disk 使用率の評価|
    +----+----+        +------------------+
         |
         | Yes
         V
    +----------+
    |メモリ不足|
    +----------+
        
### CPU 使用率の調査

    +---------+
    | vmstat  |
    +----+----+
         |
         V
    +---------+  Yes   
    | sy > 30 +-------------+
    +----+----+             |
         |                  V
         | No          +----------+  No    +------------------+
         |             | in > 200 +------->| Disk 使用率の評価|
         |             +----+-----+        +------------------+
         |                  |
         |                  | Yes
         |                  V
         |             +------------------+
         |             |ハードウェアの問題|
         V             +------------------+
    +----------+  Yes
    |  r > 0   +-------------+
    +----+-----+             |
         |                   |
         | No                |
         V                   V
    +------------+     +------------+
    |CPU のアップ|     | CPU の追加 |
    |グレード    |     +------------+
    +------------+

### Disk 使用率の調査

    +---------+
    |iostat -x|
    +----+----+
         |
         V
    +---------+  Yes   +------------------+
    |%util >80+------->|デバイスの負荷分散|
    +----+----+        +------------------+
         |
         | No
         V
    +---------+  Yes   +--------------------------+
    |w/s > r/s+------->|ディスク・キャッシュの使用|
    +----+----+        +--------------------------+
         |
         | No
         V
    +------------------+
    |ネットワークの調査|
    +------------------+

## 参考

* [Linuxトラブルシューティング探偵団　番外編（1）：減り続けるメモリ残量！ 果たしてその原因は!? (1/3) - ＠IT](http://www.atmarkit.co.jp/ait/articles/0810/01/news134.html)
* [ Linuxトラブルシューティング探偵団　番外編（3）：SystemTapで真犯人を捕まえろ！ (1/4) - ＠IT](http://www.atmarkit.co.jp/ait/articles/0903/25/news131.html)
* [2.CPU使用率(第5章 パフォーマンス管理～上級:基本管理コースII)](https://users.miraclelinux.com/technet/document/linux/training/2_5_2.html)
* [4.メモリ使用率(第5章 パフォーマンス管理～上級:基本管理コースII)](https://users.miraclelinux.com/technet/document/linux/training/2_5_4.html)
* [3. 性能管理実践編(システムリソース管理)](http://www.oracle.com/technetwork/jp/ats-tech/tech/useful-class-3-520773-ja.html)
* [6. システムがパフォーマンスを維持するためのメモリ管理について](http://www.oracle.com/technetwork/jp/ats-tech/tech/useful-class-6-520778-ja.html)
* [9. I/Oボトルネックの計測](http://www.oracle.com/technetwork/jp/ats-tech/tech/useful-class-9-520784-ja.html)
* [netstat - ホストのネットワーク統計や状態を確認する](http://www.atmarkit.co.jp/fnetwork/netcom/netstat/netstat.html)
