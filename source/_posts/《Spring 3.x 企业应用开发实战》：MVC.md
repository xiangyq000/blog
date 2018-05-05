---
title: 《Spring 3.x 企业应用开发实战》：MVC
date: 2016-03-26 15:53:38 +0800
categories: Spring
tags: [Spring, 读书笔记]
---
### 1. 处理流程
- 请求提交给 `DispatchServlet`
- 查找 `HandlerMapping`
- 调用由 `HandlerAdapter` 封装后的 `Handler`
- 返回 `ModelAndView` 到 `DispatcherServlet`
- 借由 `ViewResolver` 完成逻辑视图到真实视图的转换
- 返回响应

<!-- more -->

### 2. 配置 DispatcherServlet
- `web.xml`

```xml
<web-app>
    <!-- 父容器配置 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            /WEB-INF/applicationServlet.xml,
        </param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- 子容器配置 -->
    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>logLevel</param-name>
            <param-value>FINE</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!-- url映射 -->
    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/api/*</url-pattern>
    </servlet-mapping>
</web-app>
```
- 子容器可以访问父容器的 bean，父容器不能访问子容器的 bean
- 默认采用 `org.springframework.web.context.ContextLoaderListener`
- 默认使用 `/WEB-INF/{servlet-name}-*.xml` 加载子容器

### 3. DispatcherServlet 的初始化
- `DispatcherServlet` 初始化

```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```
- 默认配置： `DispatcherServlet.properties`

```xml
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

### 4. Controller 注解
#### 4.1 类注解
- `@Controller` （由 Spring 识别 `Handler` 实例）
- `@RequestMapping(value=, method=, params=)`

#### 4.2 方法注解
- `@RequestMapping(value=, method=, params=)`
- `@PathVariable`，配合占位符 `{parameter}`
- `@RequestParam(value=, required=, defaultValue=)`
- `@CookieValue(value=, required=, defaultValue=)`
- `@RequestHeader(value=, required=, defaultValue=)`

### 5. 封装入参
- `HttpServletRequest`，`WebRequest`，`ServletRequest` 的 `InputStream` / `Reader`
- `HttpServeltResponse`，`ServletResponse的OutputStream` / `Writer`

### 6. 请求信息和对象的转换
#### 6.1 基本
- `HttpMessageConverter<T>`
- 使用 `@RequestBody` / `@ResponseBody`；使用 `HttpEntity<T>` / `ResponseEntity<T>`
- Spring 根据 HTTP 报文头部的 `Accept` 指定的 MIME 类型，查找匹配的 `HttpMessageConverter`
- Spring MVC 默认装配 `AnnotationMethodHandlerAdapter`，调用 `HttpMessageConverter`
- MVC 命名空间的 `<mvc:annotation-driven/>` 标签会创建并注册一个默认的 `DefaultAnnotationHandlerMapping` 和一个 `AnnotationMethodHandlerAdapter` 实例，如果上下文中存在自定义的对应组件 bean，则覆盖默认配置

#### 6.2 样例
- `app-servlet.xml`

```xml
<bean id="handlerMapping" class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping">
    <property name="alwaysUseFullPath" value="true"/>
    <property name="interceptors">
        <list>
            <ref bean="monitorInterceptor"/>
            <ref bean="commonOutLogInterceptor"/>
            <ref bean="serverTraceInterceptor"/>
        </list>        
    </property>
</bean>

<bean id="handlerAdapter" class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
    <property name="messageConverters">
	    <list>
            <bean class="org.springframework.http.converter.ByteArrayHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.StringHttpMessageConverter">
                <property name="writeAcceptCharset" value="false"/>
            </bean>
            <bean class="org.springframework.http.converter.xml.SourceHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.xml.XmlAwareFormHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
                <property name="supportedMediaTypes">
                    <list>
                        <bean class="org.springframework.http.MediaType">
                            <constructor-arg value="text"/>
                            <constructor-arg value="json"/>
                            <constructor-arg>
                                <map>  
                                    <entry key="charset" value="UTF-8" />  
                                </map>
                            </constructor-arg>
                        </bean>
                    </list>
                </property>
            </bean>
        </list>
    </property>
