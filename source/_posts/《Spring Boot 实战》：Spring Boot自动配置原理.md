---
title: '《Spring Boot 实战》: Spring Boot 自动配置原理'
date: 2017-05-11 02:10:58
tags: Spring Boot
categories: [读书笔记, Spring Boot]
---
SpringBoot 框架可用于创建可执行的 Spring 应用程序，采用了习惯优于配置的方法。其中奥秘在于 `@EnableAutoConfiguration` 注释，此注释自动载入应用程序所需的所有 Bean——这依赖于 SpringBoot 在类路径中的查找。
<!--more-->

## 1. @SpringBootApplication
首先来看 `@SpringBootApplication` 注解:

```Java
package org.springframework.boot.autoconfigure;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.boot.SpringBootConfiguration;
import org.springframework.boot.context.TypeExcludeFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.ComponentScan.Filter;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.FilterType;
import org.springframework.core.annotation.AliasFor;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class, attribute = "exclude")
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class, attribute = "excludeName")
	String[] excludeName() default {};

	/**
	 * Base packages to scan for annotated components. Use {@link #scanBasePackageClasses}
	 * for a type-safe alternative to String-based package names.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	/**
	 * Type-safe alternative to {@link #scanBasePackages} for specifying the packages to
	 * scan for annotated components. The package of each class specified will be scanned.
	 * <p>
	 * Consider creating a special no-op marker class or interface in each package that
	 * serves no purpose other than being referenced by this attribute.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

}

```

该注解上存在元注解`@EnableAutoConfiguration`，这就是 Spring Boot 自动配置实现的核心入口，其定义为：

```Java
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```
可见通过`@Import`注解，引入了`EnableAutoConfigurationImportSelector`。

## 2. @EnableAutoConfigurationImportSelector

```Java
public class EnableAutoConfigurationImportSelector
		extends AutoConfigurationImportSelector {

	@Override
	protected boolean isEnabled(AnnotationMetadata metadata) {
		if (getClass().equals(EnableAutoConfigurationImportSelector.class)) {
			return getEnvironment().getProperty(
					EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
					true);
		}
		return true;
	}

}
```

父类 `AutoConfigurationImportSelector` 的 `selectImports` 方法如下：

```Java
public String[] selectImports(AnnotationMetadata metadata) {  
    try {  
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata  
                .getAnnotationAttributes(EnableAutoConfiguration.class.getName(),  
                        true));  
  
        Assert.notNull(attributes, "No auto-configuration attributes found. Is "  
                + metadata.getClassName()  
                + " annotated with @EnableAutoConfiguration?");  
  
        // Find all possible auto configuration classes, filtering duplicates  
        List<String> factories = new ArrayList<String>(new LinkedHashSet<String>(  
                SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class,  
                        this.beanClassLoader)));  
  
        // Remove those specifically disabled  
        factories.removeAll(Arrays.asList(attributes.getStringArray("exclude")));  
  
        // Sort  
        factories = new AutoConfigurationSorter(this.resourceLoader)  
                .getInPriorityOrder(factories);  
  
        return factories.toArray(new String[factories.size()]);  
    }  
    catch (IOException ex) {  
        throw new IllegalStateException(ex);  
    }  
}  
```
该方法使用了 Spring Core 包的 `SpringFactoriesLoader` 类的 `loadFactoryNamesof()` 方法，查询 `META-INF/spring.factories` 文件下以 `EnableAutoConfiguration` 的全限定名（`org.springframework.boot.autoconfigure.EnableAutoConfiguration`）为 key 的对应值，其结果为：

