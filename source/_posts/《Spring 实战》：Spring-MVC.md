---
title: 《Spring 实战》：Spring MVC
date: 2017-05-31 01:56:47
tags: Spring
categories: [读书笔记, Spring]
---

## 1. 请求处理流程

![](https://user-images.githubusercontent.com/12514722/30251731-612d2c8c-9698-11e7-8c34-df6e0f96bd8f.png)
<!--more-->
在请求离开浏览器时①，会带有用户所请求内容的信息，至少会包含请求的 URL。

请求旅程的第一站是 Spring 的 `DispatcherServlet`。Spring MVC 所有的请求都会通过一个前端控制器（front controller）Servlet。前端控制器是常用的 Web 应用程序模式，在这里一个单实例的 Servlet 将请求委托给应用程序的其他组件来执行实际的处理。在 Spring MVC 中，`DispatcherServlet` 就是前端控制器。

`DispatcherServlet` 的任务是将请求发送给 Spring MVC 控制器（controller）。控制器是一个用于处理请求的 Spring 组件。在典型的应用程序中可能会有多个控制器，`DispatcherServlet` 需要知道应该将请求发送给哪个控制器。所以`DispatcherServlet` 以会查询一个或多个处理器映射（handler mapping）②来确定请求的下一站在哪里。处理器映射会根据请求所携带的 URL 信息来进行决策。

一旦选择了合适的控制器，`DispatcherServlet` 会将请求发送给选中的控制器③。到了控制器，请求会卸下其负载（用户提交的信息）并耐心等待控制器处理这些信息。（实际上，设计良好的控制器本身只处理很少甚至不处理工作，而是将业务逻辑委托给一个或多个服务对象进行处理。）

控制器在完成逻辑处理后，通常会产生一些信息，这些信息需要返回给用户并在浏览器上显示。这些信息被称为模型（model）。不过仅仅给用户返回原始的信息是不够的——这些信息需要以用户友好的方式进行格式化，一般会是 HTML。所以，信息需要发送给一个视图（view），通常会是 JSP。

控制器所做的最后一件事就是将模型数据打包，并且标示出用于渲染输出的视图名。它接下来会将请求连同模型和视图名发送回 `DispatcherServlet`④。

这样，控制器就不会与特定的视图相耦合，传递给 `DispatcherServlet` 的视图名并不直接表示某个特定的 JSP。实际上，它甚至并不能确定视图就是 JSP。相反，它仅仅传递了一个逻辑名称，这个名字将会用来查找产生结果的真正视图。`DispatcherServlet` 将会使用视图解析器（view resolver）⑤来将逻辑视图名匹配为一个特定的视图实现，它可能是也可能不是 JSP。

既然 `DispatcherServlet` 已经知道由哪个视图渲染结果，那请求的任务基本上也就完成了。它的最后一站是视图的实现（可能是 JSP）⑥，在这里它交付模型数据。请求的任务就完成了。视图将使用模型数据渲染输出，这个输出会通过响应对象传递给客户端。

## 2. 基于 Java 类配置的 DispatcherServlet
```Java
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;
public class MyWebAppInitializer extends AbstractAnnotationConfigDispactcherServletInitializer
{
    @Override
    protected String[] getServletMappings() // 将DispatcherServlet映射到“/”
    {
        return new String[] {"/"};
    }
    
    @Override
    protected class<?>[] getRootConfigClasses()
    {
        return new class<?>[] {RootConfig.class};
    }
    
    @Override
    protected class<?>[] getServletConfigClasses() // 指定配置类
    {
        return new class<?>[] {WebConfig.class};
    }
}
```

扩展 `AbstractAnnotationConfigDispatcherServletInitializer` 的任意类都会自动地配置 `DispatcherServlet` 和 Spring 应用上下文，Spring 的应用上下文会位于应用程序的 Servlet 上下文之中。

在 Servlet 3.0 环境中，容器会在类路径中查找实现 `javax.servlet.ServletContainerInitializer` 接口的类，如果能发现的话，就会用它来配置 Servlet 容器。

Spring 提供了这个接口的实现，名为 `SpringServletContainerInitializer`，这个类反过来又会查找实现 `WebApplicationInitializer` 的类并将配置的任务交给它们来完成。Spring 3.2 引入了一个便利的 `WebApplicationInitializer` 基础实现，也就是 `AbstractAnnotationConfigDispatcherServletInitializer`。因为我们的 `Spittr-WebAppInitializer` 扩展了 `AbstractAnnotationConfigDispatcherServletInitializer`（同时也就实现了 `WebApplicationInitializer`），因此当部署到 Servlet 3.0 容器中的时候，容器会自动发现它，并用它来配置 Servlet 上下文。

在上述程序中，`SpittrWebAppInitializer` 重写了三个方法：第一个方法 `getServletMappings()`，它会将一个或多个路径映射到 `DispatcherServlet` 上。在本例中，它映射的是“/”，这表示它会是应用的默认 Servlet。它会处理进入应用的所有请求。

`AbstractAnnotationConfigDispatcherServletInitializer` 会同时创建 `DispatcherServlet` 和 `ContextLoaderListener`。`GetServletConfigClasses()` 方法返回的带有 `@Configuration` 注解的类将会用来定义 `DispatcherServlet` 应用上下文中的 bean。`getRootConfigClasses()` 方法返回的带有 `@Configuration` 注解的类将会用来配置 `ContextLoaderListener` 创建的应用上下文中的 bean。

## 3. ViewResolver
Spring 自带了 12 个视图解析器，能够将逻辑视图名转换为物理实现。

| 视图解析器                          | 描述                                       |
| ------------------------------ | ---------------------------------------- |
| BeanNameViewResolver           | 将视图解析为 Spring 应用上下文中的 bean，其中 bean 的 ID 与视图的名字相同 |
| ContentNegotiatingViewResolver | 通过考虑客户端需要的内容类型来解析视图，委托给另外一个能够产生对应内容类型的视图解析器 |
| FreeMarkerViewResolver         | 将视图解析为 FreeMarker 模板                     |
| InternalResourceViewResolver   | 将视图解析为 Web 应用的内部资源（一般为 JSP）              |
| JasperReportsViewResolver      | 将视图解析为 JasperReports 定义                  |
| ResourceBundleViewResolver     | 将视图解析为资源 bundle（一般为属性文件）                 |
| TilesViewResolver              | 将视图解析为 Apache Tile 定义，其中 tile ID 与视图名称相同。注意有两个不同的 TilesViewResolver 实现，分别对应于 Tiles 2.0 和 Tiles 3.0 |
| UrlBasedViewResolver           | 直接根据视图的名称解析视图，视图的名称会匹配一个物理视图的定义          |
| VelocityLayoutViewResolver     | 将视图解析为 Velocity 布局，从不同的 Velocity 模板中组合页面 |
| VelocityViewResolver           | 将视图解析为 Velocity 模板                       |
| XmlViewResolver                | 将视图解析为特定 XML 文件中的 bean 定义。类似于 BeanNameViewResolver |
| XsltViewResolver               | 将视图解析为 XSLT 转换后的结果                       |
## 4. HTTPMessageConverter
消息转换（message conversion）它能够将控制器产生的数据转换为服务于客户端的表述形式。Spring 提供了多个 HTTP 信息转换器，用于实现资源表述与各种 Java 类型之间的互相转换。

| 信息转换器                                | 描述                                       |
| ------------------------------------ | ---------------------------------------- |
| AtomFeedHttpMessageConverter         | Rome Feed 对象和 Atom feed（媒体类型 application/atom+xml）之间的互相转换。如果 Rome 包在类路径下将会进行注册 |
| BufferedImageHttpMessageConverter    | BufferedImages 与图片二进制数据之间互相转换            |
| ByteArrayHttpMessageConverter        | 读取/写入字节数组。从所有媒体类型(\*/\*)中读取，并以 application/octet-stream 格式写入 |
| FormHttpMessageConverter             | 将 application/x-www-form-urlencoded 内容读入到 `MultiValueMap<String,String>` 中，也会将 `MultiValueMap<String,String>` 写入到 application/x-www-form-urlencoded 中或将 `MultiValueMap<String,Object>` 写入到 multipart/form-data 中 |
| Jaxb2RootElementHttpMessageConverter | 在 XML（ text/xml 或 application/xml）和使用 JAXB2 注解的对象间互相读取和写入。如果 JAXB v2 库在类路径下，将进行注册 |
| MappingJackson2HttpMessageConverter  | 在 JSON 和类型化的对象或非类型化的 HashMap 间互相读取和写入。如果 Jackson 2 JSON 库字类路径下，将进行注册 |
| MarshallingHttpMessageConverter      | 使用注入的编排器和解排器（marshaller 和 unmarshaller）来读取和写入 XML。支持的编排器和解排器包括 Castor、JAXB2、JIBX、XMLBeans 以及 Xstream |
| ResourceHttpMessageConverter         | 读取或写入 Resource                           |
| RssChannelHttpMessageConverter       | 在 RSS feed 和 Rome Channel 对象间互相读取或写入。如果 Rome 库在类路径下，将进行注册 |
| SourceHttpMessageConverter           | 在 XML 和 `javax.xml.transform.Source` 对象间互相读取和写入。默认注册 |
| StringHttpMessageConverter           | 将所有媒体类型(\*/\*)读取为 String。将 String 写入为 text/plain |
| XmlAwareFormHttpMessageConverter     | FormHttpMessageConverter 的扩展，使用 SourceHttpMessageConverter 来支持基于 XML 的部分 |

表中的 HTTP 信息转换器除了其中的五个以外都是自动注册的，所以要使用它们的话，不需要 Spring 配置。但是为了支持它们，需要添加一些库到应用程序的类路径下。

## 5. 异常
Spring 的一些异常会默认映射为 HTTP 状态码。

| Spring 异常                               | HTTP 状态码                     |
| --------------------------------------- | ---------------------------- |
| BindException                           | 400 - Bad Request            |
| ConversionNotSupportedException         | 500 - Internal Server Error  |
| HttpMediaTypeNotAcceptableExceptio      | 406 - Not Acceptable         |
| HttpMediaTypeNotSupportedException      | 415 - Unsupported Media Type |
| HttpMessageNotReadableException         | 400 - Bad Request            |
| MissingServletRequestParameterException | 400 - Bad Request            |
| MissingServletRequestPartException      | 400 - Bad Requestn           |
| NoSuchRequestHandlingMethodExceptio     | 404 - Not Found              |
| TypeMismatchException                   | 400 - Bad Request            |
| HttpMessageNotWritableException         | 500 - Internal Server Error  |
| HttpRequestMethodNotSupportedException  | 405 - Method Not Allowed     |
异常一般会由 Spring 自身抛出，作为 `DispatcherServlet` 处理过程中或执行校验时出现问题的结果。

Spring 提供了一种机制，能够通过 `@ResponseStatus` 注解将异常映射为 HTTP 状态码。（作用于自定义异常类上）
