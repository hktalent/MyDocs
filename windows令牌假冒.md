# windows令牌假冒

------





## 访问令牌

------

**访问令牌**（Access Token）是Windows操作系统用于描述进程或线程安全上下文的一种对象。不同的用户登录计算机后， 都会生成一个Access Token，这个Token在用户创建进程或者线程时会被使用，不断的拷贝。

系统使用访问令牌来辨识拥有进程的用户，以及线程试图执行系统任务时是否具有所需的特权.与进程相关联，进程创建时根据LoginSession分配对应的TOKEN，含有与该进程用户账号、组信息、权限信息等。Token每次在用户登录时根据LoginSession分配，访问资源时提交Token进行身份验证。



**令牌分类**

- 访问令牌(Access Token): 表示访问控制操作主体的系统对象
- 会话令牌(Session Token): 交互会话中唯一的身份标识符
- 密保令牌(Security Token): 又叫做认证令牌或硬件令牌，是一种计算机身份校验的物理设备，例如U盾



windows系统下的令牌有两种表现形式:

1.Delegation token(授权令牌):用于交互会话登录(例如本地用户直接登录、远程桌面登录)
2.Impersonation token(模拟令牌):用于非交互登录(利用net use访问共享文件夹)

两种令牌都是在重启后才会消除，具有Delegation token的用户在注销后，该Token将变成Impersonation token，依旧有效。



## 令牌窃取/假冒

------

这里使用msf的模块对系统中从令牌进行窃取，并对获取到的令牌进行利用



### incognito模块

```
load incognito 加载模块
```

![image-20211021210024276](windows令牌假冒.assets/image-20211021210024276.png)

相关参数命令:

```
 list_tokens               列举token，-u参数列举用户，-g参数列举用户组
 impersonate_token [token] 令牌假冒登录用户
```



列举所有用户令牌

![image-20211021210254864](windows令牌假冒.assets/image-20211021210254864.png)



列举所有用户组令牌

![image-20211021210343650](windows令牌假冒.assets/image-20211021210343650.png)



此时的权限为SYSTEM权限

![image-20211021211343377](windows令牌假冒.assets/image-20211021211343377.png)



假冒NETWORK SERVICE的令牌，这里的\有两个，前面的符号是转义符号

```
impressonate_token ndsec-PC\\ndsec
```

![image-20211021211257111](windows令牌假冒.assets/image-20211021211257111.png)



令牌已经被修改

![image-20211021211407856](windows令牌假冒.assets/image-20211021211407856.png)



### steal_token盗取进程令牌

------

使用steal_token命令盗用进程的令牌



列出当前状态下服务器的运行的进程

```
ps
```

![image-20211021212304540](windows令牌假冒.assets/image-20211021212304540.png)



可以看到我们此时的会话进程号是1288，令牌是SYSTEM

![image-20211021212412558](windows令牌假冒.assets/image-20211021212412558.png)



盗取dwm.exe进程的令牌ndsec-PC\ndsec

```
steal_token 2884
```

