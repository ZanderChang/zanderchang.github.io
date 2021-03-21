---
title: MemLabs Memory Forensics Challenges
mathjax: false
date: 2020-01-10 21:07:47
categories:
tags:
---

[MemLabs](https://github.com/stuxnet999/MemLabs)

[volatility使用说明](https://github.com/volatilityfoundation/volatility/wiki/Command-Reference)

内存取证挑战，`Lab1~6`
<!-- more -->

# Lab 1 Beginner's Luck
Easy
```sh
volatility -f MemoryDump_Lab1.raw imageinfo # 检测profile类型
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 consoles # 获取cmd的指令和结果（cmdscan只包括cmd指令）
```
得到cmd的缓存中有`St4G3$1`的运行结果`ZmxhZ3t0aDFzXzFzX3RoM18xc3Rfc3Q0ZzMhIX0`解码得到`flag{th1s_1s_th3_1st_st4g3!!}`。

内存数据通过`DumpIt.exe`获取。

# Lab 2 A New World
Easy
```sh
volatility -f MemoryDump_Lab2.raw --profile=Win7SP1x64 envars # 查看环境变量
```
存在`NEW_TMP`环境变量，值为`C:\Windows\ZmxhZ3t3M2xjMG0zX1QwXyRUNGczXyFfT2ZfTDRCXzJ9`，解码得到`flag{w3lc0m3_T0_$T4g3_!_Of_L4B_2}`。

# Lab 3 The Evil's Den
InCTF 2018 Evil Crypter

Easy - Medium
```sh
volatility -f MemoryDump_Lab3.raw imageinfo # Win7SP1x86
volatility -f MemoryDump_Lab3.raw --profile=Win7SP1x86 iehistory # ie浏览器历史，在Desktop目录下有可疑文件suspision1.jpeg、evilscript.py.py和vip.txt
volatility -f MemoryDump_Lab3.raw --profile=Win7SP1x86 filescan # 扫描文件，获取文件对应的Offset
volatility -f MemoryDump_Lab3.raw --profile=Win7SP1x86 dumpfiles -D ./files -Q 0x000000003e727e50 -n # 提取vip.txt，其余文件同理
```
第一部分根据`evilscript.py.py`（先`xor(3)`后`base64`）和加密结果`vip.txt`可以逆向得到`inctf{0n3_h4lf`；根据[提示](https://github.com/stuxnet999/MemLabs/tree/master/Lab%203)使用`steghide extract -sf suspision1.jpeg`提取`suspision1.jpeg`中的隐藏信息，密码为`inctf{0n3_h4lf`，得到第二部分`_1s_n0t_3n0ugh}`，即为`inctf{0n3_h4lf_1s_n0t_3n0ugh}`。

# Lab 4 Obsession
[Just Do It - InCTF Internationals 2019](https://blog.bi0s.in/2019/09/24/Forensics/InCTFi19-JustDoIt/)

Medium
```sh
volatility -f MemoryDump_Lab4.raw imageinfo # Win7SP1x64
volatility -f MemoryDump_Lab4.raw --profile=Win7SP1x64 iehistory # 发现Important.txt但无法dumpfiles
volatility -f MemoryDump_Lab4.raw --profile=Win7SP1x64 mftparser > mftparse # 利用MFT查看Important.txt
```
`Important.txt`已被删除，无法通过`dumpfiles`获取。使用`mftparser`检查`NTFS`的主控文件表`MFT`的`$DATA`属性（每个`MFT`表项为1024字节），参考[OMFW 2012: Reconstructing the MBR and MFT from Memory](https://volatility-labs.blogspot.com/2012/10/omfw-2012-reconstructing-mbr-and-mft.html)。最后在`mftparse`中得到`inctf{1_is_n0t_EQu4l_7o_2_bUt_th1s_d0s3nt_m4ke_s3ns3}`。

# Lab 5 Black Tuesday
Medium - Hard
```sh
volatility -f MemoryDump_Lab5.raw imageinfo # Win7SP1x64
volatility -f MemoryDump_Lab5.raw --profile=Win7SP1x64 iehistory
volatility -f MemoryDump_Lab5.raw --profile=Win7SP1x64 filescan # SW1wb3J0YW50.rar
volatility -f MemoryDump_Lab5.raw --profile=Win7SP1x64 dumpfiles -D ./files -Q 0x000000003eed56f0 -n
```
在`iehistory`中找到记录`C:/Windows/AppPatch/ZmxhZ3shIV93M0xMX2QwbjNfU3Q0ZzMtMV8wZl9MNEJfNV9EMG4zXyEhfQ.bmp`，`base64`解密得到`flag{!!_w3LL_d0n3_St4g3-1_0f_L4B_5_D0n3_!!}`。在`filescan`中找到记录`\Device\HarddiskVolume2\Users\SmartNet\Documents\SW1wb3J0YW50.rar`，`dumpfiles`下来后使用上一`flag`作为密码解压，得到`Stage2.png`，得到第二个`flag`，为`flag{W1th_th1s_$taGe_2_1s_c0mPL3T3_!!}`。

![](/img/memlabs/Stage2.png)

# Lab 6 The Reckoning
[InCTF 2019 - Notch It Up](https://fireshellsecurity.team/inctf2019-notch-it-up/)

Hard
```sh
volatility -f MemoryDump_Lab6.raw imageinfo # Win7SP1x64
volatility -f MemoryDump_Lab6.raw --profile=Win7SP1x64 iehistory
volatility -f MemoryDump_Lab6.raw --profile=Win7SP1x64 filescan # flag.rar History
volatility -f MemoryDump_Lab6.raw --profile=Win7SP1x64 pstree # WinRAR.exe 3716
volatility -f MemoryDump_Lab6.raw --profile=Win7SP1x64 envars -p 3716 # RAR password easypeasyvirus
```
查看`WinRAR.exe`的环境变量发现密码，解密`flag.rar`得到`flag2.png`，为`aN_Am4zING_!_i_gU3Ss???_}`。

![flag2.png](/img/memlabs/flag2.png)

```sh
volatility -f MemoryDump_Lab6.raw --profile=Win7SP1x64 dumpfiles -D ./files -Q 0x000000005da5a610 -n # History.vacb History.dat
volatility -f MemoryDump_Lab6.raw --profile=Win7SP1x64 screenshot -D ./screenshot
```
检查`Chrome`的`\User Data\Default\History`，获得`sqlite3`数据库文件`History.dat`，查找记录`sqlite3 History.dat "select url from urls;"`发现可疑网址`https://pastebin.com/RSGSi1hk`。

里面有一Google文档的[地址](https://www.google.com/url?q=https://docs.google.com/document/d/1lptcksPt1l_w7Y29V4o6vkEnHToAPqiCkgNNZfS9rCk/edit?usp%3Dsharing&sa=D&source=hangouts&ust=1566208765722000&usg=AFQjCNHXd6Ck6F22MNQEsxdZo21JayPKug)，打开后为一大段拉丁语文字，中间有一个MEGA[链接](https://mega.nz/#!SrxQxYTQ)，但是存在密码保护。搜索整个内存`cat MemoryDump_Lab6.raw.strings | grep -i mega | grep -i key`，发现`THE KEY IS zyWxCjCYYSEMA-hZe552qWVXiPwa5TecODbjnsscMIU`即为下载密钥，获得`flag_.png`，为另一半flag：`inctf{thi5_cH4LL3Ng3_!s_g0nn4_b3_?_`。

（也可以通过`screenshot`获取浏览器标题然后再搜索）

![flag_.png](/img/memlabs/flag_.png)

`inctf{thi5_cH4LL3Ng3_!s_g0nn4_b3_?_aN_Am4zINg_!_i_gU3Ss???_}`

# 参考资料
- https://www.hackingarticles.in/memory-forensics-using-volatility-framework/