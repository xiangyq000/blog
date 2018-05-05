---
title: Spring IoC
date: 2016-02-10 15:53:42 +0800
categories: Spring
tags: Spring
---
### 1. Spring的资源抽象接口
假如有一个文件位于Web应用的类路径下，用户可以通过以下方式对这个文件资源进行访问：
- 通过`FileSystemResource`以文件系统绝对路径的方式进行访问；
- 通过`ClassPathResource`以类路径的方式进行访问；
- 通过`ServletContextResource`以相对于Web应用根目录的方式进行访问。

<!-- more -->

### 2. BeanFactory的类体系结构
![image](http://dl.iteye.com/upload/picture/pic/78494/d13e3946-c5d2-352b-89d0-0ae1507b78f3.jpg)

- `BeanFactory`：位于类结构树的顶端，最主要的方法是`getBean(String beanName)`，从容器中返回特定类型的bean
- `ListableBeanFactory`：该接口定义了访问容器中Bean基本信息的若干方法
- `HierarchicalBeanFactory`：父子级联IoC容器的接口，子容器可以通过接口方法访问父容器
- `ConfigurableBeanFactory`：增强了IoC容器的可定制性
- `AutowireCapableBeanFactory`：定义了将容器中的bean按照某种规则进行自动装配的方法
- `SingletonBeanRegistry`：定义了允许在运行期向容器注册单实例bean的方法
- `BeanDefinitionRegistry`：每一个bean在容器中通过`BeanDefinition`对象表示，`BeanDefinitionRegistry`定义了向容器手工注册bean的方法
- Spring在`DefaultSingletonBeanRegistry`类中提供了一个用于缓存单实例bean的缓存器，以`HashMap`实现，单实例的bean以`beanName`为key保存在这个`HashMap`中

### 3. ApplicationContext的类体系结构
![image](http://dl.iteye.com/upload/attachment/0071/0968/f885078e-bc34-385e-9d8d-0ee61eb15c11.jpg)

- `ApplicationEventPublisher`：让容器拥有发布应用上下文事件的功能，包括容器启动事件、关闭事件等。实现了`ApplicationListener`事件监听接口的Bean 可以接收到容器事件，并对事件进行响应处理。在`ApplicationContext`抽象实现类`AbstractApplicationContext`中，我们可以发现存在一个`ApplicationEventMulticaster`，它负责保存所有监听器，以便在容器产生上下文事件时通知这些事件监听者。
- `MessageSource`：为应用提供i18n国际化消息访问的功能；
- `ResourcePatternResolver`：所有`ApplicationContext`实现类都实现了类似于`PathMatchingResourcePatternResolver`的功能，可以通过带前缀的Ant风格的资源文件路径装载Spring的配置文件。
- `LifeCycle`：该接口是Spring 2.0加入的，该接口提供了`start()`和`stop()`两个方法，主要用于控制异步处理过程。在具体使用时，该接口同时被`ApplicationContext`实现及具体Bean实现，`ApplicationContext`会将`start/stop`的信息传递给容器中所有实现了该接口的Bean，以达到管理和控制JMX、任务调度等目的。
- `ConfigurableApplicationContext`扩展于`ApplicationContext`，它新增加了两个主要的方法：`refresh()`和`close()`，让`ApplicationContext`具有启动、刷新和关闭应用上下文的能力。在应用上下文关闭的情况下调用`refresh()`即可启动应用上下文，在已经启动的状态下，调用`refresh()`则清除缓存并重新装载配置信息，而调用`close()`则可关闭应用上下文。

### 4. WebApplicantContext体系结构
- 它允许从相对于Web根目录的路径中加载配置文件完成初始化工作。从`WebApplicationContext`中可以获取`ServletContext`引用，整个Web应用上下文对象将作为属性放置在`ServletContext`中，以便Web应用环境可以访问spring上下文。
- `WebApplicationContext`扩展了`ApplicationContext`，`WebApplicationContext`定义了一个常量`ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`，在上下文启动时，我们可以直接通过下面的语句从web容器中获取`WebApplicationContext`:

```
WebApplicationContext wac=(WebApplicationContext)servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
```

### 5. BeanFactory中Bean的生命周期
#### 5.1 Bean生命周期
1. 如果容器注册`InstantiationAwareBeanPostProcessor`接口，调用`postProcessBeforeInstantiation`方法
2. Bean的实例化(调用默认构造器)
3. 如果容器注册`InstantiationAwareBeanPostProcessor`接口，调用`postProcessAfterInstantiation`方法
4. 如果容器注册`InstantiationAwareBeanPostProcessor`接口，调用`postProcessPropertyValues`方法
5. 根据配置设置属性值
6. 如果Bean实现了`BeanNameAware`接口，调用`BeanNameAware`接口的`setBeanName`方法
7. 如果Bean实现了`BeanFactoryAware`接口，调用`BeanFactoryAware`接口的`setBeanFactory`方法
8. 如果容器注册了`BeanPostProcessor`接口，调用`BeanPostProcessor`接口的`postProcessBeforeInitialization`方法
9. 如果Bean实现了`InitializingBean`接口，调用`InitializingBean`接口的`afterPropertiesSet`方法
10. 通过`init-method`属性配置的初始方法
11. 如果容器注册了`BeanPostProcessor`接口，调用`BeanPostProcessor`接口的`postProcessAfterInitialization`方法
12. 如果是单例模式，将Bean放入缓存池中；容器销毁时，调用`DisposableBean的destroy`方法；最后调用`destroy-method`方法
13. 如果是多例模式，将Bean交给调用者。

#### 5.2 初始化过程中的方法分类
- bean自身的方法：如调用bean构造函数实例化bean，调用`Setter`设置bean的属性值，以及通过`<bean>`的`init-method`和`destory-method`所指定的方法；
- bean级生命周期接口方法：如`BeanNameAware`、`BeanFactoryAware`、`InitializingBean`和`DisposableBean`，这些接口方法由bean类直接实现；
- 容器级生命周期接口方法：如`InstantiationAwareBeanPostProcessor`和 `BeanPostProcessor`这两个接口实现，一般称它们的实现类为“后处理器”。

#### 5.3 说明
- Spring的AOP等功能即通过`BeanPostProcessor`实现
- 如果`<bean>`通过`init-method`属性定义了初始化方法，将执行这个方法
- 如果bean的作用范围为`scope="prototype"`，将bean返回给调用者之后，调用者负责bean的后续生命的管理，Spring不再管理这个bean的生命周期；如果`scope="singleton"`，则将bean放入到Spring IoC容器的缓存池中，并将bean的引用返回给调用者，Spring继续管理这些bean的后续生命周期
- 对于单例的bean，当容器关闭时，将触发Spring对bean的后续生命周期的管理工作。如果bean实现了`DisposableBean`接口，则将调用接口的`destroy()`方法
- 对于单例的bean，如果通过`destroy-method`指定了bean的销毁方法，Spring将执行这个方法
- 后处理器的实际调用顺序和注册顺序无关，在具有多个后处理器的情况下，必须通过实现`org.springframework.core.Ordered`接口确定调用顺序

#### 5.4 测试
- `applicationContext.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">

    <bean id="car" class="com.data.Car"
        p:color="color"
        init-method="init"
        destroy-method="destroy2" />
</beans>
```

- `Car`

```java
package com.data;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;

public class Car implements BeanNameAware, BeanFactoryAware,
    InitializingBean, DisposableBean {

    public Car() {
        System.out.println("construct car");
    }

    private String name;

    private String color;

    private BeanFactory beanFactory;

    private String beanName;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
        System.out.println("set color=" + color);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("after properties set method");
    }

    public void init() {
        System.out.println("init method");
    }

    @Override
    public void destroy() {
        System.out.println("destroy method");
    }

    public void destroy2() {
        System.out.println("my destroy method");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("set bean factory");
        this.beanFactory = beanFactory;
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("set bean name");
        this.beanName = name;
    }

    public BeanFactory getBeanFactory() {
        return beanFactory;
    }

    public String getBeanName() {
        return beanName;
    }
}
```

- `MyBeanPostProcessor`

```java
package com.beanfactory;
import java.beans.PropertyDescriptor;
import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessorAdapter;

public class MyBeanPostProcessor
    extends InstantiationAwareBeanPostProcessorAdapter{

    public MyBeanPostProcessor() {
        System.out.println("construct MyBeanPostProcessor");
    }

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        System.out.println("post process before instantiation");
        return null;
    }

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        System.out.println("post process after instantiation");
        return true;
    }

    @Override
    public PropertyValues postProcessPropertyValues(
    PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        System.out.println("post process property values");
        return pvs;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("post process before initialization");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("post process after initialization");
        return bean;
    }
}
```

- `TestBeanFactory`

```java
package config;

import org.springframework.beans.factory.xml.XmlBeanFactory;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;

import com.beanfactory.MyBeanPostProcessor;
import com.data.Car;

@SuppressWarnings("deprecation")
public class TestBeanFactory {

    public static void main(String[] args) {
        Resource res = new ClassPathResource(
                "/applicationContext.xml");
        XmlBeanFactory bf = new XmlBeanFactory(res);
        bf.addBeanPostProcessor(new MyBeanPostProcessor());
        System.out.println("bean factory initialization done");
        Car car1 = bf.getBean("car", Car.class);
        Car car2 = bf.getBean("car", Car.class);
        System.out.println("(car1 == car2) = " + (car1 == car2));
        System.out.println("get color=" + car1.getColor());
        bf.destroySingletons();
    }
}
```

- 结果

```
二月 09, 2017 10:58:59 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [applicationContext.xml]
construct MyBeanPostProcessor
bean factory initialization done
post process before instantiation
construct car
post process after instantiation
post process property values
set color=color
set bean name
set bean factory
post process before initialization
after properties set method
init method
post process after initialization
(car1 == car2) = true
get color=color
destroy method
my destroy method
```

### 6. ApplicationContext中的Bean生命周期
#### 6.1 流程图
![image](http://img.blog.csdn.net/20140420140946843)

#### 6.2 说明
- 如果bean实现了`org.springframework.context.ApplicationContextAware`接口，会增加一个调用该接口方法`setApplicationContext()`的步骤
- 如果配置文件中声明了工厂后处理器接口`BeanFactoryPostProcessor`的实现类，则应用上下文在加载配置文件之后、初始化bean实例之前将调用这些`BeanFactoryPostProcessor`对配置信息进行加工处理
- `ApplicationContext`和`BeanFactory`的不同之处在于：前者会利用Java反射机制自动识别出配置文件中定义的`BeanPostProcessor`、`InstantiationAwareBeanPostProcessor`和`BeanFactoryPostProcessor`，并自动将它们注册到应用上下文中；而后者需要在代码中通过手工调用`addBeanPostProcessor()`方法进行注册
- 对bean的初始化，`BeanFactory`发生在第一次调用bean时，而`ApplicationContext`发生在初始化容器时

#### 6.3 测试
- `MyBeanPostProcessor`同上
- `Car`增加对`ApplicationContextAware`接口的实现，并添加`@PostConstruct`和`@PreDestroy`的注解方法

```java
package com.data;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class Car implements BeanNameAware, BeanFactoryAware,
    InitializingBean, DisposableBean, ApplicationContextAware {

    public Car() {
        System.out.println("construct car");
    }

    private String name;

    private String color;

    private BeanFactory beanFactory;

    private String beanName;

    private ApplicationContext ctx;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("after properties set method");
    }

    public void init() {
        System.out.println("init method");
    }

    @Override
    public void destroy() {
        System.out.println("destroy method");
    }

    public void destroy2() {
        System.out.println("my destroy method");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("set bean factory");
        this.beanFactory = beanFactory;
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("set bean name");
        this.beanName = name;
    }

    public BeanFactory getBeanFactory() {
        return beanFactory;
    }

    public String getBeanName() {
        return beanName;
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("post construct");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("pre destroy");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext)
            throws BeansException {
        System.out.println("set application context");
        this.ctx = applicationContext;
    }

    public ApplicationContext getApplicationContext() {
        return ctx;
    }

}
```

- `MyBeanFactoryPostProcessor`

```java
package com.beanfactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;

public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    public MyBeanFactoryPostProcessor() {
        System.out.println("construct MyBeanFactoryPostProcessor");
    }

    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("post process bean factory");
    }
}
```

- 基于Java类的Spring配置：`AnnotationBeans`

```java
package config;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import com.beanfactory.MyBeanFactoryPostProcessor;
import com.beanfactory.MyBeanPostProcessor;
import com.data.Car;

@Configuration
public class AnnotationBeans {

    @Bean(name = "car", initMethod = "init", destroyMethod = "destroy2")
    public Car getCar() {
        Car car = new Car();
        car.setColor("color");
        return car;
    }

    @Bean(name = "myBeanPostProcessor")
    public MyBeanPostProcessor getMyBeanPostProcessor() {
        return new MyBeanPostProcessor();
    }

    @Bean(name = "myBeanFactoryPostProcessor")
    public MyBeanFactoryPostProcessor getMyBeanFactoryPostProcessor() {
        return new MyBeanFactoryPostProcessor();
    }
}
```

- `TestApplicationContext`

```java
package config;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import com.data.Car;

public class TestApplicationContext {

    public static void main(String[] args) {
        /*对于以xml形式初始化的ctx，也可以用ClassPathXmlApplicationContext
        或者FileSystemXmlApplicationContext*/
        AnnotationConfigApplicationContext ctx =
                new AnnotationConfigApplicationContext(
                        AnnotationBeans.class);
        System.out.println("application context done");
        Car car = ctx.getBean("car", Car.class);
        System.out.println("get color=" + car.getColor());
        ctx.close();
    }
}
```

- 结果

```
二月 09, 2017 11:55:25 下午 org.springframework.context.annotation.AnnotationConfigApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@306a30c7: startup date [Thu Feb 09 23:55:25 CST 2017]; root of context hierarchy
二月 09, 2017 11:55:25 下午 org.springframework.context.annotation.ConfigurationClassEnhancer intercept
警告: @Bean method AnnotationBeans.getMyBeanFactoryPostProcessor is non-static and returns an object assignable to Spring's BeanFactoryPostProcessor interface. This will result in a failure to process annotations such as @Autowired, @Resource and @PostConstruct within the method's declaring @Configuration class. Add the 'static' modifier to this method to avoid these container lifecycle issues; see @Bean javadoc for complete details.
construct MyBeanFactoryPostProcessor
post process bean factory
construct MyBeanPostProcessor
post process before instantiation
post process after instantiation
post process property values
post process before initialization
post process after initialization
post process before instantiation
post process after instantiation
post process property values
post process before initialization
post process after initialization
post process before instantiation
construct car
post process after instantiation
post process property values
set bean name
set bean factory
set application context
post process before initialization
post construct
after properties set method
init method
post process after initialization
application context done
get color=color
二月 09, 2017 11:55:25 下午 org.springframework.context.annotation.AnnotationConfigApplicationContext doClose
信息: Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@306a30c7: startup date [Thu Feb 09 23:55:25 CST 2017]; root of context hierarchy
pre destroy
destroy method
my destroy method
```

### 7. 容器内部工作机制
#### 7.1 启动源码
Spring的`AbstractApplicationContext的refresh()`方法定义了Spring容器在加载配置文件后的各项处理工作

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
        // Reset common introspection caches in Spring's core, since we
        // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```
#### 7.2 容器启动流程
`ContextLoaderListener`通过调用继承自`ContextLoader`的`initWebApplicationContext`方法实例化SpringIoC容器。在实例化Spring IoC容器的过程中，最主要的两个方法是`createWebApplicationContext`和`configureAndRefreshWebApplicationContext方`法。`createWebApplicationContext`方法用于返回`XmlWebApplicationContext`实例，即Web环境下的SpringIoC容器。`configureAndRefreshWebApplicationContext`用于配`XmlWebApplicationContext`，读取`web.xml`中通过`contextConfigLocation`标签指定的XML文件，通过调用`refresh`来调用`AbstractApplicationContext`中的`refresh`初始化。

1. `BeanFactory`实例化XML文件中配置的bean，Spring将配置文件的bean的信息解析成为一个个的`BeanDefinition`对象并装入到容器的Bean定义注册表，但此时Bean还未初始化；`obtainFreshBeanFactory()`会调用自身的`refreshBeanFactory()`，而`refreshBeanFactory()`方法由子类`AbstractRefreshableApplicationContext`实现，该方法返回了一个创建的`DefaultListableBeanFactory`对象，这个对象就是由`ApplicationContext`管理的`BeanFactory`容器对象；
2. 调用工厂后处理器：根据反射机制从`BeanDefinitionRegistry`中找出所有`BeanFactoryPostProcessor`类型的Bean，并调用其`postProcessBeanFactory()`接口方法。经过第一步加载配置文件，已经把配置文件中定义的所有bean装载到`BeanDefinitionRegistry`这个`Beanfactory`中，对于`ApplicationContext`应用来说这个`BeanDefinitionRegistry`类型的`BeanFactory`就是Spring默认的`DefaultListableBeanFactory`
3. 注册Bean后处理器：根据反射机制从`BeanDefinitionRegistry`中找出所有`BeanPostProcessor`类型的Bean，并将它们注册到容器Bean后处理器的注册表中；
4. 初始化消息源：初始化容器的国际化信息资源；
5. 初始化应用上下文事件广播器；
6. 初始化其他特殊的Bean；
7. 注册事件监听器；
8. 初始化singleton的Bean：实例化所有singleton的Bean，并将它们放入Spring容器的缓存中；
9. 发布上下文刷新事件：在此处时容器已经启动完成，发布容器`refresh`事件创建上下文刷新事件，事件广播器负责将些事件广播到每个注册的事件监听器中。

#### 7.3 Bean加载流程
1. `ResourceLoader`从存储介质中加载Spring配置文件，并使用`Resource`表示这个配置文件的资源；  
2. `BeanDefinitionReader`读取`Resource`所指向的配置文件资源，然后解析配置文件。配置文件中每一个`<bean>`解析成一个`BeanDefinition`对象，并保存到`BeanDefinitionRegistry`中；  
3. 容器扫描`BeanDefinitionRegistry`中的`BeanDefinition`，使用Java的反射机制自动识别出Bean工厂后处理器（实现`BeanFactoryPostProcessor`接口）的Bean，然后调用这些Bean工厂后处理器对`BeanDefinitionRegistry`中的`BeanDefinition`进行加工处理。主要完成以下两项工作：  
    - 对使用到占位符的`<bean>`元素标签进行解析，得到最终的配置值，这意味对一些半成品式的`BeanDefinition`对象进行加工处理并得到成品的`BeanDefinition`对象；  
    - 对`BeanDefinitionRegistry`中的`BeanDefinition`进行扫描，通过Java反射机制找出所有属性编辑器的Bean（实现`java.beans.PropertyEditor`接口的Bean），并自动将它们注册到Spring容器的属性编辑器注册表中（`PropertyEditorRegistry`）；  
4. Spring容器从`BeanDefinitionRegistry`中取出加工后的`BeanDefinition`，并调用`InstantiationStrategy`着手进行Bean实例化的工作；  
5. 在实例化Bean时，Spring容器使用`BeanWrapper`对Bean进行封装，`BeanWrapper`提供了很多以Java反射机制操作Bean的方法，它将结合该Bean的`BeanDefinition`以及容器中属性编辑器，完成Bean属性的设置工作；  
6. 利用容器中注册的Bean后处理器（实现`BeanPostProcessor`接口的Bean）对已经完成属性设置工作的Bean进行后续加工，直接装配出一个准备就绪的Bean。

### 8. Spring事件
Spring事件体系包括三个组件：事件，事件监听器，事件广播器。
![image](http://my.csdn.net/uploads/201204/14/1334392012_7683.JPG)

- 事件：`ApplicationEvent`
- 事件监听器：`ApplicationListener`，对监听到的事件进行处理。
- 事件广播器：`ApplicationEventMulticaster`，将Spring publish的事件广播给所有的监听器。Spring在`ApplicationContext`接口的抽象实现类`AbstractApplicationContext`中完成了事件体系的搭建。
- `AbstractApplicationContext`拥有一个`applicationEventMulticaster`成员变量，`applicationEventMulticaster`提供了容器监听器的注册表。

#### 8.1 事件广播器的初始化

```java
private void initApplicationEventMulticaster() throws BeansException {  
    if (containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME )) {  
        this.applicationEventMulticaster = (ApplicationEventMulticaster)  
            getBean( APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class );  
        if (logger.isInfoEnabled()) {
            logger.info("Using ApplicationEventMulticaster [" + this. applicationEventMulticaster + "]" );
        }  
    }  
    else {  
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster();  
        if (logger.isInfoEnabled()) {
            logger.info("Unable to locate ApplicationEventMulticaster with name '"+  APPLICATION_EVENT_MULTICASTER_BEAN_NAME +  
                        "': using default [" + this .applicationEventMulticaster + "]");  
        }  
    }  
 }  
```
用户可以在配置文件中为容器定义一个自定义的事件广播器，只要实现`ApplicationEventMulticaster`就可以了，Spring会通过反射的机制将其注册成容器的事件广播器，如果没有找到配置的外部事件广播器，Spring自动使用 `SimpleApplicationEventMulticaster`作为事件广播器。

#### 8.2 注册事件监听器

```java
private void registerListeners () throws BeansException {
    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    Collection listeners = getBeansOfType(ApplicationListener.class,true,false).values();
    for (Iterator it = listeners.iterator(); it.hasNext();) {
        addListener((ApplicationListener) it.next());
    }
}
protected void addListener(ApplicationListener listener) {
    getApplicationEventMulticaster().addApplicationListener(listener);
}
```
Spring根据反射机制，使用`ListableBeanFactory`的`getBeansOfType`方法，从`BeanDefinitionRegistry`中找出所有实现 `org.springframework.context.ApplicationListener`的Bean，将它们注册为容器的事件监听器，实际的操作就是将其添加到事件广播器所提供的监听器注册表中。

#### 8.3 发布事件

```java
public void publishEvent(ApplicationEvent event) {
    Assert.notNull(event, "Event must not be null");
    if (logger.isDebugEnabled()) {
        logger.debug("Publishing event in context ["
                      + getDisplayName() + "]: " + event);
    }
    getApplicationEventMulticaster().multicastEvent(event);
    if (this.parent != null) {
        this.parent.publishEvent(event);
    }
}
```
在`AbstractApplicationContext`的`publishEvent`方法中， Spring委托`ApplicationEventMulticaster`将事件通知给所有的事件监听器

#### 8.4 Spring默认的事件广播器SimpleApplicationEventMulticaster

```java
public void multicastEvent( final ApplicationEvent event) {
    for (Iterator it = getApplicationListeners().iterator(); it.hasNext();) {
        final ApplicationListener listener = (ApplicationListener) it.next();
        getTaskExecutor().execute(new Runnable() {
            public void run() {
                listener.onApplicationEvent(event);
            }
        });
    }
 }
```
遍历注册的每个监听器，并启动来调用每个监听器的`onApplicationEvent`方法。
由于`SimpleApplicationEventMulticaster`的`taskExecutor`的实现类是`SyncTaskExecutor`，因此，事件监听器对事件的处理，是同步进行的。

#### 8.5 举例
- `springEvent.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.event" />
</beans>
```

- `MockEvent`

```java
package com.event;
import org.springframework.context.ApplicationContext;
import org.springframework.context.event.ApplicationContextEvent;

public class MockEvent extends ApplicationContextEvent {

    public MockEvent(ApplicationContext source) {
        super(source);
    }
}
```

- `MockEventListener`

```java
package com.event;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

@Component
public class MockEventListener implements ApplicationListener<MockEvent> {

    public void onApplicationEvent(MockEvent event) {
        System.out.println("mock event received");
    }
}
```

- `MockEventPublisher`

```java
package com.event;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;;

@Component
public class MockEventPublisher implements ApplicationContextAware {

    private ApplicationContext ctx;

    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.ctx = applicationContext;
    }

    public void publishEvent() {
        System.out.println("publish event");
        MockEvent event = new MockEvent(this.ctx);
        ctx.publishEvent(event);
    }
}
```

- `MockEventTest`

```java
package com.event;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MockEventTest {

    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext(
                "/springEvent.xml");
        MockEventPublisher publisher = ctx.getBean(MockEventPublisher.class);
        publisher.publishEvent();
        ctx.close();
    }
}
```

- 结果

```
二月 09, 2017 9:57:43 下午 org.springframework.context.support.ClassPathXmlApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 21:57:43 CST 2017]; root of context hierarchy
二月 09, 2017 9:57:43 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [springEvent.xml]
publish event
mock event received
二月 09, 2017 9:57:44 下午 org.springframework.context.support.ClassPathXmlApplicationContext doClose
信息: Closing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 21:57:43 CST 2017]; root of context hierarchy
```
