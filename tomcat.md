# Tomcat

## CVE-2021-24122

### 靶场搭建
- Apache Tomcat 10.0.0-M1 - 10.0.0-M9
- Apache Tomcat 9.0.0.M1 - 9.0.39
- Apache Tomcat 8.5.0 - 8.5.59
- Apache Tomcat 7.0.0 - 7.0.106 

受影响版本范围内即可

### 漏洞分析
根据NVD上的描述：

When serving resources from a network location using the NTFS file system, Apache Tomcat versions 10.0.0-M1 to 10.0.0-M9, 9.0.0.M1 to 9.0.39, 8.5.0 to 8.5.59 and 7.0.0 to 7.0.106 were susceptible to JSP source code disclosure in some configurations. The root cause was the unexpected behaviour of the JRE API File.getCanonicalPath() which in turn was caused by the inconsistent behaviour of the Windows API (FindFirstFileW) in some circumstances.

可以得到信息：windows环境，问题出在File.getCanonicalPath()对路径的处理方式和 FindFirstFileW不一样上面。

![avatar](../../../images/java/tomcat/CVE-2021-24122/1.png)

根据补丁可以看出处理方式不一样就不一样在对冒号的处理上

如何在路径里创建一个冒号呢？我首先想到的是windows里的符号链接
```
mklink index:.jsp index.jsp
```
给index.jsp创建一个带冒号的符号链接

![avatar](../../../images/java/tomcat/CVE-2021-24122/2.png)
![avatar](../../../images/java/tomcat/CVE-2021-24122/3.png)

在Apache Tomcat中访问index，Tomcat未能正确识别路径，导致jsp文件没有被解析，而是当作文本直接返回到页面中。

![avatar](../../../images/java/tomcat/CVE-2021-24122/4.png)

### 漏洞修复
- https://github.com/apache/tomcat#diff-1496e8a847a9b5af77f7f2429ce0a4eaea6b373475885b176d56514b8a2f03e5

### 参考

- https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-findfirstfilew

- https://www.rapid7.com/db/vulnerabilities/apache-tomcat-cve-2021-24122/

- https://www.jianshu.com/p/bb4412ef7c37

- java.io.WinNTFileSystem.canonicalize.WinNTFileSystem.java

