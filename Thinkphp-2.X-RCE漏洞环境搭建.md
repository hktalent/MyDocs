---
title: 'Thinkphp-2.X-RCE漏洞环境搭建'
date: Thu, 27 Aug 2020 16:29:03 +0000
draft: false
tags: ['白阁-环境库']
---

#### 安装docker环境

##### 点击查看教程

#### 下载vulhub项目

项目地址：[https://github.com/vulhub/vulhub](https://github.com/vulhub/vulhub)

下载后进入项目目录下thinkphp/2-rce目录：

```
cd vunhub-master/thinkphp/2-rce
```

开启本次漏洞环境：

```
docker-compose up -d
```

浏览器访问环境服务器**ip:8080** ![](Thinkphp-2.X-RCE漏洞环境搭建.assets/1627364296888139.jpg)

#### 环境搭建完成！