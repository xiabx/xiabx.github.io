---
title: Eureka
author: XIA
categories:
  - SpringCloud
tags:
  - null
date: 2019-06-20 20:25:10
---

# 服务治理

服务治理是微服务系统架构中最核心和基础的模块，它主要用来实现各个微服务实例的自动化注册与发现。

**服务注册**

在服务治理框架中往往都会构建一个注册中心，每个服务单元向注册中心登记自己提供的服务，将主机与端口号、版本号等信息告知注册中心。注册中心会按照服务名分类组织服务清单。

**服务发现**

由于在服务治理框架下运行，服务间的调用不再通过指定具体的实例地址来实现，而是通过向服务名发起请求调用实现。所以服务调用方在调用服务提供方提供接口的时候，并不知道具体的服务实例位置。因此调用方需要向服务注册中心咨询服务，并获取所有服务的实例清单，以实现对具体服务实例的访问。

# Eureka简介

Spring Cloud Eureka，使用Netflix Eureka来实现服务注册与发现，它既包含了服务端组件，也包含了客户端组件。

Eureka服务端，我们也称之为服务注册中心，它同其他服务注册中心一样，支持高可用配置。当Eureka以集群方式部署时，当集群中有分片出现故障时，Eureka会进入自我保护模式。它允许在分片故障期间继续提供服务的注册和发现，当故障分片恢复时，集群中的其他分片会把他们的状态再次同步过来。

Eureka客户端，主要处理服务的注册于发现。客户端通过注册和参数配置的方式，嵌入在客户端应用代码中，在应用程序运行时，Eureka客户端向注册中心注册自己提供的服务并周期性地发送心跳来更新他的租约。同时，他也能从服务端查询当前注册地服务信息并缓存到本地，并周期性地刷新服务状态。

# Eureka应用

## 搭建服务注册中心

引入依赖：spring-cloud-starter-netflix-eureka-server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

配置application.properties

```properties
server.port=1111

eureka.instance.hostname=localhost
# 由于该应用为注册中心，所以设置为false，代表不向注册中心注册自己
eureka.client.register-with-eureka=false
# 由于注册中心地职责就是维护服务实例，它并不需要去检索服务，所以也要设置为false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```

使用@EnableEurekaServer启动服务

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaserverApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaserverApplication.class, args);
    }

}
```

然后访问：`http://localhost1111/eureka/`就可以看到服务注册中心地面板

## 注册服务提供者

添加客户端依赖：spring-cloud-starter-netflix-eureka-client

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

配置Application.properties

```properties
# 服务名，在服务注册中心中显示
spring.application.name=eureka-client-1
# 配置服务注册中心地址
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```

使用@EnableDiscoveryClient注解启动服务

```java
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClient1Application {

    public static void main(String[] args) {
        SpringApplication.run(EurekaClient1Application.class, args);
    }

}
```

**@EnableDiscoveryClient与@EnableEurekaClient区别：**

>@EnableDiscoveryClient基于spring-cloud-commons,@EnableEurekaClient基于spring-cloud-netflix。其实用更简单的话来说，就是如果选用的注册中心是eureka，那么就推荐@EnableEurekaClient，如果是其他的注册中心，那么推荐使用@EnableDiscoveryClient。

## 高可用注册中心

Eureka Server对高可用有很好的支持。Eureka服务治理设计中，所有节点既是服务提供方，也是服务消费方，服务注册中心也不例外。

Eureka Server的高可用实际上就是将自己作为服务向其他服务注册中心注册自己，这样就可以形成一组相互注册的服务注册中心，以实现服务清单的相互同步，达到高可用的效果。

----

**构建简单的高可用注册中心：**

创建两个properties文件，serviceUrl相互指向：

```properties
spring.application.name=eureka-server-1
server.port=1112

eureka.instance.hostname=peer1

eureka.client.serviceUrl.defaultZone=http://peer2:1113/eureka/
```

```properties
spring.application.name=eureka-server-2
server.port=1113

eureka.instance.hostname=peer2

eureka.client.serviceUrl.defaultZone=http://peer1:1112/eureka/
```

hosts文件：

```
127.0.0.1       peer1
127.0.0.1       peer2
```

