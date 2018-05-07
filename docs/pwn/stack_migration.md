# Stack Migration

stack migration 是在可以輸入的 ROPchain 長度不夠時，可以擴展輸入的方法

核心概念是將 ROPchain 分好幾次寫在指定的區域，最後再將 stack 移動過去執行

## leave

`leave` 是 stack migration 的關鍵，實際上 `leave` 做了兩個動作

```
mov rsp, rbp
pop rbp
```

而每次進行 buffer overflow 時，`rbp` 是可控的，在下一輪時也可以進一步控制 `rsp`

### rbp

buffer overflow 以後，stack 上的情況如下

在不用 stack migration 的時候，rbp 就隨便填就好了，反正之後不會用到，但要使用 stack migration 時，就必須先把 rbp 指定到特定的 buffer ，之後才能移動 stack

```
rsp             AAAAAAAA
rsp + 8         AAAAAAAA
rsp + 16        AAAAAAAA
...             ...
rbp             buffer      ＃ 這裡是原本的 rbp 指定的位置，就是 mov 後 stack 的最上層
ret             ROP gadget
```

接著在下一次遇到 `leave` 時，`mov rsp, rbp` 就會將 stack 改到 buffer 上了

## Simple Migration

在可以控制輸入長度時，可以一口氣將要用到的 gadgets 都寫上 buffer 並將 stack 移動過去執行，具體操作如下：

```
payload + buf + set_input_parameters + input_func + leave_ret 
           |        |                                  |
           |        +----------------------------------+
          rbp                    ROP chain
```

這邊要注意的是，input 要指定到 buf 上，且第一個必須要輸入 rbp，如果沒有要執行第二次的 migration 的話，就隨便填就好了

```
buf2 + rop_chain + [set_input_parameters + input_func + leave_ret]
```

### 流程

1. leave：把 `rbp` 設定成 `buf`
2. 執行 input_func 並將輸入讀取到 `buf`
    - 輸入新的 rbp 以及 ROP chain
3. leave：stack 跳轉到 `buf` 並將 `rbp` 設成 `buf2` 
4. 執行 ROP chain
5. 是否執行下一次 migration
    - 是：跳回 2 (`buf` 和 `buf2` 對調，多次 migration 就在 `buf` 和 `buf2` 互換即可)
    - 否：結束

## Fixed Size Migration

在不能控制輸入長度時，就只能使用原本的 `read` 來多次輸入 ROP chain

架設有一個 `read` 如下，只能夠輸入一個 Gadget

```
payload (4 bytes) + rbp + ret
```

這時候就只能 return 到原本的 `read` (以下稱 `main_read`)

仔細觀察一下會發現 `main_read` 其實會將輸入存到 `rbp - 0x20`，可以利用這個特性來寫入 ROP chain

### 第一次 migration

第一次只是為了將 stack 移動到 buf 上而已

```
payload (4 bytes) + buf1 + main_read
```

1. leave：把 `rbp` 設定成 `buf1`
2. 回到 `main_read` 讀取輸入，讀取到 `buf1 - 0x20` 
3. leave：把 `rsp` 設定成 `buf1` ，`rbp` 設定成 `buf1` 上寫的新 `rbp`

### 交替 migration 寫入 ROP chain

一開始是先送第一次 migration 的輸入

```
payload (4 bytes) + buf2 + main_read
      ｜              |
  rbp - 0x20         rbp
```

前面 4 bytes 就隨便放就好了，接著將 `rbp` 設成 `buf2` ，然後回到 `main_read`

第二次執行 `main_read` 時，會把輸入寫到 `buf2 - 0x20`，這時後輸入：

```
data (4 bytes) + buf1 + main_read
   ｜              |
buf2 - 0x20     rbp(buf2)
```

這樣輸入以後，就可以把 4 bytes 的 ROP chain 寫入 `buf2 - 0x20` ，然後再跳回 `buf1` 

接著就重複執行以上兩步就可以了，需要注意的是，如果每次都是給 `buf2` 會一直覆蓋掉，所以要一直更新位置

```
+-------------+------+-------------+-------------+--------------+
| buf2 - 0x20 | buf2 | buf2 + 0x20 | buf2 + 0x40 |     ...      |
+-------------+------+-------------+-------------+--------------+
       |         |          |             |
     第一次    第二次     第三次        第四次
```

