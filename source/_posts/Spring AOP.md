---
title: Spring AOP
date: 2016-02-13 15:53:38 +0800
categories: Spring
tags: Spring
---

### 1. 术语
- 连接点（JointPoint）：代码中具有边界性质特定点；Spring仅支持方法的连接点，包含方法和方位两方面信息
- 切点（Pointcut）：定位到某个方法
- 增强（Advice）：织入到目标连接点上的代码
- 目标对象（Target）：增强逻辑的目标织入类
- 引介（Introduction）：特殊的增强，为类添加一些属性和方法
- 织入（Weaving）：将增强添加到目标连接点上的过程：编译期织入、类装载期织入、动态代理织入（Spring的方案）
- 代理（Proxy）：被AOP织入增强后的结果类
- 切面（Aspect）：切点+增强

<!-- more -->

### 2. 动态代理的两种实现：JDK和CGLib
- JDK动态代理动态创建一个符合某一接口的实力，生成目标类的代理对象，缺点是需要提供接口；方法必须是`public`或`public final`的
- CGLib采用底层的字节码技术，在子类中对父类的方法进行拦截，织入横切逻辑；不能为`final`和`private`方法代理
- 样例

```java
package test;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class ProxyTest {

    public static void main(String[] args) {
        ServiceImpl jdkTarget = new ServiceImpl();
        ProxyHandler handler = new ProxyHandler(jdkTarget);
        Service jdkProxy = (Service)Proxy.newProxyInstance(
            jdkTarget.getClass().getClassLoader(),
            jdkTarget.getClass().getInterfaces(),
            handler);
        jdkProxy.process("jdk proxy");

        System.out.println();

        CglibProxy cglibProxy = new CglibProxy();
        ServiceImpl cglibTarget = (ServiceImpl)cglibProxy.getProxy(ServiceImpl.class);
        cglibTarget.process("cglib proxy");
    }

    public interface Service {
        public void process(String arg);
    }

    public static class ServiceImpl implements Service {
        @Override
        public void process(String arg) {
            System.out.println("do something with " + arg);
        }
    }

    //jdk proxy
    public static class ProxyHandler implements InvocationHandler {
        private Object target;
        public ProxyHandler(Object target) {
            this.target = target;
        }
        @Override
        public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
            System.out.println("before process jdk proxy");
            Object obj = method.invoke(target, args);
            System.out.println("after process jdk proxy");
            return obj;
        }
    }

    //cglib proxy
    public static class CglibProxy implements MethodInterceptor {
        private Enhancer enhancer = new Enhancer();
        public Object getProxy(Class clazz) {
            enhancer.setSuperclass(clazz);
            enhancer.setCallback(this);
            return enhancer.create();
        }

        @Override
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            System.out.println("before process cglib proxy");
            Object result = proxy.invokeSuper(obj, args);
            System.out.println("after process cglib proxy");
            return result;
        }
    }
}
```

- 结果

```
before process jdk proxy
do something with jdk proxy
after process jdk proxy

before process cglib proxy
do something with cglib proxy
after process cglib proxy
```
- 性能：CGLib所创建的动态代理对象性能比JDK方式高（约10倍），但CGLib在创建代理对象时所花费的时间比JDK方式多（约8倍）；CGLib适合Spring里singleton模式的bean管理

### 3. ProxyFactory
- Spring定义了`org.springframework.aop.framework.AopProxy`接口及`Cglib2AopProxy`和`JdkDynamicAopProxy`两个`final`实现类
- 如果通过`ProxyFactory`的`setInterfaces(Class[] interfaces)`指定针对接口代理，则使用`JdkDynamicAopProxy`；如果使用`setOptimize(true)`，使用`Cglib2AopProxy`
- `ProxyFacotry`通过`addAdvice(Advice)`形成增强链

### 4. 增强类型

#### 4.1 前置增强
- 接口：`org.springframework.aop.BeforeAdvice`
- 样例

