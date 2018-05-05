---
title: 'Spring Cloud Config: 基于服务发现的客户端启动过程'
date: 2017-08-13 10:56:06
tags: [Spring Cloud Config]
categories: Spring Cloud
---

当Config Server已注册到Spring Cloud Eureka上时，想从Config Server获取配置信息的Client可以通过从Eureka获得服务注册信息，动态发现Config Server实例。
<!--more-->

### 1. Config Client配置

1.  在`application.properties`中配置如下项：

  ```
  spring.application.name=demo-server
  server.port=10001
  ```

2. 在`bootstrap-test.properties`中配置如下项：

  ```
  spring.cloud.config.discovery.enabled=true
  spring.cloud.config.discovery.service-id=cloud-config
  spring.cloud.config.fail-fast=true
  spring.cloud.config.profile=test
  eureka.client.serviceUrl.defaultZone = http://localhost:9090/eureka/
  ```
  启动时设置`spring.profile.active=test`。

  也可以用`bootstrap.properties`并采用default的profile。

3. Config Server运行在localhost:9086，eureka server运行在localhost:9090。Config server已以服务名cloud-config注册到eureka server。Config Server使用本地文件作为仓库。

### 2. Config Client处理流程：

1. 构建bootstrap的context

  Spring Cloud Commons的说明：

  > A Spring Cloud application operates by creating a "bootstrap" context, which is a parent context for the main application. Out of the box it is responsible for loading configuration properties from the external sources, and also decrypting properties in the local external configuration files. The two contexts share an Environment which is the source of external properties for any Spring application. Bootstrap properties are added with high precedence, so they cannot be overridden by local configuration, by default.
  >

  根据`org.springframework.cloud.bootstrap.BootstrapApplicationListener`的类说明：

  >  A listener that prepares a SpringApplication (e.g. populating its Environment) by delegating to {@link ApplicationContextInitializer} beans in a separate bootstrap context. The bootstrap context is a SpringApplication created from sources defined in spring.factories as {@link BootstrapConfiguration}, and initialized with external config taken from "bootstrap.properties" (or yml), instead of the normal "application.properties".
  >

  Spring Cloud会根据`boostrap.properties`及`boostrap-{profile}.properties`的配置构建单独的BootStrapContext。这个BootStrapContext是Main Application的父Context，其配置属性可以被子Context获取。

2. 构建`CompositePropertySource`

  根据Spring-Cloud-Commons的文档：

  > "bootstrap": an optional CompositePropertySource appears with high priority if any PropertySourceLocators are found in the Bootstrap context, and they have non-empty properties. An example would be properties from the Spring Cloud Config Server. See below for instructions on how to customize the contents of this property source.
  >
  > ​
  >
  > "applicationConfig: [classpath:bootstrap.yml]" (and friends if Spring profiles are active). If you have a bootstrap.yml (or properties) then those properties are used to configure the Bootstrap context, and then they get added to the child context when its parent is set. They have lower precedence than the application.yml (or properties) and any other property sources that are added to the child as a normal part of the process of creating a Spring Boot application. See below for instructions on how to customize the contents of these property sources.
  >

  注意，Bootstrap获取到的远程配置具有高优先级，但`bootstrap.properties`里的配置项本身是低优先级。

  在Spring Cloud Config项目中配置了`ConfigServicePropertySourceLocator`，在BootstrapContext构建阶段，它从`bootstrap.properties`中拿到`eureka.client.serviceUrl.defaultZone`，并访问eureka server，请求获取服务注册信息。在config client的console输出中可以看到：

  ```
  2017-09-07 15:59:53.704  INFO [bootstrap,,,] 77988 --- [  restartedMain] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
  2017-09-07 15:59:53.736  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
  2017-09-07 15:59:53.736  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
  2017-09-07 15:59:53.736  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
  2017-09-07 15:59:53.736  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Application is null : false
  2017-09-07 15:59:53.736  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
  2017-09-07 15:59:53.737  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
  2017-09-07 15:59:53.737  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
  2017-09-07 15:59:53.964  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : The response status is 200
  2017-09-07 15:59:53.966  INFO [bootstrap,,,] 77988 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Not registering with Eureka server per configuration
  ```
  与此对应的eureka server的log：

  ```
  2017-09-07 15:59:53.882 DEBUG [http-nio-9090-exec-6] org.apache.coyote.http11.Http11InputBuffer: Received [GET /eu
  reka/apps/ HTTP/1.1
  Accept: application/json
  DiscoveryIdentity-Name: DefaultClient
  DiscoveryIdentity-Version: 1.4
  DiscoveryIdentity-Id: 10.236.19.51
  Accept-Encoding: gzip
  Host: localhost:9090
  Connection: Keep-Alive
  User-Agent: Java-EurekaClient/v1.6.2

  ]
  ```
  可以看到Bootstrap阶段config client并不向eureka server注册自己，仅是获取服务列表，并从中查找config server。Bootstrap Context的工作到此结束。

