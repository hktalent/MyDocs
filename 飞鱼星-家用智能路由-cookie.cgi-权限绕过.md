# 飞鱼星-家用智能路由-cookie.cgi-权限绕过

## 漏洞描述

飞鱼星 家用智能路由存在权限绕过，通过Drop特定的请求包访问未授权的管理员页面

## 漏洞影响

> [!NOTE]
>
> 飞鱼星 家用智能路由 
>
> 飞鱼星 企业级智能上网行为管理系统

## FOFA

> [!NOTE]
>
> title="飞鱼星家用智能路由"

## 漏洞复现

登录页面如下

![](飞鱼星-家用智能路由-cookie.cgi-权限绕过.assets/1627363621518305.jpg)

访问 index.html 时会请求 cookie.cgi

```
http://xxx.xxx.xxx.xxx/index.html
```

页面抓包 Drop掉 cookie.cgi

![](飞鱼星-家用智能路由-cookie.cgi-权限绕过.assets/16273636217218199.jpg)

跳转后台获取了权限

![](飞鱼星-家用智能路由-cookie.cgi-权限绕过.assets/1627363622043443.jpg)