</bean>
```

### 7. 处理模型数据
- Spring MVC 在调用方法前会创建一个隐含的模型对象
- `ModelAndView`：返回值类型为 `ModelAndView` 时，方法体通过该对象添加模型数据
- `@ModelAttribute`：入参对象添加到数据模型中
- Map 和 Model：入参为 `org.springframework.ui.Model`、`org.springframework.ui.ModelMap`或 `java.util.Map` 时，处理方法返回时，Map 中的数据会自动添加到模型中
- `@SessionAttributes`：将模型中的某个属性暂存到 `HttpSession` 中，以便多个请求共享

### 8. 数据绑定
#### 8.1 基本
- Spring MVC 将 `ServletRequest` 对象及处理方法的入参实例传递给 `DataBinder`；
- `DataBinder `调用 `ConversionService` 组件进行数据类型转换、数据格式化等工作，将 `ServletRequest` 中的消息填充到入参对象中；
- 然后再调用 `Validator` 组件对已经绑定了请求消息数据的入参对象进行数据合法性校验，并最终生成绑定结果 `BindingResult` 对象，其中还包含相应的校验错误对象
- 抽取 `BindingResult` 中的入参对象及校验错误对象，赋给处理方法的相应入参

#### 8.2 数据转换
- 例如：将请求信息中A类型参数转换并绑定到 Controller 对应的处理方法的 B 类型入参中
- 基于 `ConversionService` 接口，Spring 将自动识别其实现类，用于入参类型转换，类似于 C++ 中的自定义类型转换。例如，将 HTTP 请求信息中的字符串格式参数转换为 Controller 对应方法中的类类型入参
- 可通过 `ConversionServiceFactoryBean` 的 `converters` 属性注册自定义转换器。可接受 `Converter`，`ConverterFacotory`，`GenericConverter `或 `ConditionalConverterFactory` 的实现类，并统一封装到一个 `ConversionService` 的实例（即 `GenericConversionService`）中

```xml
<bean id="conversionService" class="org.springframework.context.support.ConversionServiceFactoryBean">
    <property name="converters">
        <list>
            <bean class="MyConverters" />
        </list>
    </property>
</bean>
```
- `<mvc:annotation-driven/> `标签还会注册一个默认的 `ConversionService`，即 `FormattingConversionServiceFactoryBean`

#### 8.3 数据格式化
- 例如：将请求信息中字符串类型参数转换并绑定到 Controller 对应的处理方法入参中的 Date 类型属性中
- 格式化框架定义了 `Formatter<T>` 接口，扩展于 `Printer<T>` 和 `Parser<T>` 接口
- Srping 提供 `AnnotationFormatterFactory<A extends Annotation>` 接口及两个实现类：`NumberFormatAnnotationFormatterFactory `和 `JodaDateTimeFormatAnnotationFormatterFactory`
- 在入参类属性上使用注解：`@DateTimeFormat`，`@NumberFormat`
- Spring 通过 `<mvc:annotation-driven/>` 标签创建 `FormattingConversionServiceFactoryBean` 作为 `FormattingConversionService` 的实例，自动装配 `NumberFormatAnnotationFormatterFactory` 和 `JodaDateTimeFormatAnnotationFormatterFactory`


#### 8.4 数据验证
- `<mvc:annotation-driven/> `标签会默认装配 `LocalValidatorFactoryBean`，实现了 Spring 的`Validator` 接口，通过在入参上标注 `@Valid` 注解即可让 Spring 在完成数据绑定后执行数据校验
- `@Valid `注解标注的入参和其后的 `BindingResult` 或 `Errors` 入参成对出现，后者保存前者的校验结果
- 校验结果也保存在 MVC 的隐含模型中
- 样例

```java
public Class User {
    private String userId;

    @Pattern(regexp="w{4,30}")
    private String userName;

    @Length(min=2, max=100)
    private String nickName;

    @Past
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private Date birthday;

    @DecimalMin(value = "1000.00")
    @DecimalMax(value = "10000.00")
    @NumberFormat(pattern = "#,###.##")
    pirvate Long salary;
}


