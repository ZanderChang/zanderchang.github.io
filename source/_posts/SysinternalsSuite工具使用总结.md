---
title: SysinternalsSuite工具使用总结
mathjax: false
date: 2020-01-10 10:48:49
categories:
- 经验总结
tags:
---

`SysinternalsSuite`一共有`76`款实用工具程序，部分程序有`32`和`64`位之分。

<!-- more -->

`-accepteula`可以避免第一次运行时弹出的提示。

||名称|功能|
|-|-|-|
|1|accesschk(64)||
|2|AccessEnum||
|3|ADExplorer||
|4|ADInsight||
|5|adrestore||
|6|Autologon||
|7|Autoruns(64)||
|8|autorunsc(64)||
|9|Bginfo(64)||
|10|Cacheset||
|11|Clockres(64)||
|12|Contig(64)||
|13|Coreinfo|查看CPU信息，`-v`查看CPU对虚拟化的支持，需要管理员权限|
|14|ctrl2cap||
|15|Dbgview|查看`DbgPrint()`的输出，捕获kernel需要管理员权限|
|16|Desktops||
|17|disk2vhd||
|18|diskext(64)||
|19|Diskmon||
|20|DiskView||
|21|du(64)||
|22|efsdump||
|23|FindLinks(64)||
|24|handle(64)||
|25|hex2dec(64)||
|26|junction(64)||
|27|ldmdump||
|28|Listdlls(64)|`Listdlls64.exe -v chrome` 列出已启动程序加载的DLL|
|29|livekd(64)||
|30|LoadOrd(64)||
|31|LoadOrdC(64)||
|32|logonsessions(64)||
|33|movefile(64)||
|34|notmyfault(64)||
|35|notmyfaultc(64)||
|36|ntfsinfo(64)||
|37|pagedfrg||
|38|pendmoves(64)||
|39|pipelist(64)||
|40|portmon||
|41|procdump(64)|`Procdump64.exe -ma 6276 firefox-6276.dmp` dump进程内存空间|
|42|procexp(64)|查看系统进程信息|
|43|Procmon|根据设定的过滤条件监控进程的各种行为|
|44|PsExec(64)|交互式执行远程系统上的进程 [参考](https://mp.weixin.qq.com/s/HOck5KYFGrVvCxrRCtrnHw)|
|45|psfile(64)||
|46|PsGetsid(64)||
|47|PsInfo(64)||
|48|pskill(64)||
|49|pslist(64)||
|50|PsLoggedon(64)||
|51|psloglist(64)||
|52|pspasswd(64)||
|53|psping(64)||
|54|PsService(64)||
|55|psshutdown||
|56|pssuspend(64)||
|57|RAMMap||
|58|RegDelNull(64)||
|59|regjump||
|60|ru(64)||
|61|sdelete(64)||
|62|ShareEnum||
|63|ShellRunas||
|64|sigcheck(64)||
|65|streams(64)||
|66|strings(64)|`strings.exe –o -accepteula win7.dd`类似于`Linux`下的`strings -a -td win7.dd`，以`<decimal_offset>:<string>`的格式输出二进制文件中的字符串|
|67|sync(64)||
|68|Sysmon(64)||
|69|Tcpvcon||
|70|Tcpview||
|71|Testlimit(64)||
|72|vmmap||
|73|Volumeid(64)||
|74|whois(64)||
|75|Winobj|查看内核对象和命名空间，比如驱动|
|76|ZoomIt||