```
# Auto Configure  
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\  
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\  
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\  
org.springframework.boot.autoconfigure.MessageSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration,\  
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\  
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\  
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\  
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\  
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration,\  
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration,\  
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration,\  
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration,\  
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\  
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\  
org.springframework.boot.autoconfigure.jms.hornetq.HornetQAutoConfiguration,\  
org.springframework.boot.autoconfigure.jta.JtaAutoConfiguration,\  
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchAutoConfiguration,\  
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchDataAutoConfiguration,\  
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,\  
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration,\  
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration,\  
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration,\  
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\  
org.springframework.boot.autoconfigure.mobile.DeviceResolverAutoConfiguration,\  
org.springframework.boot.autoconfigure.mobile.DeviceDelegatingViewResolverAutoConfiguration,\  
org.springframework.boot.autoconfigure.mobile.SitePreferenceAutoConfiguration,\  
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\  
org.springframework.boot.autoconfigure.mongo.MongoDataAutoConfiguration,\  
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration,\  
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\  
org.springframework.boot.autoconfigure.reactor.ReactorAutoConfiguration,\  
org.springframework.boot.autoconfigure.redis.RedisAutoConfiguration,\  
org.springframework.boot.autoconfigure.security.SecurityAutoConfiguration,\  
org.springframework.boot.autoconfigure.security.FallbackWebSecurityAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.SocialWebAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.FacebookAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.LinkedInAutoConfiguration,\  
org.springframework.boot.autoconfigure.social.TwitterAutoConfiguration,\  
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration,\  
org.springframework.boot.autoconfigure.velocity.VelocityAutoConfiguration,\  
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.GzipFilterAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.HttpEncodingAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.HttpMessageConvertersAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.MultipartAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.ServerPropertiesAutoConfiguration,\  
org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration,\  
org.springframework.boot.autoconfigure.websocket.WebSocketAutoConfiguration  
```
在这个文件中，可以看到一系列 Spring Boot 自动配置的列表。

## 3. MongoAutoConfiguration
以 `MongoAutoConfiguration` 为例：

```Java
@Configuration
@ConditionalOnClass(Mongo.class)
@EnableConfigurationProperties(MongoProperties.class)
public class MongoAutoConfiguration {

    @Autowired
    private MongoProperties properties;

    private Mongo mongo;

    @PreDestroy
    public void close() throws UnknownHostException {
        if (this.mongo != null) {
            this.mongo.close();
        }
    }

    @Bean
    @ConditionalOnMissingBean
    public Mongo mongo() throws UnknownHostException {
        this.mongo = this.properties.createMongoClient();
        return this.mongo;
    }

}
```
这个类进行了简单的 Spring 配置，声明了 MongoDB 所需典型 Bean，并且和其他很多自动配置类一样，重度依赖于 Spring Boot 的注解（`@Condition*`）。

## 4. 调试
以 DEBUG 级 log 启动 Springboot 项目，Spring Boot 会产生一个报告，如下：

```shell
Positive matches:
-----------------

   MessageSourceAutoConfiguration
      - @ConditionalOnMissingBean (types: org.springframework.context.MessageSource; SearchStrategy: all) found no beans (OnBeanCondition)

   JmxAutoConfiguration
      - @ConditionalOnClass classes found: org.springframework.jmx.export.MBeanExporter (OnClassCondition)
      - SpEL expression on org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration: ${spring.jmx.enabled:true} (OnExpressionCondition)
      - @ConditionalOnMissingBean (types: org.springframework.jmx.export.MBeanExporter; SearchStrategy: all) found no beans (OnBeanCondition)

   DispatcherServletAutoConfiguration
      - found web application StandardServletEnvironment (OnWebApplicationCondition)
      - @ConditionalOnClass classes found: org.springframework.web.servlet.DispatcherServlet (OnClassCondition)


Negative matches:
-----------------

   DataSourceAutoConfiguration
      - required @ConditionalOnClass classes not found: org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType (OnClassCondition)

   DataSourceTransactionManagerAutoConfiguration
      - required @ConditionalOnClass classes not found: org.springframework.jdbc.core.JdbcTemplate,org.springframework.transaction.PlatformTransactionManager (OnClassCondition)

   MongoAutoConfiguration
      - required @ConditionalOnClass classes not found: com.mongodb.Mongo (OnClassCondition)

   FallbackWebSecurityAutoConfiguration
      - SpEL expression on org.springframework.boot.autoconfigure.security.FallbackWebSecurityAutoConfiguration: !${security.basic.enabled:true} (OnExpressionCondition)

   SecurityAutoConfiguration
      - required @ConditionalOnClass classes not found: org.springframework.security.authentication.AuthenticationManager (OnClassCondition)

   EmbeddedServletContainerAutoConfiguration.EmbeddedJetty
      - required @ConditionalOnClass classes not found: org.eclipse.jetty.server.Server,org.eclipse.jetty.util.Loader (OnClassCondition)

   WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter#localeResolver
      - @ConditionalOnMissingBean (types: org.springframework.web.servlet.LocaleResolver; SearchStrategy: all) found no beans (OnBeanCondition)
      - SpEL expression: '${spring.mvc.locale:}' != '' (OnExpressionCondition)

   WebSocketAutoConfiguration
      - required @ConditionalOnClass classes not found: org.springframework.web.socket.WebSocketHandler,org.apache.tomcat.websocket.server.WsSci (OnClassCondition)
```
对于每个自动配置，可以看到它启动或失败的原因。