@RequestMapping(value = "/handle")
public void handle(@Valid User uer, BindingResult bindingResult) {
    //do something
}
```


### 9. 视图解析
#### 9.1 基本
- 视图对象是一个 bean，由视图解析器负责实例化。
- 不同的视图实现技术对应于不同的 `View` 实现类
- 所有的视图解析器都实现了 `ViewResolver` 接口

#### 9.2 样例
- `app-servlet.xml`

```xml
<!-- For multipart encoding support -->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <property name="maxUploadSize" value="209715200"/>
</bean>

<bean id="beanNameResolver" class="org.springframework.web.servlet.view.BeanNameViewResolver"/>

<!--VM 模板文件解析 -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.velocity.VelocityViewResolver">
    <property name="cache" value="true" />
    <property name="prefix" value="" />
    <property name="suffix" value=".vm" />
    <property name="contentType">
        <value>text/html; charset=UTF-8</value>
    </property>
    <property name="toolboxConfigLocation" value="/WEB-INF/toolbox.xml"/>
</bean>

<bean id="velocityConfig" class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
    <property name="resourceLoaderPath" value="/WEB-INF/vm/" />
    <property name="velocityPropertiesMap">
        <props>
            <prop key="input.encoding">UTF-8</prop>
            <prop key="output.encoding">UTF-8</prop>
            <prop key="velocimacro.library"></prop>
            <prop key="velocimacro.library.autoreload">true</prop>
            <prop key="directive.foreach.counter.name">loopCursor</prop>
            <prop key="directive.foreach.counter.initial.value">0</prop>
        </props>
    </property>
</bean>
```

### 10. 静态资源处理
- 需要对项目中的图片、html 静态资源等单独处理
- 配置 `<mvc:default-servlet-handler>` 后，会装配一个 `DefaultServletHttpRequestHandler`，对静态资源做检查
- 配置 `<mvc:resources/>`，允许对静态资源文件路径作映射，并提供文件在浏览器端的缓存控制，例如

```xml
<mvc:resources mapping="/resources/**" location="/,classpath:/META-INF/publicResources" />
```

### 11. 拦截器
#### 11.1 基本
- `DispatcherServlet `将请求交给处理器映射（`HandlerMapping`），找到对应的 `HandlerExecutionChain`
- 找到对应的 `HandlerExecutionChain` 包含若干 `HandlerInterceptor`，和一个 `Handler`
- `HandlerInterceptor` 接口：

```java
public interface HandlerInterceptor {

  boolean preHandle (HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

  void postHandle (HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception;

  void afterCompletion (HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception;
}
```
#### 11.2 样例
- `app-servlet.xml`:

```xml
<mvc:interceptors>
    <mvc:interceptor>
      <mvc:mapping path="/api/**" />
      <bean class="outfox.ynote.webserver.web.HttpsInterceptor" />
    </mvc:interceptor>
</mvc:interceptors>
```

### 12. 异常处理
#### 12.1 基本
- Spring MVC 通过 `HandlerExceptionResolver` 处理异常
- `HandlerExceptionResolver` 接口：

```java
public interface HandlerExceptionResolver {
  ModelAndView resolveException (HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);
}
```
- Spring MVC 默认装配了 `DefaultHandlerExceptionResolver`
- Spring MVC 默认注册了 `AnnotationMethodHandlerExceptionResolver`，允许通过 `@ExceptionHandler` 指定处理特定异常

#### 12.2 样例
- `web.xml`

```xml
<web-app>
    <!-- 对各种异常和错误，交由 ErrorController 处理 -->
    <error-page>
        <error-code>404</error-code>
        <location>/error</location>
    </error-page>
    <error-page>
        <error-code>403</error-code>
        <location>/error</location>
    </error-page>
    <error-page>
        <exception-type>java.lang.Throwable</exception-type>
        <location>/error</location>
    </error-page>
</web-app>
```