3. 从config server获取配置

  config client日志：

  ```
  2017-09-07 16:00:08.855 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.c.e.PropertySourcesPropertyResolver  : Could not find key 'spring.profiles.default' in any property source
  2017-09-07 16:00:08.856 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.retry.support.RetryTemplate          : Retry: count=0
  2017-09-07 16:00:08.856 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.c.e.PropertySourcesPropertyResolver  : Could not find key 'spring.application.name:application' in any property source
  2017-09-07 16:00:08.856 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.c.e.PropertySourcesPropertyResolver  : Found key 'spring.application.name' in [applicationConfig: [classpath:/application.properties]] with type [String]
  2017-09-07 16:00:08.856 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.c.e.PropertySourcesPropertyResolver  : Could not find key 'spring.cloud.config.name:demo-server' in any property source
  2017-09-07 16:00:08.856 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.c.e.PropertySourcesPropertyResolver  : Could not find key 'spring.cloud.config.name' in any property source
  2017-09-07 16:00:08.856 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.c.e.PropertySourcesPropertyResolver  : Found key 'spring.cloud.config.profile' in [applicationConfig: [classpath:/bootstrap-test.properties]] with type [String]
  2017-09-07 16:00:08.862  INFO [demo-server,,,] 77988 --- [restartedMain] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://localhost:9086/
  2017-09-07 16:00:08.862 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.web.client.RestTemplate              : Created GET request for "http://localhost:9086/demo-server/test"
  2017-09-07 16:00:08.865 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.web.client.RestTemplate              : Setting request Accept header to [application/json, application/*+json]
  2017-09-07 16:00:08.866 DEBUG [demo-server,,,] 77988 --- [restartedMain] s.n.www.protocol.http.HttpURLConnection  : sun.net.www.MessageHeader@a83b1655 pairs: {GET /demo-server/test HTTP/1.1: null}{Accept: application/json, application/*+json}{User-Agent: Java/1.8.0_91}{Host: localhost:9086}{Connection: keep-alive}
  2017-09-07 16:00:08.997 DEBUG [demo-server,,,] 77988 --- [restartedMain] s.n.www.protocol.http.HttpURLConnection  : sun.net.www.MessageHeader@231715a35 pairs: {null: HTTP/1.1 200}{X-Application-Context: cloud-config:native:9086}{Content-Type: application/json;charset=UTF-8}{Transfer-Encoding: chunked}{Date: Thu, 07 Sep 2017 08:00:08 GMT}
  2017-09-07 16:00:08.998 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.web.client.RestTemplate              : GET request for "http://localhost:9086/demo-server/test" resulted in 200 (null)
  2017-09-07 16:00:08.998 DEBUG [demo-server,,,] 77988 --- [restartedMain] o.s.web.client.RestTemplate              : Reading [class org.springframework.cloud.config.environment.Environment] as "application/json;charset=UTF-8" using [org.springframework.http.converter.json.MappingJackson2HttpMessageConverter@7044a1c0]
  2017-09-07 16:00:08.999  INFO [demo-server,,,] 77988 --- [restartedMain] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=demo-server, profiles=[test], label=null, version=null, state=null
  2017-09-07 16:00:08.999  INFO [demo-server,,,] 77988 --- [restartedMain] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource@1198162374 [name='classpath:native/demo-server-test.properties', properties={test-key=test-value}]]]
  ```

  此时已经到Main Application Context构建阶段，作为Bootstrap Context的子Context，它可以从`application.properties`，`application-{profile}.properties`，`bootstrap.properties`，`bootstrap-{profile}.properties`和System properties中查找访问config server所需的application name，label，profile等属性。

  Moreover，config server可能配置了basic authentication。在这种情况下，config server需要在向eureka server注册的metadataMap中上传user和password参数，以供config client访问。这里的源码如下：
   `org.springframework.cloud.config.client.DiscoveryClientConfigServiceBootstrapConfiguration`

  ```Java
  @ConditionalOnProperty(value = "spring.cloud.config.discovery.enabled", matchIfMissing = false)
  @Configuration
  @Import({ UtilAutoConfiguration.class })
  @EnableDiscoveryClient
  public class DiscoveryClientConfigServiceBootstrapConfiguration {

  	private static Log logger = LogFactory
  			.getLog(DiscoveryClientConfigServiceBootstrapConfiguration.class);

  	@Autowired
  	private ConfigClientProperties config;

  	@Autowired
  	private DiscoveryClient client;

  	private HeartbeatMonitor monitor = new HeartbeatMonitor();

  	@EventListener(ContextRefreshedEvent.class)
  	public void startup(ContextRefreshedEvent event) {
  		refresh();
  	}

  	@EventListener(HeartbeatEvent.class)
  	public void heartbeat(HeartbeatEvent event) {
  		if (monitor.update(event.getValue())) {
  			refresh();
  		}
  	}

  	private void refresh() {
  		try {
  			logger.debug("Locating configserver via discovery");
  			String serviceId = this.config.getDiscovery().getServiceId();
  			List<ServiceInstance> instances = this.client.getInstances(serviceId);
  			if (instances.isEmpty()) {
  				logger.warn("No instances found of configserver (" + serviceId + ")");
  				return;
  			}
  			ServiceInstance server = instances.get(0);
  			String url = getHomePage(server);
  			if (server.getMetadata().containsKey("password")) {
  				String user = server.getMetadata().get("user");
  				user = user == null ? "user" : user;
  				this.config.setUsername(user);
  				String password = server.getMetadata().get("password");
  				this.config.setPassword(password);
  			}
  			if (server.getMetadata().containsKey("configPath")) {
  				String path = server.getMetadata().get("configPath");
  				if (url.endsWith("/") && path.startsWith("/")) {
  					url = url.substring(0, url.length() - 1);
  				}
  				url = url + path;
  			}
  			this.config.setUri(url);
  		}
  		catch (Exception ex) {
  			logger.warn("Could not locate configserver via discovery", ex);
  		}
  	}

  	private String getHomePage(ServiceInstance server) {
  		return server.getUri().toString() + "/";
  	}

  }
  ```

