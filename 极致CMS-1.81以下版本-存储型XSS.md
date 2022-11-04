# 极致CMS-1.81以下版本-存储型XSS

### 漏洞复现

登录管理员添加模块

![](极致CMS-1.81以下版本-存储型XSS.assets/1627363286323327.jpg)

注册用户

![](极致CMS-1.81以下版本-存储型XSS.assets/1627363286586855.jpg)

点击发布文章

![](极致CMS-1.81以下版本-存储型XSS.assets/1627363286926416.jpg)

在文章标题处插入xss payload

```<details open ontoggle= confirm(document[`coo`+`kie`])>```

当管理员访问时XSS成功

![](极致CMS-1.81以下版本-存储型XSS.assets/1627363287471601.jpg)



### 参考

[极致CMS代码审计](https://xz.aliyun.com/t/7861)