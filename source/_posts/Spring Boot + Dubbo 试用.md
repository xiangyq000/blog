---
title: Spring Boot + Dubbo 试用
date: 2018-01-02 21:04:44
tags: Dubbo
categories: RPC
---

## 1. 启动 ZooKeeper

Mac 下启动单机版 ZooKeeper：

`zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties`

<!--more-->

## 2. dubbo-api

创建服务接口定义项目。该项目中只定义了一个服务接口 `iservice.IUserService`：

```java
public interface IUserService {

    public String hello(String s);
}
```

正常情况下应该将这个项目发布成 jar 包供提供方和消费者调用，不过这里为了简便起见，会在 Intellij 中直接让服务提供方和消费者依赖 dubbo-api 项目。

## 2. dubbo-provider

创建服务提供方项目 dubbo-provider，基于 Spring Boot，并设置依赖 dubbo-api。

1. 服务提供方启动类 `DubboProviderApplication`

   ```java
   @SpringBootApplication
   @ImportResource("classpath:dubbo-provider.xml")
   public class DubboProviderApplication {

   	public static void main(String[] args) {
   		SpringApplication.run(DubboProviderApplication.class, args);
   	}
   }
   ```

   这里 import 了 dubbo 项目的配置 xml。

2. 定义服务实现 `serviceimpl.UserServiceImpl`

   ```java
   public class UserServiceImpl implements IUserService {

       public String hello(String s) {
           return "hello " + s + "!";
       }
   }
   ```

3. 在 `application-properties` 里设置服务的 HTTP 端口 `server.port=8888`。

