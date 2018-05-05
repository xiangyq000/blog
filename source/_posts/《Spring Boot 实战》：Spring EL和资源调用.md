---
title: '《Spring Boot 实战》: Spring EL 和资源调用'
date: 2017-05-04 01:12:22
tags: Spring Boot
categories: [读书笔记, Spring Boot]
---
Spring EL 也就是 Spring 表达式语言，支持在 xml 和注解中使用表达式，类似于 JSP 的 EL 表达式语言。

Spring 开发中我们可能经常涉及到调用各种资源的情况，包含普通文件、网址、配置文件、系统环境变量等，我们可以使用 Spring 的表达式语言实现资源的注入。
<!--more-->

1. 因为需要将 file 转换成字符串，我们增加 commons-io 可以简化文件的相关操作。`build.gradle` 中增加

  ```groovy
  compile group: 'commons-io', name: 'commons-io', version: '2.5'
  ```

2. resource 目录下新建 `test.txt`，内容随意。

3. resource 目录下新建新建 `test.properties` 文件，内容如下：

  ```
  project.name=SpringEL
  ```

4. 需要被注入的 bean

  ```Java
  package com.springel;

  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.stereotype.Service;

  @Service
  public class DemoService {
      @Value("DemoService类的属性")//注入字符串
      private String another;
      public String getAnother() {
          return another;
      }
      public void setAnother(String another) {
          this.another = another;
      }
  }
  ```

5. 配置类

  ```Java
  package com.springel;

  import org.apache.commons.io.IOUtils;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.ComponentScan;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.context.annotation.PropertySource;
  import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
  import org.springframework.core.env.Environment;
  import org.springframework.core.io.Resource;

  import java.io.IOException;

  @Configuration
  @ComponentScan("com.springel")
  @PropertySource("classpath:test.properties")
  public class ElConfig {

      @Value("I LOVE YOU!")//注入字符串
      private String normal;

      @Value("#{systemProperties['os.name']}")//获取操作系统名
      private String osName;

      @Value("#{ T(java.lang.Math).random() * 100.0 }")//注入表达式结果
      private double randomNumber;

      @Value("#{demoService.another}")//注入其他Bean的属性
      private String fromAnother;

      @Value("${project.name}")//注入配置文件
      private String projectName;

      @Value("classpath:cn/hncu/p2_2_2SpringEL/test.txt")
      private Resource testFile;//注意这个Resource是:org.springframework.core.io.Resource;

      @Autowired //注入配置文件
      private Environment environment;

      @Value("http://www.baidu.com")//注入网址资源
      private Resource testUrl;

      @Bean //注入配置文件
      public static PropertySourcesPlaceholderConfigurer propertyConfigurer(){
          return new PropertySourcesPlaceholderConfigurer();
      }

      public void outputResource(){
          try {
              System.out.println("normal:"+normal);
              System.out.println("osName:"+osName);
              System.out.println("randomNumber:"+randomNumber);
              System.out.println("fromAnother:"+fromAnother);
              System.out.println("projectName:"+projectName);
              System.out.println("测试文件:"+IOUtils.toString(testFile.getInputStream()));
              System.out.println("配置文件project.author:"+environment.getProperty("project.author"));
              System.out.println("网址资源:"+IOUtils.toString(testUrl.getInputStream()));
          } catch (IOException e) {
              e.printStackTrace();
          }
      }

  }
  ```
  注入配置配件需要使用 `@PropertySource` 指定文件地址，若使用 `@Value` 注入，则要配置一个 `PropertySourcesPlaceholderConfigurer` 的 Bean。 注意，`@Value("${project.name}")` 使用的是 "$" 而不是 "#"。

6. 运行类

  ```Java
  package com.springel;

  import org.springframework.context.annotation.AnnotationConfigApplicationContext;

  public class Main {

      public static void main(String[] args) {
          AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ElConfig.class);
          ElConfig resourceService = context.getBean(ElConfig.class);
          resourceService.outputResource();
          context.close();
  	}
  }
  ```

