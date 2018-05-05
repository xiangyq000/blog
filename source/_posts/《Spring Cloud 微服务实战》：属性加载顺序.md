---
title: 《Spring Cloud 微服务实战》：属性加载顺序
date: 2017-08-30 11:18:20
tags: Spring Cloud
categories: [读书笔记, Spring Cloud]
---

为了能够更合理的重写各属性的值，Spring Boot 使用了下面这种较为特别的属性加载顺序：

1. 命令行中传入的参数。
2. `SPRING_APPLICATION_JSON` 中的属性，`SPRING_APPLICATION_JSON` 是以 JSON 格式配置在系统环境变量中的内容。
3. java：comp/env 中的 JNDI 属性。
4. java 的系统属性，可以通过 `System.getProperties()` 获取的内容操作系统的环境变量。
5. 通过 `random.*` 配置的随机属性。
6. 位于当前应用 jar 包之外，针对不同 {profile} 环境的配置文件内容。
7. 位于当前应用 jar 包之内，针对不同 {profile} 环境的配置文件内容。
8. 位于当前应用 jar 包之外的 `application.properties` 配置内容。
9. 位于当前应用 jar 包之内的 `application.properties` 配置内容。
10. 在 `@Configuration` 注解修改的类，通过 `@PropertySource` 注解定义的属性。
11. 应用默认属性，使用 `SpringApplication.setDefaultProperties` 定义的内容。
