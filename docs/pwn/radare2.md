# radare2

Radare2 是一個可以在 linux 用的靜態分析工具

## Links

[Github](https://github.com/radare/radare2)

[Radare2 book](https://www.gitbook.com/book/radare/radare2book)

## Install

```bash
cd ~
git clone https://github.com/radare/radare2.git
cd radare2
sys/install.sh
```

## Start

使用時就直接打 r2 binary 就可以進入 r2 的介面了

```bash
r2 {binary}
```

一開始看到類似 shell 的畫面是 normal mode，可以輸入 `?` 來看可以使用的指令

```
[XD] % r2 ./test.sym
 -- Use 'e' and 't' in Visual mode to edit configuration and track flags.
[0x00400500]> ?
Usage: [.][times][cmd][~grep][@[@iter]addr!size][|>pipe] ; ...
Append '?' to any char command to get detailed help
Prefix with number to repeat command N times (f.ex: 3x)
|%var =valueAlias for 'env' command
| *[?] off[=[0x]value]    Pointer read/write data/values (see ?v, wx, wv)
| (macro arg0 arg1)       Manage scripting macros
| .[?] [-|(m)|f|!sh|cmd]  Define macro or load r2, cparse or rlang file
| =[?] [cmd]              Send/Listen for Remote Commands (rap://, http://, <fd>)
| <[...]                  Push escaped string into the RCons.readChar buffer
| /[?]                    Search for bytes, regexps, patterns, ..
| ![?] [cmd]              Run given command as in system(3)
| #[?] !lang [..]         Hashbang to run an rlang script
| a[?]                    Analysis commands
| b[?]                    Display or change the block size
| c[?] [arg]              Compare block with given data
| C[?]                    Code metadata (comments, format, hints, ..)
| d[?]                    Debugger commands
| e[?] [a[=b]]            List/get/set config evaluable vars
| f[?] [name][sz][at]     Add flag at current address
| g[?] [arg]              Generate shellcodes with r_egg
| i[?] [file]             Get info about opened file from r_bin
...
```

也可以再進一步用 `?` 看某個指令的用法

```
[0x00400500]> p?
|Usage: p[=68abcdDfiImrstuxz] [arg|len] [@addr]
| p-[?][jh] [mode]               bar|json|histogram blocks (mode: e?search.in)
| p=[?][bep] [blks] [len] [blk]  show entropy/printable chars/chars bars
| p2 [len]                       8x8 2bpp-tiles
| p3 [file]                      print stereogram (3D)
| p6[de] [len]                   base64 decode/encode
| p8[?][j] [len]                 8bit hexpair list of bytes
| pa[edD] [arg]                  pa:assemble  pa[dD]:disasm or pae: esil from hexpairs
| pA[n_ops]                      show n_ops address and type
| p[b|B|xb] [len] ([skip])       bindump N bits skipping M
| pb[?] [n]                      bitstream of N bits
| pB[?] [n]                      bitstream of N bytes
...
```

## Analyze

### aa

分析 binary

> 只會看 global name

### aaa

深度分析 binary

> function 底下使用到的 functoin 也會分析

### afl

列出分析過的 function

必須先使用 `aa` 或是 `aaa` 以後，用 `afl` 才會有結果

### afn

更改 function 名稱

```
afn {new_function_name} {old_function_name}
```

### afvn

更改區域變數的名稱

```
afvn {old_var_name} {new_var_name}
```

## Seek

### s

移動到某個 address 或 function

```
s {address}
s {function_name}
```

## Comment

### CC

在某處增加註解

> 註解中不能包含 @，可以用 \n 來換行

```
CC {comment} @ {address}
```

### CC-

刪除某處的註解

```
CC- @ {address}
```

## Print

### pd

印出從當前位置開始的 n 行指令

```
pd n 		# print n lines
pd 3		# print 3 lines
```

### pdc

印出從當前 functoin 的 C-like pseudo code

> 一定要在 function 的開頭才能用

```
pdc
```

## Project

radare2 的 project 管理，可以在標記完 symbol 以後儲存在 project ，下次就直接開啟同一個 project 即可

```
[0x00400618]> P?
|Usage: P[?osi] [file]Project management
| Pc [file]    show project script to console
| Pd [file]    delete project
| Pi [file]    show project information
| Pl           list all projects
| Pn[j]        show project notes (Pnj for json)
| Pn [base64]  set notes text
| Pn -         edit notes with cfg.editor
| Po [file]    open project
| Ps [file]    save project
| PS [file]    save script file
| P- [file]    delete project (alias for Pd)
| NOTE:        See 'e??prj.'
| NOTE:        project are stored in ~/.config/radare2/projects
```

### Ps

儲存當前專案

> 有專案以後，在關閉時會詢問是否儲存

```
Ps {project_name}
```

### Po

開啟某專案

```
Po {project_name}
```

### PS

儲存 script

將當前所有的操作 (主要是 symbol 上的) 儲存成 script

> 在開啟時用 `r2 -i {script} {binary}` 就把 symbol 都標回來了，不過會打不開 Visual Mode (原因不明)

```
PS {script_name}
```

## Quit

### q

退出 r2 ，或者是退出某個模式

```
q
```

## Visual Mode

visual mode 可以看到 binary 的圖形

### 進入

打一次 `V` 可以看到 binary 的 hexdump，再輸入一次 `V`  可以看到圖形

```
V
```

### 退出

退到上一層

```
q
```

### 圖形模式格式切換

可以切換看不同的顯示模式

```
p
```

### 輸入 normal mode 指令

先打 `:` 以後可以看到一個類似 normal mode shell 的介面，輸入完指令多按一次 enter 就可以退回 Visual Mode

```
: 				# 進入 normal mode shell
<enter> 		# 退回 Visual Mode
```

### define function

輸入 df 就會自動定義 function 了

### patch

先用 `s` 移動到要 patch 的位置，接著輸入 `A` 可以看到當前的指令

此時鍵入要改的組合語言即可

> 開啟時要用 r2 -w {file} 才能寫入

![](https://i.imgur.com/IUX1TxV.png)

## Evaluable vars

一些環境設定

### 開啟 jump table

分析 switch 的時候，開啟 jump table 可以連 switch options 的內容都分析

> 開啟以後，如果切換到 Visual Mode 會看到從 libc_start_main 開始的整張圖，要跑很久 (原因不明)

```
e anal.jmptbl = true
```

## Notes
- [r2 analyzes stripped binary](https://blog.techorganic.com/2016/03/08/radare-2-in-0x1e-minutes/)