```java
package com.aop;
import java.lang.reflect.Method;
import org.springframework.aop.MethodBeforeAdvice;

public class BeforeAdvice implements MethodBeforeAdvice {
    @Override
    public void before(Method arg0, Object[] arg1, Object arg2)
        throws Throwable {
        String arg = (String)arg1[0];
        System.out.println("before advice " + arg);
    }
}
```

#### 4.2  后置增强
- 接口：`org.springframework.aop.AfterReturninigAdvice`
- 样例

```java
package com.aop;
import java.lang.reflect.Method;
import org.springframework.aop.AfterReturningAdvice;

public class AfterAdvice implements AfterReturningAdvice {
    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
        String arg = (String)args[0];
        System.out.println("after advice " + arg);
    }
}
```

#### 4.3 环绕增强
- 接口：`org.aopalliance.intercept.MethodInterceptor`
- 样例

```java
package com.aop;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

public class AroundAdvice implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Object[] args = invocation.getArguments();
        String arg = (String)args[0];
        System.out.println("around advice: before " + arg);
        Object obj = invocation.proceed();
        System.out.println("around advice: after " + arg);
        return obj;
    }
}
```

#### 4.4 异常抛出增强
- 接口：`org.springframework.aop.ThrowsAdvice`
- 样例

```java
package com.aop;
import java.lang.reflect.Method;
import org.springframework.aop.ThrowsAdvice;

public class ExceptionAdvice implements ThrowsAdvice {
    public void afterThrowing(Method method, Object[] args, Object target, Exception ex)
        throws Throwable {
        System.out.println("------");
        System.out.println("throws exception, method=" + method.getName());
        System.out.println("throws exception, message=" + ex.getMessage());
    }
}
```

#### 4.5 测试
##### 4.5.1 基于代码的测试
- `TestAopAdvice`

```java
package com.aop;
import org.springframework.aop.ThrowsAdvice;
import org.springframework.aop.framework.ProxyFactory;

public class TestAopAdvice {
    public static void main(String[] args) {
        AopExample example = new AopExample();
        BeforeAdvice beforeAdvice = new BeforeAdvice();
        AfterAdvice afterAdvice = new AfterAdvice();
        AroundAdvice aroundAdvice = new AroundAdvice();
        ThrowsAdvice throwsAdvice = new ExceptionAdvice();

        ProxyFactory pf = new ProxyFactory();
        pf.setTarget(example);
        pf.addAdvice(beforeAdvice);
        pf.addAdvice(afterAdvice);
        pf.addAdvice(aroundAdvice);
        pf.addAdvice(throwsAdvice);
        AopExample proxy = (AopExample)pf.getProxy();

        proxy.handle("blabla");
        System.out.println();

        try{
            proxy.throwExp("blabla");
        } catch(Exception e) {
        }
    }
}
```

- 输出

```
before advice blabla
around advice: before blabla
aop example blabla
around advice: after blabla
after advice blabla

before advice blabla
around advice: before blabla
----after throwing----
throws exception, method=throwExp
throws exception, message=try throws advice
```

##### 4.5.2 基于Spring配置的测试
- `springAop.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/aop
    http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="aopExample" class="com.aop.AopExample" />
    <bean id="beforeAdvice" class="com.aop.BeforeAdvice" />
    <bean id="afterAdvice" class="com.aop.AfterAdvice" />
    <bean id="aroundAdvice" class="com.aop.AroundAdvice" />
    <bean id="exceptionAdvice" class="com.aop.ExceptionAdvice" />
    <bean id="aopTest" class="org.springframework.aop.framework.ProxyFactoryBean"
        p:proxyTargetClass="true"
        p:interceptorNames="beforeAdvice,afterAdvice,aroundAdvice,exceptionAdvice"
        p:target-ref="aopExample" />
</beans>
```

- `TestAopAdvice2`

```java
package com.aop;

import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class TestAopAdvice2 {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("/springAop.xml");
        AopExample aopExample = (AopExample)ctx.getBean("aopTest");
        aopExample.handle("blabla");
        System.out.println();
        try{
            aopExample.throwExp("blabla");
        } catch(Exception e) {
        }
        ctx.close();
    }
}
```

