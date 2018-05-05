---
title: "Spring Cloud Eureka：配置项"
date: 2017-08-04 15:28:15
tags: [Spring Cloud Eureka]
categories: Spring Cloud
---
### 1. 服务端配置: 
org.springframework.cloud.netflix.eureka.server.EurekaServerConfigBean

| 配置参数                                   | 默认值   | 说明                                       |
| -------------------------------------- | ----- | ---------------------------------------- |
| eureka.server.enable-self-preservation | false | 关闭注册中心的保护机制，Eureka 会统计15分钟之内心跳失败的比例低于85%将会触发保护机制，不剔除服务提供者，如果关闭服务注册中心将不可用的实例正确剔除 |
<!--more-->

### 2. 客户端实例相关配置
org.springframework.cloud.netflix.eureka.EurekaInstanceConfigBean

| 配置参数                                     | 默认值     | 说明                                       |
| ---------------------------------------- | ------- | ---------------------------------------- |
| eureka.instance.prefer-ip-address        | false   | 不使用主机名来定义注册中心的地址，而使用IP地址的形式，如果设置了eureka.instance.ip-address 属性，则使用该属性配置的IP，否则自动获取除环路IP外的第一个IP地址 |
| eureka.instance.ip-address               |         | IP地址                                     |
| eureka.instance.hostname                 |         | 设置当前实例的主机名称                              |
| eureka.instance.appname                  |         | 服务名，默认取 spring.application.name 配置值，如果没有则为 unknown |
| eureka.instance.lease-renewal-interval-in-seconds | 30      | 定义服务续约任务（心跳）的调用间隔，单位：秒                   |
| eureka.instance.lease-expiration-duration-in-seconds | 90      | 定义服务失效的时间，单位：秒                           |
| eureka.instance.status-page-url-path     | /info   | 状态页面的URL，相对路径，默认使用HTTP访问，如果需要使用 HTTPS则需要使用绝对路径配置 |
| eureka.instance.status-page-url          |         | 状态页面的URL，绝对路径                            |
| eureka.instance.health-check-url-path    | /health | 健康检查页面的URL，相对路径，默认使用HTTP访问，如果需要使用HTTPS则需要使用绝对路径配置 |
|eureka.instance.health-check-url| |健康检查页面的URL，绝对路径

### 3. 客户端注册相关配置: 
org.springframework.cloud.netflix.eureka.EurekaClientConfigBean

| 配置参数| 默认值| 说明|
| --- | --- | --- |
|eureka.client.service-url| |指定服务注册中心地址，类型为 HashMap，并设置有一组默认值，默认的Key为 defaultZone；默认的Value为http://localhost:8761/eureka，如果服务注册中心为高可用集群时，多个注册中心地址以逗号分隔。如果服务注册中心加入了安全验证，这里配置的地址格式为：http://<username>:<password>@localhost:8761/eureka，其中`<username>`为安全校验的用户名；`<password>`为该用户的密码
|eureka.client.fetch-registery|true|检索服务|
|eureka.client.registery-fetch-interval-seconds|30|从Eureka服务器端获取注册信息的间隔时间，单位：秒|
|eureka.client.register-with-eureka|true|启动服务注册|
|eureka.client.eureka-server-connect-timeout-seconds|5|连接Eureka Server的超时时间，单位：秒|
|eureka.client.eureka-server-read-timeout-seconds|8|读取 Eureka Server 信息的超时时间，单位：秒|
|eureka.client.filter-only-up-instances|true|获取实例时是否过滤，只保留UP状态的实例|
|eureka.client.eureka-connection-idle-timeout-seconds|30|Eureka 服务端连接空闲关闭时间，单位：秒|
|eureka.client.eureka-server-total-connections|200|从Eureka 客户端到所有Eureka服务端的连接总数|
|eureka.client.eureka-server-total-connections-per-host|50|从Eureka客户端到每个Eureka服务主机的连接总数|