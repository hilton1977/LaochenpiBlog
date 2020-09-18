---
title: 大数据 Json 压缩
tags:
  - Java
originContent: >
  ![](/images/java.jpg)


  ### 问题

  对于页面大数据传输`Content-type:text/html`可以看到`Content-Encoding:
  gzip`进行了`gzip`压缩，加快了页面渲染默认`application/json`不属于普通格式不会被压缩处理，当遇到一个大`json`时，页面渲染会特别的慢


  ### SpringBoot 开启gzip 压缩

  `SpringBoot` 内置了 `Tomcat`可以通过配置文件进行`Gzip`压缩

  ``` yml

  server:
    compression:
      enabled: true
      min-response-size: 1024KB
  ```

  - min-response-size 相应大于某值后启用压缩

  - mime-types 媒体类型需要压缩的类：application/json、text/html 等

  - enabled 是否开启压缩，默认是 false


  ``` java

  public class Compression {
      private boolean enabled = false;
      private String[] mimeTypes = new String[]{"text/html", "text/xml", "text/plain", "text/css", "text/javascript", "application/javascript", "application/json", "application/xml"};
      private String[] excludedUserAgents = null;
      private DataSize minResponseSize = DataSize.ofKilobytes(2L);
  }

  ```
categories:
  - 代码块
toc: false
date: 2020-09-17 09:41:13
---

![](/images/java.jpg)

### 问题
对于页面大数据传输`Content-type:text/html`可以看到`Content-Encoding: gzip`进行了`gzip`压缩，加快了页面渲染默认`application/json`不属于普通格式不会被压缩处理，当遇到一个大`json`时，页面渲染会特别的慢

### SpringBoot 开启gzip 压缩
`SpringBoot` 内置了 `Tomcat`可以通过配置文件进行`Gzip`压缩
``` yml
server:
  compression:
    enabled: true
    min-response-size: 1024KB
```
- min-response-size 相应大于某值后启用压缩
- mime-types 媒体类型需要压缩的类：application/json、text/html 等
- enabled 是否开启压缩，默认是 false

``` java
public class Compression {
    private boolean enabled = false;
    private String[] mimeTypes = new String[]{"text/html", "text/xml", "text/plain", "text/css", "text/javascript", "application/javascript", "application/json", "application/xml"};
    private String[] excludedUserAgents = null;
    private DataSize minResponseSize = DataSize.ofKilobytes(2L);
}
```
