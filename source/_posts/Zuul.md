---
title: Zuul
author: XIA
categories:
  - null
tags:
  - null
date: 2020-04-11 19:41:16
---

# Zuul是什么

在使用微服务构建系统时，往往需要提供多个用于对外暴露接口的边缘服务实例，然后通过像Nginx这种服务进行负载均衡处理。这种方式虽然是可行的，但是当边缘服务发生变化，如变更ip、增减实例时则需要手动修改nginx配置。这使得维护项目的成本变大，同时也增加了出错的可能性。

对于开发人员来说，在微服务开发中，往往每个微服务由不同的开发人员负责，而服务间往往需要像用户登录状态校验这种校验工作。如果每个微服务系统都实现一套这种校验服务显然不可取，而把校验服务单独抽取出来作为一个独立的微服务供其他模块调用也不是最好的选择。

API网关的出现就是为了解决这些问题。Spring Cloud Zuul就是一种API网关的实现。他就像一座立在各个微服务之前的“大门”，所有外部对微服务的访问都要经过这个“大门”。它通过与Eureka结合解决了路由规则与服务实例维护的问题。Zuul默认将服务名作为ContextPath的来创建路由映射。对于像登录校验这类的服务，Zuul通过自己的过滤器机制来完成。

总的来说Zuul提供了两个主要功能：路由、过滤。

# 构建Zuul服务

Zuul服务像Eureka Server是一个独立的微服务。

**创建SpringBoot应用，引入依赖：**

spring-cloud-starter-netflix-zuul包括了spring-cloud-starter-netflix-ribbon、spring-cloud-starter-netflix-hystrix、spring-boot-starter-actuator等依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

**创建应用主类，使用@EnableZuulProxy注解开启Zuul的API网关服务：**

```java
@EnableZuulProxy
@SpringBootApplication
public class BackendApigwZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(BackendApigwZuulApplication.class, args);
    }

}
```

**配置文件：**

```yaml
server:
  port: 8080

eureka:
  client:
    service-url:
      defaultZone: http://localhost:1111/eureka/
```

# 路由

路由配置由两种方式：通过手动指定服务url的传统配置方式、基于服务发现的服务路由配置。

## 传统路由配置

这种配置方式不依赖于服务发现机制。

**单实例配置：**

通过`zuul.routes.<route>.path`与`zuul.routes.<route>.path`搭配的方式配置。其中`<route>`表示一条路由规则，可以理解为一个标识符吧。

```yaml
zuul:
  routes:
    user-service:
      path: /userapi/**
      url: http://localhost:8081/
```

**多实例配置：**

通过`zuul.routes.<route>.path`与`zuul.routes.<route>.serviceId`参数对的方式配置。这里需要用到Ribbon，由于Zuul已经默认集成了Ribbon，所以我们只需要简单的配置即可。

```yaml
zuul:
  routes:
    user-service:
      path: /userapi/**
      serviceId: user-service
      
ribbon:
  eureka:
    enabled: false
    
user-service:
  ribbon:
    listOfServers: http://localhost:8081,http://localhost:8082
```

## 服务路由配置

通过Zuul与Eureka的整合，可以实现对服务实例的自动化维护。我们将Zuul注册到Eureka服务上，这样Zuul服务就可以拉取Eureka上注册的服务。在配置路由规则时只需指定serviceId即可。

```yaml
zuul:
  routes:
    meetingfilm-user:
      path: /userapi/**
      serviceId: user-service
```

这样在访问路径为/userapi/**时，就会路由到eureka中serviceId为user-service的服务。

当不显式手动配置路由规则时，Zuul会使用默认路由规则，会使用服务名作为外部请求的前缀，如`/order-service/**`则会默认路由到serviceId为order-service的服务。

路由路径匹配规则采用Ant规则，简单来说他一共有三种通配符：

+ `?`:匹配任意单个字符
+ `*`:匹配任意数量的字符
+ `**`:匹配任意数量的字符，支持多级目录

除了这些还可以配置：忽略表达式、路由前缀、本地跳转等。

默认情况下，Zuul在请求路由时，会过滤掉HTTP请求头信息中的一些敏感信息，防止他们传递到下游的外部服务器。默认的敏感头通过`zuul.sensitiveHeaders`定义，包括Cookie、set-Cookie、Authorization三个属性。如果需要向下游服务器传递这些信息可以使用`zuul.sensitiveHeaders=`设置为空来覆盖默认值。



Zuul默认集成了Ribbon与Hystrix所以可以直接在配置文件中对Ribbon与Hystrix进行配置。

# 过滤器

当我们需要对外部请求进行校验等功能时，这时就可以使用Zuul提供的过滤器。实际上路由功能的请求映射、请求转发都是通过不同的过滤器实现的。其中，路由映射主要通过pre类型过滤器完成，它将请求路径与配置的路由规则进行匹配，找到需要转发的目标地址。而转发的部分由route类型的过滤器完成，对pre类型过滤器获得的路由地址进行转发。

在Zuul中实现过滤器需要包含4个基本特征：过滤类型、执行顺序、执行条件、具体操作。这些条件就是在ZuulFilter的四个抽象方法。

```java
    public String filterType();
    public int filterOrder();
    public boolean shouldFilter();
    public Object run();
```

+ filterType：返回的字符串由4种类型，表示4种生命周期阶段。
  + pre：在请求被路由之前调用
  + routing：在路由请求时被调用
  + post：在routing和error过滤器之后被调用
  + error：处理请求时发生错误时被调用
+ filterOrder：通过int值来表示过滤器执行顺序，数值越小优先级越高。
+ shouldFilter：通过返回boolean值来判断该过滤器是否要执行
+ run：过滤器的具体逻辑

-----

**请求生命周期：**

![image-20200411225038395](https://xbxblog2.bj.bcebos.com/Zuul%2Fimage-20200411225038395.png)

+ 当外部请求到达网关后，首先进入pre类型过滤器，该类型过滤器主要目的是进行一些进行请求路由的前置加工，比如请求的校验等。
+ 在pre类型过滤器处理之后，将进入第二阶段routing类型过滤器处理。这里的具体内容就是将外部请求转发到具体服务实例上去
+ 当服务实例的响应返回后，此时就会进入post类型过滤器处理，这种类型的过滤器可以获取服务实例的响应信息，所以可以在post类型的过滤器种做一些内容加工之类工作。
+ 当上述三个阶段发生错误时，会进入error类型过滤器。但是error类型过滤器最终流向还是post类型过滤器，因为它需要post类型过滤器将最终结果返回给客户端。

-----

在定义过滤器时只需要集成ZuulFilter重写抽象方法。然后将其装配进容器即可。

需要了解一个类RequestContext。

RequestContext：用于在过滤器之间传递消息。它的数据保存在每个请求的ThreadLocal中。它用于存储请求路由到哪里、错误、HttpServletRequest、HttpServletResponse都存储在RequestContext中。RequestContext扩展了ConcurrentHashMap，所以，任何数据都可以存储在上下文中。