然后打包通过`java -jar eurekaserver-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer1 `来启动两个服务。之后这两个注册中心的服务就可以相互注册，相互复制共享服务实例列表。

----

**将服务注册到高可用集群**

只需要设置defaultZone指向多个服务注册中心即可。

```properties
eureka.client.serviceUrl.defaultZone=http://localhost:1113/eureka/,http://localhost:1112/eureka/
```

## 服务发现与消费

现在我们已经有了服务注册中心和服务提供者，接下来就是构建服务消费者。他有两个目标：发现服务和服务消费。其中服务发现由Eureka客户端实现，服务消费则是由Ribbon完成。

创建消费服务，引入依赖eureka-client与ribbon：

```properties
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <version>2.2.2.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
            <version>2.2.2.RELEASE</version>
        </dependency>

```

启动eureka-client服务与注册RestTemplate：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsumerApplication {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }

}
```

为该服务指定defaultZone：

```properties
server.port=9001
spring.application.name=consumer

eureka.client.serviceUrl.defaultZone=http://localhost:1113/eureka/,http://localhost:1112/eureka/
```

然后就是使用restTemplate通过ribbon使用服务名访问服务了：

```java
@RestController
public class ConsumerController {
    @Autowired
    RestTemplate restTemplate;

    @GetMapping("/hello")
    public String helloConsumer(){
        return restTemplate.getForEntity("http://eureka-client-1/hello",String.class).getBody();
    }


}
```

# 服务治理机制

Eureka各个服务之间通过Rest通信的方式相互协作运行起来。如下图，其中有这样几个重要的元素：

+ 服务注册中心-1和服务注册中心-2，他们相互注册成为高可用集群
+ 服务提供者启动了两个实例，一个注册到服务注册中心-1，一个注册到服务注册中心-2.
+ 两个服务消费者，他们也都分别指向了一个注册中心

![image-20200321131636103](https://xbxblog2.bj.bcebos.com/Eureka%2Fimage-20200321131636103.png)

下面从服务提供者、服务消费者、服务注册中心三个服务的角度来归纳他们的通信行为。

## 服务提供者

**服务注册**

“服务提供者“启动的时候会通过Rest请求的方式将自己注册到Eureka Server上，同时带上了自身服务的一些元数据信息。

Eureka Server在收到这个Rest请求后，将元数据信息存储在双层Map中，其中该双层Map的第一层key为服务名，第二层key为具体服务的实例名。这时因为同一个服务名的服务启动两次，会有不同的实例。

![image-20200321141623630](https://xbxblog2.bj.bcebos.com/Eureka%2Fimage-20200321141623630.png)

`eureka.client.register-with-eureka=true`该参数为true时才表示会将服务注册到服务注册中心，为false则不进行注册。

-------

**服务同步**

通过以上架构图所示，这里两个服务提供者分别注册到两个不同的服务注册中心上。但是这两个服务注册中心互相注册为服务，当服务提供者发送一个注册请求时，它会将请求转发给集群中其他注册中心，从而实现注册中心之间的服务同步。

------

**服务续约**

在服务注册到服务注册中心后，服务提供这会维护一个心跳来告诉Eureka Server自身存活情况，以防止Eureka Server的”剔除任务“将该服务实例从服务列表中排除出去。

服务续约有两个重要属性：

```properties
#表示eureka client发送心跳给server端的频率。
eureka.instance.lease-renewal-interval-in-seconds=30
#表示eureka server至上一次收到client的心跳之后，等待下一次心跳的超时时间，在这个时间内若没收到下一次心跳，则将移除该instance。
eureka.instance.lease-expiratiion-duration-in-seconds=90
```

## 服务消费者

**获取服务**

假设我们的服务注册中心注册了一个服务，这个服务有两个服务实例。当我们启动服务消费者时，它会发送Rest请求给注册中心，来获取注册中心上的服务清单。服务注册中心出于性能考虑，他会在内部维护一个服务只读的服务清单来返回给客户端，同时该清单每30秒会刷新一次。

消费者要获取注册中心上服务列表的前提是保证配置文件中`eureka.client.fetch-registry=true`。`eureka.client.registry-fetch-interval-seconds=30`表示消费者每隔30秒获取一次注册中心的服务清单。

-----

**服务调用**

在服务消费者获取到服务清单后，通过服务名可以获得具体提供服务的实例名和服务实例的元数据信息。通过这些服务实例的信息，客户端就可以根据自己的需求调用具体服务实例。

----

**服务下线**

在系统运行过程中，若有一个服务提供者实例需要临时关闭，那么他会将下线信息发送给Eureka Server，Eureka Server然后会将该服务实例的下线信息传播下去。

## 服务注册中心

**失效剔除**

服务注册中心会维护一个定时任务，默认会每隔一段时间（默认为60秒）将服务清单中超时（默认为90秒）没有续约的服务剔除。

-----

**自我保护**

![image-20200321145304064](https://xbxblog2.bj.bcebos.com/Eureka%2Fimage-20200321145304064.png)

Eureka Server在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%，如果出现低于情况，Eureka Server会将当前实例注册信息保护起来，让这些实例不会过期，但是这样就会出现客户端拿到已经不存在的实例，会出现调用失败的情况，所以就需要使用容错机制如：断路器和请求重试等。

由于在本地调试时很容易出现自我保护机制，这会使得注册中心维护的实例不准确。所以在本地开发时可以使用`eureka.server.enable-self-presevation=false`来关闭自我保护机制。

# 相关源码分析

在spring-cloud-netflix-eurka-client包下的spring.factories文件中，定义了EurekaDiscoveryClient自动装配类。在EurekaDiscoveryClientConfiguration中定义了EurekaDiscoveryClient的Bean。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=
...........
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration,\
```