- 输出

```
二月 09, 2017 9:54:11 下午 org.springframework.context.support.ClassPathXmlApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 21:54:11 CST 2017]; root of context hierarchy
二月 09, 2017 9:54:11 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [springAop.xml]
before advice blabla
around advice: before blabla
aop example blabla
around advice: after blabla
after advice blabla

before advice blabla
around advice: before blabla
----after throwing----
throws exception, method=throwExp
throws exception, message=try throws advice
二月 09, 2017 9:54:11 下午 org.springframework.context.support.ClassPathXmlApplicationContext doClose
信息: Closing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 21:54:11 CST 2017]; root of context hierarchy
```

#### 4.6 引介增强
- 接口：`org.springframework.aop.IntroductionInterceptor`

##### 4.6.1 基于Spring配置的测试代码

- `IntroductionAdvice`

```java
package com.aop;

public interface IntroductionAdvice {
    public void setIntroductionActive(boolean active);
}
```

- `ConfigurableIntroduction`

```java
package com.aop;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.support.DelegatingIntroductionInterceptor;

public class ConfigurableIntroduction extends DelegatingIntroductionInterceptor implements IntroductionAdvice {

    private ThreadLocal<Boolean> map = new ThreadLocal<Boolean>();

    @Override
    public void setIntroductionActive(boolean active) {
        map.set(active);
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Object obj = null;
        if(map.get() != null && map.get()) {
            System.out.println("before monitor operation");
            obj = super.invoke(invocation);
            System.out.println("after monitor operation");
        } else {
            obj = super.invoke(invocation);
        }
        return obj;
    }
}
```

- `TestIntroductionAdvice`

```java
package com.aop;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class TestIntroductionAdvice {

    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("/springAop.xml");
        AopExample introductionAop = (AopExample)ctx.getBean("introductionAop");
        introductionAop.handle("introduction advice");
        IntroductionAdvice ci = (IntroductionAdvice)introductionAop;
        ci.setIntroductionActive(true);
        introductionAop.handle("introduction advice");
        ctx.close();
    }
}
```

- `springAop.xml`添加

```xml
<bean id="configurableIntroduction" class="com.aop.ConfigurableIntroduction" />
<bean id="introductionAop" class="org.springframework.aop.framework.ProxyFactoryBean"
    p:interfaces="com.aop.IntroductionAdvice"
    p:target-ref="aopExample"
    p:interceptorNames="configurableIntroduction"
    p:proxyTargetClass="true" />
```

- 输出

```
二月 09, 2017 9:56:10 下午 org.springframework.context.support.ClassPathXmlApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 21:56:10 CST 2017]; root of context hierarchy
二月 09, 2017 9:56:10 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [springAop.xml]
aop example introduction advice
----
before monitor operation
aop example introduction advice
after monitor operation
二月 09, 2017 9:56:11 下午 org.springframework.context.support.ClassPathXmlApplicationContext doClose
信息: Closing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 21:56:10 CST 2017]; root of context hierarchy
```

##### 4.6.2 与其他增强在配置上的区别
- 须指定引介增强所实现的接口
- 只能通过为目标类创建子类的方式生成引介增强的代理，因此`proxyTargeClass`必须为`true`

### 5. Spring中的配置

参数说明

- `target`：代理的对象
- `proxyInterfaces`：代理所要实现的接口
- `interceptorNames`：需要织入目标对象的bean列表，这些bean必须是实现了`org.aopalliance.intercept.MethodInterceptor`或`org.springframework.aop.Advisor`的bean，配置中的顺序对应调用的顺序
- `singleton`：返回的代理是否为单例，默认为`true`
- `optimize`：为`true`时使用CGLib代理
- `proxyTargetClass`：为`true`时使用CGLib代理，并覆盖`proxyInterfaces`设置

### 6. Java注解
- 一个例子

```java
package com.aspectj;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;

@Target(ElementType.METHOD) //声明可以使用该注解的目标类型
@Retention(RetentionPolicy.RUNTIME)//声明注解的保留期限
public @interface Authority {
    boolean value() default true;//声明注解成员
}
```

