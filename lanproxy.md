# Lanproxy

## CVE-2021-3019

#### 靶场搭建
组件下载： https://github.com/ffay/lanproxy
#### 漏洞分析

```java
package org.fengfei.lanproxy.server.config.web;

public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {

    private static final String PAGE_FOLDER = System.getProperty("app.home", System.getProperty("user.dir"))
            + "/webpages";

    private static final String SERVER_VS = "LPS-0.1";

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {

        // GET返回页面；POST请求接口
        if (request.getMethod() != HttpMethod.POST) {
            outputPages(ctx, request);
            return;
        }

        ResponseInfo responseInfo = ApiRoute.run(request);

        // 错误码规则：除100取整为http状态码
        outputContent(ctx, request, responseInfo.getCode() / 100, JsonUtil.object2json(responseInfo),
                "Application/json;charset=utf-8");
    }

    /**
     * 输出静态资源数据
     *
     * @param ctx
     * @param request
     * @throws Exception
     */
    private void outputPages(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
        HttpResponseStatus status = HttpResponseStatus.OK;
        URI uri = new URI(request.getUri());
        String uriPath = uri.getPath();
        uriPath = uriPath.equals("/") ? "/index.html" : uriPath;
        String path = PAGE_FOLDER + uriPath;
        File rfile = new File(path);
        if (rfile.isDirectory()) {
            path = path + "/index.html";
            rfile = new File(path);
        }

        if (!rfile.exists()) {
            status = HttpResponseStatus.NOT_FOUND;
            outputContent(ctx, request, status.code(), status.toString(), "text/html");
            return;
        }

        if (HttpHeaders.is100ContinueExpected(request)) {
            send100Continue(ctx);
        }

        String mimeType = MimeType.getMimeType(MimeType.parseSuffix(path));
        long length = 0;
        RandomAccessFile raf = null;
        try {
            raf = new RandomAccessFile(rfile, "r");
            length = raf.length();
        } finally {
            if (length < 0 && raf != null) {
                raf.close();
            }
        }

        HttpResponse response = new DefaultHttpResponse(request.getProtocolVersion(), status);
        response.headers().set(HttpHeaders.Names.CONTENT_TYPE, mimeType);
        boolean keepAlive = HttpHeaders.isKeepAlive(request);
        if (keepAlive) {
            response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, length);
            response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
        }

        response.headers().set(Names.SERVER, SERVER_VS);
        ctx.write(response);

        if (ctx.pipeline().get(SslHandler.class) == null) {
            ctx.write(new DefaultFileRegion(raf.getChannel(), 0, length));
        } else {
            ctx.write(new ChunkedNioFile(raf.getChannel()));
        }

        ChannelFuture future = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
        if (!keepAlive) {
            future.addListener(ChannelFutureListener.CLOSE);
        }
    }

    private static void send100Continue(ChannelHandlerContext ctx) {
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
        ctx.writeAndFlush(response);
    }

}
```
1.PAGE_FOLDER获取了本地项目路径

2.channelRead0判断不是POST请求就调用outputPages()

3.获取GET请求的路径赋给urlPath

4.urlPath拼接PAGE_FOLDER赋给path

5.实例化path赋给rfile

6.RandomAccessFile读取rfile返回raf

7.ChannelHandlerContext.write(ctx.write)将raf写入通道，也就是回显给用户。
#### 漏洞修复

#### 漏洞利用
如果项目地址是D:/test/proxy-server/web/
payload请求后读取文件的地址就是D:/test/proxy-server/web/../conf/config.properties
#### PSF案例




## CVE-xxxx2