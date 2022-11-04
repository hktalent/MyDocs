# 用友-NC-bsh.servlet.BshServlet-远程命令执行漏洞

## 漏洞描述

用友 NC bsh.servlet.BshServlet 存在远程命令执行漏洞，通过BeanShell 执行远程命令获取服务器权限

## 漏洞影响

> [!NOTE]
>
> 用友 NC 

## FOFA

> [!NOTE]
>
> icon_hash="1085941792"

## 漏洞复现

访问页面如下

![image-20210531220356962](用友-NC-bsh.servlet.BshServlet-远程命令执行漏洞.assets/16273635463418849.jpg)

漏洞Url为

```
/servlet/~ic/bsh.servlet.BshServlet
```

![image-20210531220503672](用友-NC-bsh.servlet.BshServlet-远程命令执行漏洞.assets/1627363546613452.jpg)