- 成员以无入参无抛出异常的方式声明
- 可以通过`default`为成员指定一个默认值
- 在方法上使用注解：`@Authority(value=true)`
- 如果注解只有一个成员，需命名为`value()`，使用时可以忽略成员名和赋值号(=)，如`@Authority(true)`
- 注解类拥有多个成员时，如果仅对`value`成员赋值，可以不适用赋值号；如果同时对多个成员赋值，则必须使用赋值号
- 注解类可以没有成员，称为标识注解
- 所有注解类隐式继承于`java.lang.annotation.Annotation`，注解不允许显式继承于其他接口
- 如果成员是数组类型，可以通过{}赋值

### 7. 基于AspectJ的AOP
#### 7.1 一个例子
- 定义切面

```java
package com.aspectj;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect
public class PreAspect {
    @Before("execution(*handle(..))")
    public void before() {
        System.out.println("aspect: before processing");
    }
}
```

- 测试

```java
package com.aspectj;
import org.springframework.aop.aspectj.annotation.AspectJProxyFactory;
import com.aop.AopExample;

public class AspectJTest {
    public static void main(String[] args) {
        AspectJProxyFactory factory = new AspectJProxyFactory();
        AopExample example = new AopExample();
        factory.setTarget(example);
        factory.addAspect(PreAspect.class);
        AopExample proxy = factory.getProxy();
        proxy.handle("pre aspect");
    }
}
```

- 结果

```
aspect: before processing
aop example pre aspect
```

#### 7.2 通过配置使用切面
##### 7.2.1 典型配置
- `springAspectj.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/aop
    http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="aopExample" class="com.aop.AopExample" />
    <bean id="preAspect" class="com.aspectj.PreAspect" />
    <bean class="org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator" />
</beans>
```

##### 7.2.2 基于Schema的配置
- `springAspectj.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/aop
    http://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:aspectj-autoproxy />
    <bean id="aopExample" class="com.aop.AopExample" />
    <bean id="preAspect" class="com.aspectj.PreAspect" />
</beans>
```

- `AspectJTest2`

```java
package com.aspectj;

import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import com.aop.AopExample;

public class AspectJTest2 {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("/springAspectj.xml");
        AopExample aopExample = (AopExample)ctx.getBean("aopExample");
        aopExample.handle("blabla");
        ctx.close();
    }
}
```

- 输出

```
二月 09, 2017 10:13:56 下午 org.springframework.context.support.ClassPathXmlApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 22:13:56 CST 2017]; root of context hierarchy
二月 09, 2017 10:13:56 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [springAspectj.xml]
aspect: before processing
aop example blabla
二月 09, 2017 10:13:57 下午 org.springframework.context.support.ClassPathXmlApplicationContext doClose
信息: Closing org.springframework.context.support.ClassPathXmlApplicationContext@69d0a921: startup date [Thu Feb 09 22:13:56 CST 2017]; root of context hierarchy
```

- 通过`<aop:aspectj-autoproxy />`引入`aop`命名空间，自动为Spring容器中匹配`@AspectJ`切面的bean创建代理，完成切面织入，其内部实现仍为`AnnotationAwareAspectJAutoProxyCreator`
- `<aop:aspectj-autoproxy />`的`proxy-target-class`属性为`false`时，采用JDK动态代理；为`true`时使用CGLib

### 8. AspectJ语法
#### 8.1 切点表达式函数
- 分类

| 类型       | 说明                  | 举例                                       |
| -------- | ------------------- | ---------------------------------------- |
| 方法切点函数   | 通过描述目标类方法信息定义连接点    | `execution()`，`@annotation()`            |
| 方法入参切点函数 | 通过描述目标类方法入参的信息定义连接点 | `args()`，`@args()`                       |
| 目标类切点函数  | 通过描述目标类类型信息定义连接点    | `within()`，`target()`，`@within()`，`@target()` |
| 代理类切点函数  | 通过描述目标类的代理类的信息定义连接点 | `this()`                                 |

