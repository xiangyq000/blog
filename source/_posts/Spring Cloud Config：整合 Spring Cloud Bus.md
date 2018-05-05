---
title: "Spring Cloud Config：整合 Spring Cloud Bus"
date: 2017-08-10 15:28:15
tags: [Spring Cloud Config, Spring Cloud Bus, kafka, zookeeper]
categories: Spring Cloud
---

1. 安装 kafka + ZooKeeper

  ```shell
  $ brew install kafka
  ```

2. 启动 kafka 和 ZooKeeper

  ```shell
  $ zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties
  $ kafka-server-start /usr/local/etc/kafka/server.properties
  ```

3. 监听 kafka

  ```shell
  $ kafka-console-consumer --zookeeper localhost:2181
  ```

4. 创建 Spring Boot 项目 cloud-config 和 demo-server，添加 actuator 和 cloud-bus 依赖：

  build.gradle:

  ```
  compile('org.springframework.boot:spring-boot-starter-actuator')
  compile('org.springframework.cloud:spring-cloud-starter-bus-kafka')
  ```

5. 以 `spring.profile.active=native` 启动 config server

  application.properties:

  ```
  spring.application.name=cloud-config
  server.port=9086
  spring.cloud.config.server.native.search-locations=classpath:native/

  spring.cloud.stream.kafka.binder.brokers=localhost:9092
  spring.cloud.stream.kafka.binder.zkNodes=localhost:2181
  ```

6. 以 `spring.profile.active=test` 启动 demo-server

  bootstrap.properties:

  ```
  spring.application.name=demo-server
  server.port=10001
  spring.cloud.config.uri=http://localhost:9086

  spring.cloud.stream.kafka.binder.brokers=localhost:9092
  spring.cloud.stream.kafka.binder.zkNodes=localhost:2181
  ```

7. 发出刷新指令

  ```shell
  $ curl -d "" "localhost:9086/bus/refresh"
  ```

8. 查看 kafka console 输出

  ```shell
  contentType "text/plain"
  originalContentType "application/json;charset=UTF-8"{
  "type":"RefreshRemoteApplicationEvent",
  "timestamp":1505074284683,
  "originService":"cloud-config:native:9086",
  "destinationService":"**",
  "id":"9b4dd274-43a6-4636-b1f3-74fbc7f00ca5"}

  contentType "text/plain"
  originalContentType "application/json;charset=UTF-8"{
  "type":"AckRemoteApplicationEvent",
  "timestamp":1505074284745,
  "originService":"cloud-config:native:9086",
  "destinationService":"**",
  "id":"df8374db-d89e-43f2-b012-1ca9e58b785e",
  "ackId":"9b4dd274-43a6-4636-b1f3-74fbc7f00ca5",
  "ackDestinationService":"**",
  "event":"org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"}

  contentType "text/plain"
  originalContentType "application/json;charset=UTF-8"{
  "type":"AckRemoteApplicationEvent",
  "timestamp":1505074376165,
  "originService":"demo-server:test:10001",
  "destinationService":"**",
  "id":"490bd6ca-aa87-40d4-bb2b-6cd6c7b44b97",
  "ackId":"9b4dd274-43a6-4636-b1f3-74fbc7f00ca5",
  "ackDestinationService":"**",
  "event":"org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"}
  ```


9.  自己往总线上发出的 Ack 消息自己也会收到，这一点可以通过 debug 模式的日志观察到。