4. 向eureka server注册

  - client log：

  ```
  2017-09-07 15:59:53.882 DEBUG [http-nio-9090-exec-6] org.apache.coyote.http11.Http11InputBuffer: Received [GET /eu
  reka/apps/ HTTP/1.1
  Accept: application/json
  DiscoveryIdentity-Name: DefaultClient
  DiscoveryIdentity-Version: 1.4
  DiscoveryIdentity-Id: 10.236.19.51
  Accept-Encoding: gzip
  Host: localhost:9090
  Connection: Keep-Alive
  User-Agent: Java-EurekaClient/v1.6.2

  ]
  ```
  - eureka server log:

  ```
  2017-09-07 16:00:13.111 DEBUG [http-nio-9090-exec-7] org.apache.coyote.http11.Http11InputBuffer: Received [PUT /eureka/apps/DEMO-SERVER/localhost:10001:demo-server?status=UP&lastDirtyTimestamp=1504771208220 HTTP/1.1
  DiscoveryIdentity-Name: DefaultClient
  DiscoveryIdentity-Version: 1.4
  DiscoveryIdentity-Id: 10.236.19.51
  Accept-Encoding: gzip
  Content-Length: 0
  Host: localhost:9090
  Connection: Keep-Alive
  User-Agent: Java-EurekaClient/v1.6.2

  ]
  ```

  这个过程甚至发生在config client向config server请求之后。

### 3. 一个脑洞

假如我在config server的远程配置中配置另一个地址作为`eureka.client.serviceUrl.defaultZone`，config client获取到配置后会怎么办呢？

1. 仓库修改配置，重启config server。访问http://localhost:9086/demo-server-test.properties得到

  ```
  eureka.client.serviceUrl.defaultZone: http://localhost:9091/eureka/
  ```

  实际上eureka server仍然运行在localhost:9090。

2. 启动Demo-server。正确地从eureka server拿到了服务注册信息，然后从config server更新了配置：

  ```
  2017-09-07 17:06:26.782  INFO [demo-server,,,] 78952 --- [  restartedMain] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource@1069590480 [name='classpath:native/demo-server-test.properties', properties={eureka.client.serviceUrl.defaultZone=http://localhost:9091/eureka/}]]]
  ```

  但此后无法注册到eureka server：

  ```
  2017-09-07 17:10:17.698  WARN [demo-server,,,] 78952 --- [tbeatExecutor-0] c.n.d.s.t.d.RetryableEurekaHttpClient    : Request execution failed with message: java.net.ConnectException: Connection refused
  2017-09-07 17:10:17.698 ERROR [demo-server,,,] 78952 --- [tbeatExecutor-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_DEMO-SERVER/localhost:10001:demo-server - was unable to send heartbeat!

  com.netflix.discovery.shared.transport.TransportException: Cannot execute request on any known server
  ```

  其原因在于`org.springframework.cloud.config.client.ConfigServicePropertySourceLocator`注解了`@Order(0)`，会首先执行并更新配置，client注册eureka server时，会使用从config server拿到的更新后的地址。

### 4. 另一个脑洞
如果我是在Demo-Server启动并连接上eureka server后再修改config server里配置的地址呢？

这种情况下config server的仓库需要配置为git等外部仓库，push到仓库后以post访问demo-server的/refresh endpoint，则之后log里抛出无法连接的异常。

那么以这种方式，应该也同样可以动态配置其他参数。

有一点比较特殊，在这个例子中，修改的是eureka server的注册地址，且config client使用的是基于服务化的查找方式。那么即使我们此后revert掉git仓库的修改，并对demo-server发起refresh请求，由于demo-server无法连接到eureka-server，那么自然也就无法查找config server并获取更新后的配置了。若client是基于uri的形式配置的config server，则可以刷新配置。