4. 在 classpath 下创建配置文件 `dubbo-provider.xml`：

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
          xsi:schemaLocation="http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd
          http://code.alibabatech.com/schema/dubbo
          http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
       <dubbo:application name="dubbo-provider" />
       <dubbo:registry address="zookeeper://localhost:2181"/>
       <dubbo:provider cluster="failfast"/>
       <dubbo:protocol name="dubbo" port="12345" />

       <bean id="userServiceImpl" class="xyq.serviceimpl.UserServiceImpl"/>
       <dubbo:service interface="iservice.IUserService" ref="userServiceImpl"/>
   </beans>
   ```

   这个 xml 中定义了：

   - 服务提供方的名称：dubbo-provider
   - 注册中心的协议及地址：运行在本地 2181 端口上的 ZooKeeper
   - dubbo 服务运行的端口：12345
   - 暴露的服务接口及其实现：`IUserService` 和 `UserServiceImpl`

5. 启动项目。

   ```
   2017-12-28 21:39:54.739  INFO 13377 --- [           main] xyq.DubboProviderApplication             : Starting DubboProviderApplication on bogon with PID 13377 (/Users/xyq/workspace/myproject/dubbo-provider/target/classes started by xyq in /Users/xyq/workspace)
   2017-12-28 21:39:54.743  INFO 13377 --- [           main] xyq.DubboProviderApplication             : No active profile set, falling back to default profiles: default
   2017-12-28 21:39:54.854  INFO 13377 --- [           main] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@2d127a61: startup date [Thu Dec 28 21:39:54 CST 2017]; root of context hierarchy
   2017-12-28 21:39:55.323  INFO 13377 --- [           main] o.s.b.f.xml.XmlBeanDefinitionReader      : Loading XML bean definitions from class path resource [dubbo-provider.xml]
   2017-12-28 21:39:55.530  INFO 13377 --- [           main] c.a.dubbo.common.logger.LoggerFactory    : using logger: com.alibaba.dubbo.common.logger.log4j.Log4jLoggerAdapter
   2017-12-28 21:39:56.281  INFO 13377 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8888 (http)
   2017-12-28 21:39:56.295  INFO 13377 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
   2017-12-28 21:39:56.296  INFO 13377 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.23
   2017-12-28 21:39:56.402  INFO 13377 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
   2017-12-28 21:39:56.402  INFO 13377 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1553 ms
   2017-12-28 21:39:56.549  INFO 13377 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'dispatcherServlet' to [/]
   2017-12-28 21:39:56.553  INFO 13377 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
   2017-12-28 21:39:56.554  INFO 13377 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
   2017-12-28 21:39:56.554  INFO 13377 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
   2017-12-28 21:39:56.554  INFO 13377 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
   2017-12-28 21:39:57.045  INFO 13377 --- [           main] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@2d127a61: startup date [Thu Dec 28 21:39:54 CST 2017]; root of context hierarchy
   2017-12-28 21:39:57.114  INFO 13377 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
   2017-12-28 21:39:57.115  INFO 13377 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
   2017-12-28 21:39:57.159  INFO 13377 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
   2017-12-28 21:39:57.159  INFO 13377 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
   2017-12-28 21:39:57.189  INFO 13377 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
   2017-12-28 21:39:57.318  INFO 13377 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
   2017-12-28 21:39:57.330  INFO 13377 --- [           main] com.alibaba.dubbo.config.AbstractConfig  :  [DUBBO] The service ready on spring started. service: iservice.IUserService, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.417  INFO 13377 --- [           main] com.alibaba.dubbo.config.AbstractConfig  :  [DUBBO] Export dubbo service iservice.IUserService to local registry, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.417  INFO 13377 --- [           main] com.alibaba.dubbo.config.AbstractConfig  :  [DUBBO] Export dubbo service iservice.IUserService to url dubbo://10.236.19.51:12345/iservice.IUserService?anyhost=true&application=dubbo-provider&bind.ip=10.236.19.51&bind.port=12345&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13377&side=provider&timestamp=1514468397346, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.417  INFO 13377 --- [           main] com.alibaba.dubbo.config.AbstractConfig  :  [DUBBO] Register dubbo service iservice.IUserService url dubbo://10.236.19.51:12345/iservice.IUserService?anyhost=true&application=dubbo-provider&bind.ip=10.236.19.51&bind.port=12345&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13377&side=provider&timestamp=1514468397346 to registry registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=dubbo-provider&dubbo=2.5.8&pid=13377&registry=zookeeper&timestamp=1514468397338, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.535  INFO 13377 --- [           main] c.a.d.remoting.transport.AbstractServer  :  [DUBBO] Start NettyServer bind /0.0.0.0:12345, export /10.236.19.51:12345, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.555  INFO 13377 --- [           main] c.a.d.r.zookeeper.ZookeeperRegistry      :  [DUBBO] Load registry store file /Users/xyq/.dubbo/dubbo-registry-dubbo-provider-localhost:2181.cache, data: {xyq.service.UserService=empty://10.236.19.51:12345/xyq.service.UserService?anyhost=true&application=dubbo-provider&category=configurators&check=false&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=xyq.service.UserService&methods=hello&pid=12958&side=provider&timestamp=1514467127634}, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.562  INFO 13377 --- [           main] c.a.d.common.concurrent.ExecutionList    :  [DUBBO] Executor for listenablefuture is null, will use default executor!, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.571  INFO 13377 --- [-localhost:2181] org.I0Itec.zkclient.ZkEventThread        : Starting ZkClient event thread.
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:zookeeper.version=3.3.3-1073969, built on 02/23/2011 22:27 GMT
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:host.name=bogon
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.version=1.8.0_91
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.vendor=Oracle Corporation
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.class.path=/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/tools.jar:/Users/xyq/workspace/myproject/dubbo-provider/target/classes:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter-web/1.5.9.RELEASE/spring-boot-starter-web-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter/1.5.9.RELEASE/spring-boot-starter-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot/1.5.9.RELEASE/spring-boot-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/1.5.9.RELEASE/spring-boot-autoconfigure-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter-logging/1.5.9.RELEASE/spring-boot-starter-logging-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/ch/qos/logback/logback-classic/1.1.11/logback-classic-1.1.11.jar:/Users/xyq/.m2/repository/ch/qos/logback/logback-core/1.1.11/logback-core-1.1.11.jar:/Users/xyq/.m2/repository/org/slf4j/jcl-over-slf4j/1.7.25/jcl-over-slf4j-1.7.25.jar:/Users/xyq/.m2/repository/org/slf4j/jul-to-slf4j/1.7.25/jul-to-slf4j-1.7.25.jar:/Users/xyq/.m2/repository/org/slf4j/log4j-over-slf4j/1.7.25/log4j-over-slf4j-1.7.25.jar:/Users/xyq/.m2/repository/org/yaml/snakeyaml/1.17/snakeyaml-1.17.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/1.5.9.RELEASE/spring-boot-starter-tomcat-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/8.5.23/tomcat-embed-core-8.5.23.jar:/Users/xyq/.m2/repository/org/apache/tomcat/tomcat-annotations-api/8.5.23/tomcat-annotations-api-8.5.23.jar:/Users/xyq/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/8.5.23/tomcat-embed-el-8.5.23.jar:/Users/xyq/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/8.5.23/tomcat-embed-websocket-8.5.23.jar:/Users/xyq/.m2/repository/org/hibernate/hibernate-validator/5.3.6.Final/hibernate-validator-5.3.6.Final.jar:/Users/xyq/.m2/repository/javax/validation/validation-api/1.1.0.Final/validation-api-1.1.0.Final.jar:/Users/xyq/.m2/repository/org/jboss/logging/jboss-logging/3.3.1.Final/jboss-logging-3.3.1.Final.jar:/Users/xyq/.m2/repository/com/fasterxml/classmate/1.3.4/classmate-1.3.4.jar:/Users/xyq/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.8.10/jackson-databind-2.8.10.jar:/Users/xyq/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.8.0/jackson-annotations-2.8.0.jar:/Users/xyq/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.8.10/jackson-core-2.8.10.jar:/Users/xyq/.m2/repository/org/springframework/spring-web/4.3.13.RELEASE/spring-web-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-aop/4.3.13.RELEASE/spring-aop-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-webmvc/4.3.13.RELEASE/spring-webmvc-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-expression/4.3.13.RELEASE/spring-expression-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/slf4j/slf4j-api/1.7.25/slf4j-api-1.7.25.jar:/Users/xyq/.m2/repository/junit/junit/4.12/junit-4.12.jar:/Users/xyq/.m2/repository/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3.jar:/Users/xyq/.m2/repository/org/springframework/spring-core/4.3.13.RELEASE/spring-core-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/com/alibaba/dubbo/2.5.8/dubbo-2.5.8.jar:/Users/xyq/.m2/repository/org/springframework/spring-context/4.3.13.RELEASE/spring-context-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-beans/4.3.13.RELEASE/spring-beans-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/javassist/javassist/3.21.0-GA/javassist-3.21.0-GA.jar:/Users/xyq/.m2/repository/org/jboss/netty/netty/3.2.5.Final/netty-3.2.5.Final.jar:/Users/xyq/.m2/repository/com/github/sgroschupf/zkclient/0.1/zkclient-0.1.jar:/Users/xyq/.m2/repository/org/apache/zookeeper/zookeeper/3.3.3/zookeeper-3.3.3.jar:/Users/xyq/.m2/repository/jline/jline/0.9.94/jline-0.9.94.jar:/Users/xyq/.m2/repository/log4j/log4j/1.2.14/log4j-1.2.14.jar:/Users/xyq/workspace/myproject/dubbo-api/target/classes:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.library.path=/Users/xyq/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
   2017-12-28 21:39:57.577  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.io.tmpdir=/var/folders/yq/d0v9dm456zd_xzc1lhllw19m0000gn/T/
   2017-12-28 21:39:57.578  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.compiler=<NA>
   2017-12-28 21:39:57.578  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:os.name=Mac OS X
   2017-12-28 21:39:57.578  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:os.arch=x86_64
   2017-12-28 21:39:57.578  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:os.version=10.13.1
   2017-12-28 21:39:57.578  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:user.name=xyq
   2017-12-28 21:39:57.578  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:user.home=/Users/xyq
   2017-12-28 21:39:57.578  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:user.dir=/Users/xyq/workspace
   2017-12-28 21:39:57.579  INFO 13377 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Initiating client connection, connectString=localhost:2181 sessionTimeout=30000 watcher=org.I0Itec.zkclient.ZkClient@247f74ac
   2017-12-28 21:39:57.589  INFO 13377 --- [or-SendThread()] org.apache.zookeeper.ClientCnxn          : Opening socket connection to server localhost/127.0.0.1:2181
   2017-12-28 21:39:57.600  INFO 13377 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Socket connection established to localhost/127.0.0.1:2181, initiating session
   2017-12-28 21:39:57.613  INFO 13377 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Session establishment complete on server localhost/127.0.0.1:2181, sessionid = 0x1609d33be850005, negotiated timeout = 30000
   2017-12-28 21:39:57.615  INFO 13377 --- [tor-EventThread] org.I0Itec.zkclient.ZkClient             : zookeeper state changed (SyncConnected)
   2017-12-28 21:39:57.618  INFO 13377 --- [           main] c.a.d.r.zookeeper.ZookeeperRegistry      :  [DUBBO] Register: dubbo://10.236.19.51:12345/iservice.IUserService?anyhost=true&application=dubbo-provider&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13377&side=provider&timestamp=1514468397346, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.639  INFO 13377 --- [           main] c.a.d.r.zookeeper.ZookeeperRegistry      :  [DUBBO] Subscribe: provider://10.236.19.51:12345/iservice.IUserService?anyhost=true&application=dubbo-provider&category=configurators&check=false&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13377&side=provider&timestamp=1514468397346, dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.647  INFO 13377 --- [           main] c.a.d.r.zookeeper.ZookeeperRegistry      :  [DUBBO] Notify urls for subscribe url provider://10.236.19.51:12345/iservice.IUserService?anyhost=true&application=dubbo-provider&category=configurators&check=false&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13377&side=provider&timestamp=1514468397346, urls: [empty://10.236.19.51:12345/iservice.IUserService?anyhost=true&application=dubbo-provider&category=configurators&check=false&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13377&side=provider&timestamp=1514468397346], dubbo version: 2.5.8, current host: 127.0.0.1
   2017-12-28 21:39:57.707  INFO 13377 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8888 (http)
   2017-12-28 21:39:57.712  INFO 13377 --- [           main] xyq.DubboProviderApplication             : Started DubboProviderApplication in 3.544 seconds (JVM running for 4.34)
   ```

   可以看到服务提供方在 ZooKeeper 上注册成功了。

## 3. dubbo-consumer

创建服务提供方项目 dubbo-consumer，基于 Spring Boot，并设置依赖 dubbo-api。

1. 服务消费者启动类 `DubboConsumerApplication`

   ```java
   @SpringBootApplication
   @ImportResource("classpath:dubbo-consumer.xml")
   public class DubboConsumerApplication {

   	public static void main(String[] args) {
   		SpringApplication.run(DubboConsumerApplication.class, args);
   	}
   }
   ```

2. 创建控制器 `UserController`

   ```java
   @Controller
   public class UserController {

       @Autowired
       private IUserService userService;

       @RequestMapping("/hello")
       @ResponseBody
       public String sayHello(@RequestParam("name") String name) {
           String welcome = userService.hello(name);
           return welcome;
       }
   }
   ```

3. 在 `application-properties` 里设置服务的 HTTP 端口 `server.port=8889`。

4. 在 classpath 下创建配置文件 `dubbo-consumer.xml`：

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
          xsi:schemaLocation="http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd
          http://code.alibabatech.com/schema/dubbo
          http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
       <dubbo:application name="dubbo-consumer" />
       <dubbo:registry address="zookeeper://localhost:2181"/>
       <dubbo:provider cluster="failfast"/>
       <dubbo:protocol name="dubbo" port="12346" />

       <!-- 引用服务配置 -->
       <dubbo:reference id="userService" interface="iservice.IUserService" cluster="failfast" check="false"/>
   </beans>
   ```

   这个 xml 中定义了：

   - 服务消费者的名称：dubbo-consumer
   - 注册中心的协议及地址：运行在本地 2181 端口上的 ZooKeeper
   - dubbo 服务运行的端口：12346
   - 需要注册的 spring bean：`userService`，并指定它对应了 dubbo 上发布的 `IUserService` 这个服务接口。

