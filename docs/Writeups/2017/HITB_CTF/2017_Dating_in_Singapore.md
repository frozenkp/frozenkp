# 2017, Dating in Singapore

> Challenge Link: [2017, Dating in Singapore](http://hitb.xctf.org.cn/contest_challenge/)
>
> Category: MISC
>
> Writeup: [2017, Dating in Singapore](https://github.com/frozenkp/CTF/tree/master/2017/HITB_CTF/MISC/2017_Dating_in_Singapore)

01081522291516170310172431-050607132027162728-0102030209162330-02091623020310090910172423-02010814222930-0605041118252627-0203040310172431-0102030108152229151617-04050604111825181920-0108152229303124171003-261912052028211407-04051213192625

## 分析字串

- 用 - 分隔的話共有12段
- 每段的長度不一樣，但都是雙數

## 嘗試

- 每兩個一組轉成ascii -> 沒意思
- 轉成hex -> 沒意思

## 解法

- 觀察標題 “2017, Dating in Singapore” -> 可能跟日期有關
- 發現每兩個一組都不會超過31，又共有12個月
- 找到新加坡2017的月曆後將對應的日期標記出來

![](http://i.imgur.com/jJpjlMX.png)
