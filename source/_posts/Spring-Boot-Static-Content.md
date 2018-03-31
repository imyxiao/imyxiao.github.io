---
title: Spring Boot静态资源处理
author: 肖哥
avatar: /yxiao.jpg
authorLink: https://blog.imyxiao.com
authorAbout: https://blog.imyxiao.com/about
authorDesc: Java Developer
date: 2016-12-16 22:19:58
categories: java
tags:
  - static
  - webjars
  - springboot
description: Spring Boot 静态资源处理，默认配置路径为 `classpath` 下的`/static` (或 `/public` 或 `/resources` 或 `/META-INF/resources`），即将`/**`映射到这几个路径下。Spring Boot 启动的时候，我们会看到如下日志
---
Spring Boot 静态资源处理，默认配置路径为 `classpath` 下的`/static` (或 `/public` 或 `/resources` 或 `/META-INF/resources`），即将`/**`映射到这几个路径下。Spring Boot 启动的时候，我们会看到如下日志：

```log
2016-12-16 22:27:56 INFO 9812 - [restartedMain] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2016-12-16 22:27:56 INFO 9812 - [restartedMain] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2016-12-16 22:27:56 INFO 9812 - [restartedMain] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
```

可以看到，Spring Boot 静态资源处理使用 Spring MVC 的`ResourceHttpRequestHandler`来实现。这里除了将`/**`映射到默认资源路径下，还为我们配置一个`favicon.ico`以及后面会讲到的`webjars`。Spring Boot中可以通过添加一个配置类，继承`WebMvcConfigurerAdapter`来自定义Spring MVC的配置，其中`addResourceHandlers`方法可以用来自定义静态资源处理，比如修改静态资源的缓存时间。这里我们想要替换默认的静态资源列表：
```java
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/**")
                .addResourceLocations("classpath:/mydir1/","classpath:/mydir2/","classpath:/mydir3/");
    }
}

```
此时，`/**`被映射到了`"classpath:/mydir1/"`,`"classpath:/mydir2/"`,`"classpath:/mydir3/"`下，要注意，如果替换了默认静态资源地址，那么默认欢迎页面`index.html`也将会被替换到我们自定义配置的任何路径下，Spring Boot相关的实现代码在`ResourceProperties`，这个类使用了`ConfigurationProperties`注解，前缀（prefix）为`spring.resources`，静态资源字段为`staticLocations`，那么我们也可以通过配置`spring.resources.staticLocations`这个properties来替换静态资源地址。

----------


<!-- more --> 
## 1.使用webjars ##

> 我们在Web开发中，前端页面中用了越来越多的JS或CSS，如jQuery等等，平时我们是将这些Web资源拷贝到Java的目录下，这种通过人工方式拷贝可能会产生版本误差，拷贝版本错误，前端页面就无法正确展示。 WebJars 就是为了解决这种问题衍生的，将这些Web前端资源打包成Java的Jar包，然后借助Maven这些依赖库的管理，保证这些Web资源版本唯一性，升级也比较容易。

webjars中的资源所在路径为：`/META-INF/resources/webjars/`，其中`/META-INF/resources`是默认资源路径，从刚才的日志当中也可以看到，Spirng Boot为其配置一个`/webjars/**`的Url映射。我们可以去[http://www.webjars.org/](http://www.webjars.org/ "http://www.webjars.org/") 找到我们想要的资源，将其依赖加入我们项目当中，即可使用这个资源了。例如：

```xml
<dependency>
	<groupId>org.webjars</groupId>
	<artifactId>jquery</artifactId>
	<version>2.1.4</version>
</dependency>
```
这里dependency的version即为静态资源的版本号，模板中（thymeleaf）使用:
```html
<head>
    <script th:src="@{/webjars/jquery/2.1.4/jquery.js}"></script>
</head>
```

## 2.静态资源处理 ##
Spring Boot 为我们提供了 Spring MVC 中比较高级的静态资源处理方法，比如为webjars添加动态版本号、清除缓存的静态资源等。

### 2.1.为Webjars添加动态版本号  ###
实际开发当中，我们不可避免的需要升级第三方js、css库的版本，如果我们有100个页面，那么一个一个页面的去替换版本号显然不是个好方法，Spring Boot项目中我们只需要简单的添加`webjars-locator`依赖，即可为Webjars添加动态版本号管理。
```xml
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>webjars-locator</artifactId>
</dependency>
```
此依赖使用`spring-boot-dependencies`中统一管理的版本号，然后模板中就不需要写版本号了：
```html
<head>
    <script th:src="@{/webjars/jquery/jquery.js}"></script>
</head>
```
生成的结果为：
```html
<head>
    <script th:src="/webjars/jquery/2.1.4/jquery.js"></script>
</head>
```
原理很简单，运行时使用`ResourceUrlEncodingFilter`这个过滤器，将`response`通过`HttpServletResponseWrapper`进行静态包装并重写`encodeURL`方法，然后模板中使用`response`的`encodeURL`将链接地址重写。并且一定要**注意**，`Thymeleaf` 和 `FreeMarker`这两个模板自动配置了`ResourceUrlEncodingFilter`，JSP则需要手动声明这个过滤器，而其它模板则需要自已去实现这个功能。下面说到的其它版本管理中，也是同理。

此时，如果想要更换jquery版本号，只需修改jquery的webjars的dependency版本即可。

### 2.2.缓存清除  ###

当静态资源发生变化时，为了清除用户客户端的缓存，我们一般会将静态资源加上版本号或者固定时间戮，每当资源变化时手动去更新版本或时间戳。比如：

```html
<link rel="stylesheet" type="text/css" href="/css/style.css?v=1.0.0"/>
```

这种方式也是需要我们一个页面一个页面的去修改，显然不合适。我们有更好的方式，Spring Boot中我们可以去配置`VersionResourceResolver`，它有两种策略：

- FixedVersionStrategy可以使用某项属性,或者日期,或者其它来作为版本.
- ContentVersionStrategy是使用资源内容计算出来的MD5哈希作为版本

与Webjars添加动态版本号一样，这两种方式都借助了`ResourceUrlEncodingFilter`来重写模板中的URL。

更详细配置可参考：Spring Framework’s[ reference documentation.](http://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#mvc-config-static-resources "Spring MVC的官方文档")

#### 2.2.1.ContentVersionStrategy  ####
properties文件配置方法如下：
```
spring.resources.chain.strategy.content.enabled=true
spring.resources.chain.strategy.content.paths=/**
```
同样也可以在继承`WebMvcConfigurerAdapter`的配置类中配置

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/**")
            .addResourceLocations("classpath:/static/")
            .resourceChain(true)
            .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
}
```
配置成功后，页面中的：
```html
<link rel="stylesheet" type="text/css" th:href="@{/css/style.css}"/>
```
会被替换为：
```html
<link rel="stylesheet" type="text/css" th:href="/css/style-2a2d595e6ed9a0b24f027f2b63b134d6.css"/>
```
文件的hash会被缓存，使用的是Spring的`ConcurrentMapCache`，一个简单的缓存实现，文件改动后，只有重启服务再次访问才会重新生成hash

#### 2.2.2.FixedVersionStrategy  ####
在动态的加载静态资源时，比如javascript模块加截器，这时更改文件名不是个好选择，可以使用`FixedVersionStrategy`，用某项属性,或者日期,或者其它来作为版本。

比如，在properties文件中这样配置
```
spring.resources.chain.strategy.content.enabled=true
spring.resources.chain.strategy.content.paths=/**
spring.resources.chain.strategy.fixed.enabled=true
spring.resources.chain.strategy.fixed.paths=/js/lib/
spring.resources.chain.strategy.fixed.version=v12
```

这样配置后, 位于 `"/js/lib/"`下的 JavaScript 将会使用一个固定的版本号` "/v12/js/lib/mymodule.js"` ，而其它的静态资源仍然使用文件hash的方式。

同样可以在类中配置：
```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/**")
            .addResourceLocations("classpath:/static/")
            .resourceChain(true)
            .addResolver(new VersionResourceResolver()
                                    .addFixedVersionStrategy("v12", "/js/lib/")
                                    .addContentVersionStrategy("/**"));
}
```

### 2.3.URL中含有"#"字符的问题  ###
查看`ResourceUrlEncodingFilter`这个过滤器，发现当URL中含有"?",比如
```html
<script th:src="@{/jquery.js?v=1}"></script>
```
`response`的`encodeURL`执行时，会首先将"?"后面的部分`?v=1`去掉，然后再根据`/jquery.js`去资源目录去查找文件
```java
private int getQueryParamsIndex(String url) {
	int index = url.indexOf("?");
	return (index > 0 ? index : url.length());
}
```
如果URL中有其它字符，比如`/jquery.js#1`，`encodeURL`执行时会拿着`/jquery.js#1`去查找文件，这样当然找不到，所以导致没法配置资源缓存清除这个功能了。

至于为什么我会遇到资源文件的URL中有"#"字符的情况呢？
```html
<svg>
    <use xlink:href="/svg-symbols.svg#grbs"></use>
</svg>
```
我们公司前端的svg引用就是这样写的，将很多svg放在一个文件中，通过"#"选择调用。

于是我重写了`ResourceUrlEncodingFilter`过滤器，添加"#"的判断，并优化了文件URL缓存。