EurekaDiscoveryClient类结构：

![image-20200321171346863](https://xbxblog2.bj.bcebos.com/Eureka%2Fimage-20200321171346863.png)

其中左边的DiscoveryClient是SpringCloud的接口，它定义了用来实现服务发现的抽象方法，通过该接口可以屏蔽服务治理的实现细节，所以使用SpringCloud构建的微服务框架在切换不同的服务治理框架时不需要更改程序，只需要针对另一种服务框架进行必要的配置即可。

EurekaDiscoveryClient是对SpringCloud提供的DiscoveryClient接口的实现，它实现的是对Eureka服务发现的封装。所以EurekaDiscoveryClient依赖于Netflix Eureka的EurekaClient接口。EurekaClient继承LookupService接口，主要定义了针对Eureka的服务发现的抽象方法，而真正实现服务发现的则是Netflix Eureka中的DiscoveryClent类。

## 解析配置属性

先了解两个Eureka中的概念：Region和Zone。

+ region：可以简单理解为地理上的分区，比如亚洲地区，或者华北地区，再或者北京等等，没有具体大小的限制。根据项目具体的情况，可以自行合理划分region。

+ zone：可以简单理解为region内的具体机房，比如说region划分为北京，然后北京有两个机房，就可以在此region之下划分出zone1,zone2两个zone。

参考：https://www.cnblogs.com/junjiang3/p/9061867.html

在EndpointUtils类的getServiceUrlsMapFromConfig方法解析配置文件中的zone相关配置。

方法的调用Eureka项目下的DiscoveryClient类的构造方法中。

![image-20200321200405421](https://xbxblog2.bj.bcebos.com/Eureka%2Fimage-20200321200405421.png)

**EndpointUtils#getServiceUrlsMapFromConfig**

```java
public static Map<String, List<String>> getServiceUrlsMapFromConfig(EurekaClientConfig clientConfig, String instanceZone, boolean preferSameZone) {
        Map<String, List<String>> orderedUrls = new LinkedHashMap<>();
        String region = getRegion(clientConfig);
        String[] availZones = clientConfig.getAvailabilityZones(clientConfig.getRegion());
        if (availZones == null || availZones.length == 0) {
            availZones = new String[1];
            availZones[0] = DEFAULT_ZONE;
        }

        int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);

        String zone = availZones[myZoneOffset];
        List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
        if (serviceUrls != null) {
            orderedUrls.put(zone, serviceUrls);
        }
    
    	//....................
    
        return orderedUrls;
    }
```

由上面代码可以发现，客户端依次加载了两个内容：Region和Zone。

通过` getRegion(clientConfig);`用来获取Region，每个微服务只可以属于一个Region，如果没有特别配置，则默认返回default，若自己配置则可以通过`eureka.client.region`属性定义。

```java
public static String getRegion(EurekaClientConfig clientConfig) {
    String region = clientConfig.getRegion();
    if (region == null) {
        region = DEFAULT_REGION; //default
    }
    region = region.trim().toLowerCase();
    return region;
}
```

通过`clientConfig.getAvailabilityZones(clientConfig.getRegion());`可以获取Zone，当我们没有特别定义时，将默认使用defaultZone。如果需要特别指定则使用`eureka.client.availability-zones`属性进行配置。Zone可以设置多个，通过”，“进行分割。

```java
public String[] getAvailabilityZones(String region) {
    String value = this.availabilityZones.get(region);
    if (value == null) {
        value = DEFAULT_ZONE;
    }
    return value.split(",");
}
```

在获取了Region和Zone后，接下来就是ServiceUrls。解析方法位于：`EurekaClientConfigBean#getEurekaServerServiceUrls`

```java
public List<String> getEurekaServerServiceUrls(String myZone) {
    String serviceUrls = this.serviceUrl.get(myZone);
    if (serviceUrls == null || serviceUrls.isEmpty()) {
        serviceUrls = this.serviceUrl.get(DEFAULT_ZONE);
    }
    if (!StringUtils.isEmpty(serviceUrls)) {
        final String[] serviceUrlsSplit = StringUtils
            .commaDelimitedListToStringArray(serviceUrls);
        List<String> eurekaServiceUrls = new ArrayList<>(serviceUrlsSplit.length);
        for (String eurekaServiceUrl : serviceUrlsSplit) {
            if (!endsWithSlash(eurekaServiceUrl)) {
                eurekaServiceUrl += "/";
            }
            eurekaServiceUrls.add(eurekaServiceUrl.trim());
        }
        return eurekaServiceUrls;
    }

    return new ArrayList<>();
}
```

## 服务注册

服务注册的调用入口位于DiscoveryClient构造方法中。

```java
private void initScheduledTasks() {

    // ................................
    
    // InstanceInfo replicator
    instanceInfoReplicator = new InstanceInfoReplicator(
        this,
        instanceInfo,
        clientConfig.getInstanceInfoReplicationIntervalSeconds(),
        2); // burstSize

    // ................................
    instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());

}
```

InstanceInfoReplicator为Runnable子类。其主要方法在run中。

```java
public void run() {
    try {
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            //注册
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

然后在register方法：

```java
boolean register() throws Throwable {
    EurekaHttpResponse<Void> httpResponse;
	//通过Rest方式，发送请求来注册。instanceInfo为服务元数据
    httpResponse = eurekaTransport.registrationClient.register(instanceInfo);

    return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
}
```

## 服务获取与服务续约

在DiscoveryClient#initScheduledTasks中，除了会启动一个用于服务注册的任务外，还启动了服务获取与服务续约任务。

```java
//服务获取任务
if (clientConfig.shouldFetchRegistry()) {
    // registry cache refresh timer
    int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
    int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
    cacheRefreshTask = new TimedSupervisorTask(
        "cacheRefresh",
        scheduler,
        cacheRefreshExecutor,
        registryFetchIntervalSeconds,
        TimeUnit.SECONDS,
        expBackOffBound,
        new CacheRefreshThread()
    );
    scheduler.schedule(
        cacheRefreshTask,
        registryFetchIntervalSeconds, TimeUnit.SECONDS);
}
```

```java
//心跳任务
if (clientConfig.shouldRegisterWithEureka()) {
    int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
    int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
    logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

    // Heartbeat timer
    heartbeatTask = new TimedSupervisorTask(
        "heartbeat",
        scheduler,
        heartbeatExecutor,
        renewalIntervalInSecs,
        TimeUnit.SECONDS,
        expBackOffBound,
        new HeartbeatThread()
    );
    scheduler.schedule(
        heartbeatTask,
        renewalIntervalInSecs, TimeUnit.SECONDS);

    //。。。。。。。。。。
}
```

这些任务也都是通过Rest请求的方式与server通信。