- 函数说明

| 函数              | 入参      | 说明                                       |
| --------------- | ------- | ---------------------------------------- |
| `execution()`   | 方法匹配模式串 | 表示满足某一匹配模式的所有目标类方法连接点，如`execution(* handle(..))`表示所有目标类中的`handle()`方法 |
| `@annotation()` | 方法注解类名  | 表示标注了特定注解的目标方法连接点，如`@annotation(com.aspectj.Authority)`表示任何标注了`@Authority`注解的目标类方法 |
| `args()`        | 类名      | 通过判别目标类方法运行时入参对象的类型定义指定连接点，如`args(com.data.Car)`表示所有有且仅有一个按类型匹配于`Car`（含子类）入参的方法 |
| `@args()`       | 类型注解类名  | 通过判别目标方法运行时入参对象的类是否标注特定注解来制定连接点，如`@args(com.aspectj.Authority)`表示任何这样的一个目标方法：它有一个入参且入参对象的类标注`@Authority`注解。要使`@args()`生效，类继承树中，标注注解的类类型需要不高于入参类类型 |
| `within`        | 类名匹配串   | 表示特定域下的所有连接点，如`within(com.service.*)`，`within(com.service.*Service)`和`within(com.service..*)` |
| `target()`      | 类名      | 假如目标按类型匹配于指定类，则目标类的所有连接点匹配这个切点。如通过`target(com.data.Car)`定义的切点，`Car`及`Car`的子类中的所有连接点都匹配该切点，包括子类中扩展的方法 |
| `@within()`     | 类型注解类名  | 假如目标类按类型匹配于某个类`A`，且类`A`标注了特定注解，则目标类的所有连接点都匹配于这个切点。如`@within(com.aspectj.Authority)`定义的切点，假如`Car`类标注了`@Authority`注解，则`Car`以及`Car`的子类的所有连接点都匹配。`@within`标注接口类无效 |
| `@target()`     | 类型注解类名  | 目标类标注了特定注解，则目标类（不包括子类）所有连接点都匹配该切点。如通过`@target(com.aspectj.Authority)`定义的切点，若`BMWCar`标注了`@Authority`，则`BMWCar`所有连接点匹配该切点 |
| `this()`        | 类名      | 代理类按类型匹配于指定类，则被代理的目标类所有连接点匹配切点           |

#### 8.2 通配符
##### 8.2.1 通配符类型

| 类型   | 说明                                       |
| ---- | ---------------------------------------- |
| `*`  | 匹配任意字符，但只能匹配上下文中的一个元素                    |
| `..` | 匹配任意字符，可以匹配上下文中的多个元素。表示类时，和`*`联合使用；表示入参时单独使用 |
| `+`  | 按类型匹配指定类的所有类（包括实现类和继承类），必须跟在类名后面         |

##### 8.2.1 函数按通配符支持分类
- 支持所有通配符：`execution()`，`within()`
- 仅支持`+`通配符：`args()`，`this()`，`target()`
- 不支持通配符：`@args`，`@within`，`@target`，`@annotation`

#### 8.3 增强类型
- `@Before`：前置增强，相当于`BeforeAdvice`
- `@AfterReturning`：后置增强，相当于`AfterReturningAdvice`
- `@Around`：环绕增强，相当于`MethodInterceptor`
- `@AfterThrowing`：相当于`ThrowsAdvice`
- `@After`：Final增强，抛出异常或正常退出都会执行的增强
- `@DeclareParents`：引介增强，相当于`IntroductionInterceptor`

#### 8.4 Execution()
- 语法：`execution(<修饰符模式>? <返回类型模式> <方法名模式> (<参数模式>) <异常模式>?)`

##### 8.4.1 通过方法签名定义切点
- `execution(pulic * *(..))`：匹配目标类的`public`方法，第一个`*`代表返回类型，第二个`*`代表方法名，`..`代表任意入参
- `execution(* *To(..))`：匹配目标类所有以`To`结尾的方法，第一个`*`代表返回类型，`*To`代表任意以`To`结尾的方法