5. 启动项目。

   ```
   2017-12-28 21:42:43.113  INFO 13482 --- [           main] xyq.DubboConsumerApplication             : Starting DubboConsumerApplication on bogon with PID 13482 (/Users/xyq/workspace/myproject/dubbo-consumer/target/classes started by xyq in /Users/xyq/workspace)
   2017-12-28 21:42:43.115  INFO 13482 --- [           main] xyq.DubboConsumerApplication             : No active profile set, falling back to default profiles: default
   2017-12-28 21:42:43.177  INFO 13482 --- [           main] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@2d127a61: startup date [Thu Dec 28 21:42:43 CST 2017]; root of context hierarchy
   2017-12-28 21:42:43.860  INFO 13482 --- [           main] o.s.b.f.xml.XmlBeanDefinitionReader      : Loading XML bean definitions from class path resource [dubbo-consumer.xml]
   2017-12-28 21:42:44.114  INFO 13482 --- [           main] c.a.dubbo.common.logger.LoggerFactory    : using logger: com.alibaba.dubbo.common.logger.log4j.Log4jLoggerAdapter
   2017-12-28 21:42:44.920  INFO 13482 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'dubbo-consumer' of type [com.alibaba.dubbo.config.ApplicationConfig] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
   2017-12-28 21:42:44.930  INFO 13482 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'com.alibaba.dubbo.config.RegistryConfig' of type [com.alibaba.dubbo.config.RegistryConfig] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
   2017-12-28 21:42:44.931  INFO 13482 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'userService' of type [com.alibaba.dubbo.config.spring.ReferenceBean] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
   2017-12-28 21:42:45.432  INFO 13482 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8889 (http)
   2017-12-28 21:42:45.444  INFO 13482 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
   2017-12-28 21:42:45.446  INFO 13482 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.23
   2017-12-28 21:42:45.541  INFO 13482 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
   2017-12-28 21:42:45.541  INFO 13482 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 2368 ms
   2017-12-28 21:42:45.658  INFO 13482 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'dispatcherServlet' to [/]
   2017-12-28 21:42:45.661  INFO 13482 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
   2017-12-28 21:42:45.662  INFO 13482 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
   2017-12-28 21:42:45.662  INFO 13482 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
   2017-12-28 21:42:45.662  INFO 13482 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
   2017-12-28 21:42:45.774  INFO 13482 --- [           main] c.a.d.common.concurrent.ExecutionList    :  [DUBBO] Executor for listenablefuture is null, will use default executor!, dubbo version: 2.5.8, current host: 10.236.19.51
   2017-12-28 21:42:45.780  INFO 13482 --- [-localhost:2181] org.I0Itec.zkclient.ZkEventThread        : Starting ZkClient event thread.
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:zookeeper.version=3.3.3-1073969, built on 02/23/2011 22:27 GMT
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:host.name=bogon
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.version=1.8.0_91
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.vendor=Oracle Corporation
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.class.path=/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/lib/tools.jar:/Users/xyq/workspace/myproject/dubbo-consumer/target/classes:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter-web/1.5.9.RELEASE/spring-boot-starter-web-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter/1.5.9.RELEASE/spring-boot-starter-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot/1.5.9.RELEASE/spring-boot-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/1.5.9.RELEASE/spring-boot-autoconfigure-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter-logging/1.5.9.RELEASE/spring-boot-starter-logging-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/ch/qos/logback/logback-classic/1.1.11/logback-classic-1.1.11.jar:/Users/xyq/.m2/repository/ch/qos/logback/logback-core/1.1.11/logback-core-1.1.11.jar:/Users/xyq/.m2/repository/org/slf4j/jcl-over-slf4j/1.7.25/jcl-over-slf4j-1.7.25.jar:/Users/xyq/.m2/repository/org/slf4j/jul-to-slf4j/1.7.25/jul-to-slf4j-1.7.25.jar:/Users/xyq/.m2/repository/org/slf4j/log4j-over-slf4j/1.7.25/log4j-over-slf4j-1.7.25.jar:/Users/xyq/.m2/repository/org/yaml/snakeyaml/1.17/snakeyaml-1.17.jar:/Users/xyq/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/1.5.9.RELEASE/spring-boot-starter-tomcat-1.5.9.RELEASE.jar:/Users/xyq/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/8.5.23/tomcat-embed-core-8.5.23.jar:/Users/xyq/.m2/repository/org/apache/tomcat/tomcat-annotations-api/8.5.23/tomcat-annotations-api-8.5.23.jar:/Users/xyq/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/8.5.23/tomcat-embed-el-8.5.23.jar:/Users/xyq/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/8.5.23/tomcat-embed-websocket-8.5.23.jar:/Users/xyq/.m2/repository/org/hibernate/hibernate-validator/5.3.6.Final/hibernate-validator-5.3.6.Final.jar:/Users/xyq/.m2/repository/javax/validation/validation-api/1.1.0.Final/validation-api-1.1.0.Final.jar:/Users/xyq/.m2/repository/org/jboss/logging/jboss-logging/3.3.1.Final/jboss-logging-3.3.1.Final.jar:/Users/xyq/.m2/repository/com/fasterxml/classmate/1.3.4/classmate-1.3.4.jar:/Users/xyq/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.8.10/jackson-databind-2.8.10.jar:/Users/xyq/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.8.0/jackson-annotations-2.8.0.jar:/Users/xyq/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.8.10/jackson-core-2.8.10.jar:/Users/xyq/.m2/repository/org/springframework/spring-web/4.3.13.RELEASE/spring-web-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-aop/4.3.13.RELEASE/spring-aop-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-webmvc/4.3.13.RELEASE/spring-webmvc-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-expression/4.3.13.RELEASE/spring-expression-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/slf4j/slf4j-api/1.7.25/slf4j-api-1.7.25.jar:/Users/xyq/.m2/repository/junit/junit/4.12/junit-4.12.jar:/Users/xyq/.m2/repository/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3.jar:/Users/xyq/.m2/repository/org/springframework/spring-core/4.3.13.RELEASE/spring-core-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/com/alibaba/dubbo/2.5.8/dubbo-2.5.8.jar:/Users/xyq/.m2/repository/org/springframework/spring-context/4.3.13.RELEASE/spring-context-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/springframework/spring-beans/4.3.13.RELEASE/spring-beans-4.3.13.RELEASE.jar:/Users/xyq/.m2/repository/org/javassist/javassist/3.21.0-GA/javassist-3.21.0-GA.jar:/Users/xyq/.m2/repository/org/jboss/netty/netty/3.2.5.Final/netty-3.2.5.Final.jar:/Users/xyq/.m2/repository/com/github/sgroschupf/zkclient/0.1/zkclient-0.1.jar:/Users/xyq/.m2/repository/org/apache/zookeeper/zookeeper/3.3.3/zookeeper-3.3.3.jar:/Users/xyq/.m2/repository/jline/jline/0.9.94/jline-0.9.94.jar:/Users/xyq/.m2/repository/log4j/log4j/1.2.14/log4j-1.2.14.jar:/Users/xyq/workspace/myproject/dubbo-api/target/classes:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.library.path=/Users/xyq/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
   2017-12-28 21:42:45.785  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.io.tmpdir=/var/folders/yq/d0v9dm456zd_xzc1lhllw19m0000gn/T/
   2017-12-28 21:42:45.787  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:java.compiler=<NA>
   2017-12-28 21:42:45.787  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:os.name=Mac OS X
   2017-12-28 21:42:45.787  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:os.arch=x86_64
   2017-12-28 21:42:45.787  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:os.version=10.13.1
   2017-12-28 21:42:45.787  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:user.name=xyq
   2017-12-28 21:42:45.787  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:user.home=/Users/xyq
   2017-12-28 21:42:45.787  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Client environment:user.dir=/Users/xyq/workspace
   2017-12-28 21:42:45.788  INFO 13482 --- [clientConnector] org.apache.zookeeper.ZooKeeper           : Initiating client connection, connectString=localhost:2181 sessionTimeout=30000 watcher=org.I0Itec.zkclient.ZkClient@5cee7de6
   2017-12-28 21:42:45.798  INFO 13482 --- [or-SendThread()] org.apache.zookeeper.ClientCnxn          : Opening socket connection to server localhost/0:0:0:0:0:0:0:1:2181
   2017-12-28 21:42:45.818  INFO 13482 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Socket connection established to localhost/0:0:0:0:0:0:0:1:2181, initiating session
   2017-12-28 21:42:45.825  INFO 13482 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Session establishment complete on server localhost/0:0:0:0:0:0:0:1:2181, sessionid = 0x1609d33be850006, negotiated timeout = 30000
   2017-12-28 21:42:45.826  INFO 13482 --- [tor-EventThread] org.I0Itec.zkclient.ZkClient             : zookeeper state changed (SyncConnected)
   2017-12-28 21:42:45.846  INFO 13482 --- [           main] c.a.d.r.zookeeper.ZookeeperRegistry      :  [DUBBO] Register: consumer://10.236.19.51/iservice.IUserService?application=dubbo-consumer&category=consumers&check=false&cluster=failfast&dubbo=2.5.8&interface=iservice.IUserService&methods=hello&pid=13482&side=consumer&timestamp=1514468565714, dubbo version: 2.5.8, current host: 10.236.19.51
   2017-12-28 21:42:45.859  INFO 13482 --- [           main] c.a.d.r.zookeeper.ZookeeperRegistry      :  [DUBBO] Subscribe: consumer://10.236.19.51/iservice.IUserService?application=dubbo-consumer&category=providers,configurators,routers&check=false&cluster=failfast&dubbo=2.5.8&interface=iservice.IUserService&methods=hello&pid=13482&side=consumer&timestamp=1514468565714, dubbo version: 2.5.8, current host: 10.236.19.51
   2017-12-28 21:42:45.871  INFO 13482 --- [           main] c.a.d.r.zookeeper.ZookeeperRegistry      :  [DUBBO] Notify urls for subscribe url consumer://10.236.19.51/iservice.IUserService?application=dubbo-consumer&category=providers,configurators,routers&check=false&cluster=failfast&dubbo=2.5.8&interface=iservice.IUserService&methods=hello&pid=13482&side=consumer&timestamp=1514468565714, urls: [dubbo://10.236.19.51:12345/iservice.IUserService?anyhost=true&application=dubbo-provider&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13377&side=provider&timestamp=1514468397346, empty://10.236.19.51/iservice.IUserService?application=dubbo-consumer&category=configurators&check=false&cluster=failfast&dubbo=2.5.8&interface=iservice.IUserService&methods=hello&pid=13482&side=consumer&timestamp=1514468565714, empty://10.236.19.51/iservice.IUserService?application=dubbo-consumer&category=routers&check=false&cluster=failfast&dubbo=2.5.8&interface=iservice.IUserService&methods=hello&pid=13482&side=consumer&timestamp=1514468565714], dubbo version: 2.5.8, current host: 10.236.19.51
   2017-12-28 21:42:45.966  INFO 13482 --- [           main] c.a.d.remoting.transport.AbstractClient  :  [DUBBO] Successed connect to server /10.236.19.51:12345 from NettyClient 10.236.19.51 using dubbo version 2.5.8, channel is NettyChannel [channel=[id: 0x3f3c966c, /10.236.19.51:61111 => /10.236.19.51:12345]], dubbo version: 2.5.8, current host: 10.236.19.51
   2017-12-28 21:42:45.966  INFO 13482 --- [           main] c.a.d.remoting.transport.AbstractClient  :  [DUBBO] Start NettyClient bogon/10.236.19.51 connect to the server /10.236.19.51:12345, dubbo version: 2.5.8, current host: 10.236.19.51
   2017-12-28 21:42:45.998  INFO 13482 --- [           main] com.alibaba.dubbo.config.AbstractConfig  :  [DUBBO] Refer dubbo service iservice.IUserService from url zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?anyhost=true&application=dubbo-consumer&check=false&cluster=failfast&default.cluster=failfast&dubbo=2.5.8&generic=false&interface=iservice.IUserService&methods=hello&pid=13482&register.ip=10.236.19.51&remote.timestamp=1514468397346&side=consumer&timestamp=1514468565714, dubbo version: 2.5.8, current host: 10.236.19.51
   2017-12-28 21:42:46.218  INFO 13482 --- [           main] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@2d127a61: startup date [Thu Dec 28 21:42:43 CST 2017]; root of context hierarchy
   2017-12-28 21:42:46.268  INFO 13482 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/hello]}" onto public java.lang.String xyq.UserController.sayHello(java.lang.String)
   2017-12-28 21:42:46.270  INFO 13482 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
   2017-12-28 21:42:46.270  INFO 13482 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
   2017-12-28 21:42:46.294  INFO 13482 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
   2017-12-28 21:42:46.294  INFO 13482 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
   2017-12-28 21:42:46.323  INFO 13482 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
   2017-12-28 21:42:46.430  INFO 13482 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
   2017-12-28 21:42:46.472  INFO 13482 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8889 (http)
   2017-12-28 21:42:46.476  INFO 13482 --- [           main] xyq.DubboConsumerApplication             : Started DubboConsumerApplication in 3.634 seconds (JVM running for 3.99)
   ```

## 4. 测试

浏览器中打开 http://localhost:8889/hello?name=gilgamesh ，可以看见回复：hello gilgamesh!

## 5. 注意事项

项目中需要添加 ZkClient 的依赖：

```xml
		<dependency>
			<groupId>com.github.sgroschupf</groupId>
			<artifactId>zkclient</artifactId>
			<version>0.1</version>
		</dependency>
```

