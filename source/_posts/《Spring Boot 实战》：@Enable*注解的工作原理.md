---
title: '《Spring Boot 实战》: @Enable* 注解的工作原理'
date: 2017-05-19 00:57:52
tags: Spring Boot
categories: [读书笔记, Spring Boot]
---

通过观察 `@Enable*` 注解的源码，可以发现所有的注解都有一个 `@Import` 注解。

`@Import` 注解是用来导入配置类的，这也就是说这些自动开启的实现其实是导入了一些自动配置的 Bean。
<!--more-->

## @Import 注解导入配置方式的三种类型
### 1. 直接导入配置类

```Java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({SchedulingConfiguration.class})
@Documented
public @interface EnableScheduling {
}
```

直接导入配置类 `SchedulingConfiguration`，这个类注解了 `@Configuration`，且注册了一个 `scheduledAnnotationProcessor` 的 Bean。

### 2. 依据条件选择配置类
```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
    Class<? extends Annotation> annotation() default Annotation.class;
    boolean proxyTargetClass() default false;
    AdviceMode mode() default AdviceMode.PROXY;
    int order() default Ordered.LOWEST_PRECEDENCE;

}
```
`AsyncConfigurationSelector` 通过条件来选择需要导入的配置类，`AsyncConfigurationSelector` 的根接口为 `ImportSelector`，这个接口需要重写 `selectImports` 方法，在此方法内进行事先条件判断。若 `adviceMode` 为 `PORXY`，则返回 `ProxyAsyncConfiguration` 这个配置类；若 `activeMode` 为 `ASPECTJ`，则返回 `AspectJAsyncConfiguration` 配置类，源码如下：

```Java
public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {

    private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME =
            "org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";

    /**
     * {@inheritDoc}
     * @return {@link ProxyAsyncConfiguration} or {@code AspectJAsyncConfiguration} for
     * {@code PROXY} and {@code ASPECTJ} values of {@link EnableAsync#mode()}, respectively
     */
    @Override
    public String[] selectImports(AdviceMode adviceMode) {
        switch (adviceMode) {
            case PROXY:
                return new String[] { ProxyAsyncConfiguration.class.getName() };
            case ASPECTJ:
                return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
            default:
                return null;
        }
    }
}
```

### 3. 动态注册 Bean
```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
}
```
`AspectJAutoProxyRegistrar` 实现了 `ImportBeanDefinitionRegistrar` 接口，`ImportBeanDefinitionRegistrar` 的作用是在运行时自动添加 Bean 到已有的配置类，通过重写方法：

```Java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)
```
其中，`AnnotationMetadata` 参数用来获得当前配置类上的注解；`BeanDefinittionRegistry` 参数用来注册 Bean。源码如下：

```Java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

    /**
     * Register, escalate, and configure the AspectJ auto proxy creator based on the value
     * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
     * {@code @Configuration} class.
     */
    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        AnnotationAttributes enableAJAutoProxy =
                AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }
    }
}
```