##### 8.4.2 通过类定义切点
- `execution(* com.data.User.*(..))`：匹配`User`接口的所有方法
- `execution(* com.data.User+.*(..))`：匹配`User`接口的所有方法，包括其实现类中不在`User`接口中定义的方法

##### 8.4.3 通过类包定义切点
- `execution(* com.data.*(..))`：匹配`data`包下所有类的所有方法
- `execution(* com.data.User..*(..))`：匹配`data`包及其子孙包中的所有类的所有方法
- `execution(* com..*Manager.get*(..))`：匹配`com`包及其子孙包中后缀为`Manager`的类里以`get`开头的方法

##### 8.4.4 通过方法入参定义切点
- `execution(* get(String, int))`：匹配`get(String, int)`方法
- `execution(* get(String, *))`：匹配名为get且第一个入参类型为`String`、第二个入参类型任意的方法
- `execution(* get(String, ..))`：匹配名为`get`且第一个入参为`String`类型的方法
- `execution(* get(Object+))`：匹配名为`get`且唯一入参是`Object`或其子类的方法

#### 8.5 进阶
##### 8.5.1 逻辑运算符
- 与`&&`，或`||`，非`!`

##### 8.5.2 切点复合运算
- 例如：`@After("within(com.data.*) && execution(* handle(..))")`

##### 8.5.3 命名切点
- 使用`@Pointcut`命名切点
- 使用方法名作为切点的名称，方法的访问控制符控制切点的可用性

```java
@Pointcut("within(com.data.*)")
public void inPackage(){} //别名为inPackage
```

##### 8.5.4 增强织入的顺序
- 如果增强在同一个切面类中声明，则依照增强在切面类中定义的顺序织入
- 如果增强位于不同的增强类中，且都实现了`org.springframework.core.Ordered`接口，则由接口方法的顺序号决定（顺序号小的先织入）
- 如果增强位于不同的增强类中，且没有实现`org.springframework.core.Ordered`接口，织入顺序不确定

##### 8.5.5 访问连接点信息
- AspectJ使用`org.aspectj.lang.JointPoint`接口表示目标类连接点对象。如果是环绕增强时，使用`org.aspectj.lang.ProceedingJointPoint`表示连接点对象，该类是`JointPoint`接口的子接口。任何一个增强方法都可以通过将第一个入参声明为`JointPoint`访问到连接点上下文的信息

##### 8.5.6 绑定连接点方法入参
- `args()`用于绑定连接点方法的入参，`@annotation()`用于绑定连接点方法的注解对象，`@args()`用于绑定连接点方法的入参注解。下例表示方法入参为`(String, int, ..)`的方法匹配该切点，并将`name`和`age`两个参数绑定到切面方法的入参中

```java
@Before("args(name, age, ..)")
public void bindJointPointValues(String name, int age) {
    //do something
}
```

##### 8.5.7 绑定代理对象
- 使用`this()`或`target()`可以绑定被代理对象的实例。下例表示代理对象为`User`类的所有方法匹配该切点，且代理对象绑定到`user`入参中

```java
@Before("this(user)")
public void bindProxy(User user) {
    //do something
}
```

##### 8.5.8 绑定类注解对象
- `@within()`和`@target()`函数可以将目标类的注解对象绑定到增强方法中

```java
@Before("@within(a)")
public void bindAnnotation(Authority a) {
    //do something
}
```

##### 8.5.9 绑定返回值
- 通过`returning`绑定连接点方法的返回值

```java
@AfterReturning(value="target(com.data.Car)", returning="rvl")
public void bindReturningValue(int rvl) {
    //do something
}
```
- `rvl`的类型必须和连接点方法的返回值类型匹配

##### 8.5.10 绑定抛出的异常
- 使用`AfterThrowing`注解的`throwing`成员绑定

```java
@AfterThrowing(value="target(com.data.Car)", throwing="iae")
public void bindException(IllegalArgumentException iae) {
    //do something
}
```
