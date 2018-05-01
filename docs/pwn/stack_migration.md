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
rsp				AAAAAAAA
rsp + 8			AAAAAAAA
rsp + 16		AAAAAAAA
... 			...
rbp				first_buffer  ＃ 這裡是原本的 rbp 指定的位置，就是 mov 後 stack 的最上層
ret 			ROP gadget
```

