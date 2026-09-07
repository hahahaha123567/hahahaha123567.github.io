---
title: 处理HTTP接口上传文件大小超限异常
date: 2022-05-13
tags: 
- Java
description: 从HTTP接口上传文件大小超限异常查起，发现了当前业务系统存在的许多问题
---

# 问题描述

一个上传文件的HTTP POST接口，传参使用form-data格式，后端Java服务使用`MultipartFile`接收，在上传大文件后接口返回nginx 413错误

测试环境浏览器的请求直接发到应用服务器，请求发到80端口后nginx根据不同前缀将请求转发到对应Java服务的监听端口

# 排查问题

## nginx限制

nginx限制请求的大小，配置`client_max_body_size`生效，判断HTTP请求的大小依据是header中的`Content-Length`值

- size < client_body_buffer_size, 请求留在内存中
- client_body_buffer_size < size < client_max_body_size, 请求保存在临时文件中
- client_max_body_size < size, 返回413 Request Entity Too Large错误

[Nginx系列 | [转]Nginx 上传文件：client_max_body_size 、client_body_buffer_size_Tinywan的技术博客_51CTO博客](https://blog.51cto.com/tinywan/2867647)

[Module ngx_http_core_module client_body_buffer_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size)

[Module ngx_http_core_module client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)

调大`client_max_body_size`值后，请求成功发送到后端服务，但是返回的response body为空

## 文件过大抛出异常

查看后端服务日志，系统抛出了异常

```
org.springframework.web.multipart.MaxUploadSizeExceededException: Maximum upload size of 104857600 bytes exceeded; nested exception is org.apache.commons.fileupload.FileUploadBase$SizeLimitExceededException: the request was rejected because its size (113107647) exceeds the configured maximum (104857600)
	at org.springframework.web.multipart.commons.CommonsMultipartResolver.parseRequest(CommonsMultipartResolver.java:162)
	at org.springframework.web.multipart.commons.CommonsMultipartResolver$1.initializeMultipart(CommonsMultipartResolver.java:134)
	at org.springframework.web.multipart.support.AbstractMultipartHttpServletRequest.getMultipartFiles(AbstractMultipartHttpServletRequest.java:140)
	at org.springframework.web.multipart.support.AbstractMultipartHttpServletRequest.getFiles(AbstractMultipartHttpServletRequest.java:92)
Caused by: org.apache.commons.fileupload.FileUploadBase$SizeLimitExceededException: the request was rejected because its size (113107647) exceeds the configured maximum (104857600)
	at org.apache.commons.fileupload.FileUploadBase$FileItemIteratorImpl.<init>(FileUploadBase.java:968)
	at org.apache.commons.fileupload.FileUploadBase.getItemIterator(FileUploadBase.java:310)
	at org.apache.commons.fileupload.FileUploadBase.parseRequest(FileUploadBase.java:334)
	at org.apache.commons.fileupload.servlet.ServletFileUpload.parseRequest(ServletFileUpload.java:115)
	at org.springframework.web.multipart.commons.CommonsMultipartResolver.parseRequest(CommonsMultipartResolver.java:158)
	... 50 common frames omitted
```

可以看到，spring-web使用commons-fileupload包处理文件上传，限制文件大小`FileUploadBase`的`sizeMax`，参考类`CommonsFileUploadSupport`的相关初始化逻辑和`DispatcherServletAutoConfiguration.DispatcherServletConfiguration`的`multipartResolver`的注入逻辑，可以自定义一个MultipartResolver设置size限制并注入框架

本系统自定义一个`multipartResolver`并`setMaxUploadSize(100 * 1024 * 1024)`，限制文件最大100M，文件过大就会抛出上面的`MaxUploadSizeExceededException`和`FileUploadBase$SizeLimitExceededException`

因此我在自定义的`GlobalExceptionResolver`中加入对这两个exception的处理，打印日志并返回错误信息

上传大文件后在**测试**环境进行测试，该异常被捕获并执行了处理逻辑，但是前端收到的response依然没有HTTP body。奇怪的是，使用postman请求**本地**的后端服务可以正常返回错误信息

## 跟踪写HTTP response的过程

初步排查思路是跟踪spring-web把对象写到http body中的过程，查找是哪一步出了问题

debug排查发现，我们在`DispatcherServlet`中处理的`HttpServletRequest request`实体是tomcat的`RequestFacade`，使用facade外观模式，核心是`org.apache.catalina.connector.Request request`的`org.apache.coyote.Request coyoteRequest`

将对象写入http body的json序列化流程正常，但是我注意到本地环境的response header有一项`Transfer-Encoding: chunked`，测试环境的response header中没有这一项而多了`Connection: close`。body的写入使用outputStream，导致本地调试无法看到body值，header中的这个区别有可能就是导致测试环境异常的原因，因此我们开始检查是哪一步设置的connection=close

跟踪调用栈：
- DispatcherServlet.doService()
- DispatcherServlet.doDispatch()
- DispatcherServlet.processDispatchResult()
- DispatcherServlet.render()
- AbstractView.render()
- AbstractJackson2View.renderMergedOutputModel(), 从这里开始就是单纯的对象序列化为json字符串
- AbstractJackson2View.writeContent()
- com.fasterxml.jackson.databind.ObjectMapper.writeValue()
- DefaultSerializerProvider.serializeValue()
- DefaultSerializerProvider._serialize()
- serialize()

但是流程中观察到的对body流的写入都是正常的，各种字段类型的序列化也都使用了正确的序列化类，也没有找到设置connection的逻辑

## 监测HTTP header的写操作

从序列化流程开始逐步跟踪没有找到修改时机，我们改为监控所有修改response header操作，找到目标后再根据调用栈查看修改操作的调用方

response header最终保存在`org.apache.coyote.Request coyoteRequest`的`headers`字段中，所以我们在`MimeHeaders`类的`addValue()`和`setValue()`中打断点观察，最终发现是tomcat的`Http11Processor`在返回response前对消息体进行了统一修改，以符合各种RFC协议的要求
```java
/**
 * When committing the response, we have to validate the set of headers, as well as setup the response filters.
 */
@Override
protected final void prepareResponse() throws IOException {

    boolean entityBody = true;
    contentDelimitation = false;

    OutputFilter[] outputFilters = outputBuffer.getFilters();

    if (http09 == true) {
        // HTTP/0.9
        outputBuffer.addActiveFilter(outputFilters[Constants.IDENTITY_FILTER]);
        outputBuffer.commit();
        return;
    }

    int statusCode = response.getStatus();
    if (statusCode < 200 || statusCode == 204 || statusCode == 205 ||
            statusCode == 304) {
        // No entity body
        outputBuffer.addActiveFilter
            (outputFilters[Constants.VOID_FILTER]);
        entityBody = false;
        contentDelimitation = true;
        if (statusCode == 205) {
            // RFC 7231 requires the server to explicitly signal an empty response in this case
            response.setContentLength(0);
        } else {
            response.setContentLength(-1);
        }
    }

    // Check for compression
    boolean isCompressible = false;
    boolean useCompression = false;
    if (entityBody && (compressionLevel > 0) && sendfileData == null) {
        isCompressible = isCompressible();
        if (isCompressible) {
            useCompression = useCompression();
        }
        // Change content-length to -1 to force chunking
        if (useCompression) {
            response.setContentLength(-1);
        }
    }

    MimeHeaders headers = response.getMimeHeaders();
    // A SC_NO_CONTENT response may include entity headers
    if (entityBody || statusCode == HttpServletResponse.SC_NO_CONTENT) {
        String contentType = response.getContentType();
        if (contentType != null) {
            headers.setValue("Content-Type").setString(contentType);
        }
        String contentLanguage = response.getContentLanguage();
        if (contentLanguage != null) {
            headers.setValue("Content-Language")
                .setString(contentLanguage);
        }
    }

    // Add date header unless application has already set one (e.g. in a Caching Filter)
    if (headers.getValue("Date") == null) {
        headers.addValue("Date").setString(
                FastHttpDateFormat.getCurrentDate());
    }

    if ((entityBody) && (!contentDelimitation)) {
        // Mark as close the connection after the request, and add the connection: close header
        keepAlive = false;
    }

    // This may disabled keep-alive to check before working out the Connection header.
    checkExpectationAndResponseStatus();

    // If we know that the request is bad this early, add the Connection: close header.
    if (keepAlive && statusDropsConnection(statusCode)) {
        keepAlive = false;
    }
    if (!keepAlive) {
        // Avoid adding the close header twice
        if (!connectionClosePresent) {
            headers.addValue(Constants.CONNECTION).setString(Constants.CLOSE);
        }
    } else if (!http11 && !getErrorState().isError()) {
        headers.addValue(Constants.CONNECTION).setString(Constants.KEEPALIVE);
    }

    outputBuffer.commit();
}
```

那么本地环境和测试环境的区别在哪里呢？在最后判断Connection为close还是keep-alive的时候，processor根据请求的http版本执行不同操作，本地环境读到的是HTTP1.1，而测试环境读到的是HTTP1.0，就设置为了close。在浏览器的控制台-network中看到的protocol明明是h2，为什么后端服务读取到的是1.0呢？

[浏览器发起http请求时候，如何知道服务器支持什么http 版本？ - 知乎](https://www.zhihu.com/question/321943562)
[HTTP的版本是什么决定的，浏览器，服务器？ - SegmentFault 思否](https://segmentfault.com/q/1010000010612052)

越过nginx直接指定端口请求测试环境的服务，正常返回response body，因此判断是浏览器客户端和服务器协商HTTP版本时由于nginx限制没有成功使用

nginx增加配置项`proxy_http_version 1.1;`后，测试环境接口返回成功

# 其他

在找到了设置header的代码后再看下之前为什么debug没有跟踪到，设置header的调用栈：
- UTF8JsonGenerator.flush()
- CoyoteOutputStream.flush()
- org.apache.catalina.connector.OutputBuffer.flush()
- OutputBuffer.doFlush()
- org.apache.coyote.Response.sendHeaders()
- Response.action(), 这里的hook是`Http11Processor`
- AbstractProcessor.prepareResponse()
- Http11Processor.prepareResponse()

谁能想到名为`OutputStream`和`OutputBuffer`的类的flush方法竟然会执行这么多业务操作呢:(

POSTMAN不支持指定HTTP1.0，有这个需求可以导出curl指令后使用curl实现，增加`-0`参数

[How to change HTTP protocol version to HTTP 1.0 - Help - Postman](https://community.postman.com/t/how-to-change-http-protocol-version-to-http-1-0/3963)

在网上看到其他人遇到的相似问题

[Nginx proxy_http_version默认值引发的问题__alone_的博客-CSDN博客_proxy_http_version](https://blog.csdn.net/qq_38531706/article/details/117448200)
