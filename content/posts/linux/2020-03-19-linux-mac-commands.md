---
title: Linux(MacOS)命令笔记
date: 2020-03-19 10:55:52
toc: true
tags:
    - linux
    - mac
    - command
---

linux中许多常用命令是必须掌握的，很多用过没有记录下来很快就忘记，再次使用又到处搜索，本文主要列出我日常使用过的一些命令，会持续更新。<!--more-->


## 1. netstat命令

linux：

>netstat -ntlp | grep 3000

>netstat -ntlp | grep nginx

*Tips: 注意加`sudo`赋予权限*

macos:

>netstat -an | grep 3306

3000替换成需要grep的端口号

## 2. lsof命令

通过list open file命令可以查看到当前打开文件，在linux中所有事物都是以文件形式存在，包括网络连接及硬件设备。

>lsof -i:80

-i参数表示网络链接，:80指明端口号，该命令会同时列出PID，方便kill

查看所有进程监听的端口

>sudo lsof -i -P | grep -i "listen"

## 3. ps -aux

>ps -aux | grep nginx 

ps命令用于查看当前正在运行的进程
如果要终止进程:

{% alert info %}
kill -9 [PID]
{% endalert %}

-9 表示强迫进程立即停止
通常用 ps 查看进程 PID ，用 kill 命令终止进程

## 4. pv

### 4.1 安装
ubuntu:

```
sudo apt-get install pv
```
### 4.2 使用

- 拷贝 `pv ~/a.log > ~/b.log`
- 拷贝限制速率 `pv -L 2m ~/a.log > ~/b.log`
- 字符一个个匀速在命令行中显示出来 `echo "Tecmint[dot]com is a community of Linux Nerds and Geeks" | pv -qL 10`
- 压缩文件展示进度信息`pv ~/a.mp4 | gzip > ~/a.mp4.gz`
- 往mysql导入表 `pv -i 1 -p -t -e /path/test.sql | mysql -hHOST -uUSER_NAME -p DATABASE_NAME`

## 5.Mac下报App损坏不可用

允许任何来源的应用。在系统偏好设置里，打开“安全性和隐私”，将“允许从以下位置下载的应用程序”设置为“任何来源“。这个设置已经无法在Mac OS Sierra上完成。在Mac OS Sierra上，应该进行以下操作：

（1）. 开启对任何来源：`sudo spctl --master-disable`

（2）. 移除应用的安全隔离属性 `sudo xattr -r -d com.apple.quarantine /Applications/xxxxx.app`

（3）. 重新打开app




