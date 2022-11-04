# Wayos-防火墙-后台命令执行漏洞

## 漏洞描述

Wayos 防火墙 后台存在命令执行漏洞，通过命令注入可以执行远程命令

## 漏洞影响

> [!NOTE]
>
> Wayos 防火墙 

## FOFA

> [!NOTE]
>
> body="Get_Verify_Info(hex_md5(user_string)."

## 漏洞复现

登录页面如下

![image-20210531224015364](Wayos-防火墙-后台命令执行漏洞.assets/1627363597974287.jpg)

登录后台后 ping 模块命令执行

![image-20210531224745218](Wayos-防火墙-后台命令执行漏洞.assets/16273635984185002.jpg)



