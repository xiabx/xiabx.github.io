---
title: Feign
author: XIA
categories:
  - SpringCloud
tags:
  - null
date: 2019-06-10 21:22:09
---

# Feign是什么

使用Spring Cloud的项目几乎都会使用到Ribbon与Hystrix，而使用这两者需要很多重复的代码，比如在使用Ribbon时如果调用的都是同一个服务需要每次调用时都写同样的服务名，使用Hystix也需要手动生命每个方法生成Command的注解。

Feign就是解决这个问题的，简化了Ribbon与Hystrix使用，还提供了声明式服务调用，使得远程调用可以像同一个模块中使用方法直接调用。

# 简单示例

引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

开启Feign自动装配

```java
@EnableFeignClients
@EnableDiscoveryClient 
@SpringBootApplication
public class BackendShowConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(BackendShowConsumerApplication.class, args);
    }

}
```

声明一个API接口，该接口使用@FeignClient注解

```java
//name：起个名字，如果项目集成Ribbon表示服务名用于服务发现。path：统一前缀。url：HTTP访问目标地址
@FeignClient(name = "hello-service-provider",
        path = "/provider",
		url = "http://localhost:7101"
 )
public interface ProviderApi {

    //同SpringMVC注解，表示映射路径，参数绑定也和SpringMVC一样
    @RequestMapping(value = "/sayhello",method = RequestMethod.GET)
    String invokerProviderController(@RequestParam("message") String message);

}
```

使用时只需要将该接口注入，然后直接调用方法即可实现对指定服务调用

````java
@Slf4j
@RestController
public class ConsumerController {
    
    //注入
    @Resource
    private ProviderApi providerApi;

    @RequestMapping(value = "/sayhello/feign")
    public String sayHelloFeign(String message){
        System.out.println("feign message="+message);
        //调用
        return providerApi.invokerProviderController(message);
    }

}
````

# Feign支持的SpringMVC注解

支持什么类型的注解由Contract决定，Feign默认装配的是SpringMVCContract，所以支持SpringMVC注解。

+ 支持：@RequestMapping、@ResquestParam、@PathVariable、@RequestHeader、@RequestBody
+ 不支持：@GetMapping、@PostMapping

# Feign与Ribbon、Hystrix集成

Feign默认集成了Ribbon与Hystrix。

当使用Ribbon时只需要在@FeignClient的name|value属性指定微服务名称，用于服务发现。

当需要使用Hystrix时，需要在配置文件中开启Hystrix支持`Feign.hystrix.enable=true`。然后在@FeignClient注解使用fallback或fallbackFactory属性指定降级策略。

# @FeignClient属性

+ **name|value：**起个名字，如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现

+ **url：**HTTP访问的目标地址

+ **Primary：**使用@FeignClient注解的接口之所以可以被直接调用，是因为底层使用动态代理生成了该接口的实现类，并装配为Bean，同时使用@Primary进行注解。当我们不想使用Feign为我们动态代理生成的Bean，而是自己实现该接口并装配为Bean，此时可以在@FeignClient注解上将Primary设置为false，这样在别的类注入该接口时则会注入自己实现的接口实现类。注意自己的实现类上加上@Primary注解

  ```java
  @FeignClient(name = "hello-service-provider",
          primary = false,
          path = "/provider"
   )
  public interface ProviderApi {
      String invokerProviderController(String message);
  }
  ```

  ```java
  @Primary
  @Service
  public class ProviderAPIImpl implements ProviderApi{
      @Override
      public String invokerProviderController(String message) {
          return null;
      }
  }
  ```

+ **configuration：**指定一个配置类，该类中可以配置Feign所使用的Bean用来配置Feign。比如：编解码器，日志服务，Contract等。

  ```java
  @Configuration
  public class FeignHelloConf {
  	
      //装配另一个contract，用来覆盖默认的SpringMVCContract
      //Feign之所以可以使用SpringMVC注解就是因为默认的contract为SpringMVCContract类型
      @Bean
      public Contract contract(){
          return new feign.Contract.Default();
      }
  
  }
  ```

  ```java
  //使用configuration属性
  @FeignClient(name = "hello-service-provider",
          path = "/provider",
          configuration = FeignHelloConf.class,
   )
  public interface ProviderApi {
  }
  ```
  
+ **fallback：**与Hystrix集成时使用，指定fallback实现类，实现业务降级逻辑

  ```java
  //fallback实现类，注意实现api接口
  @Service
  public class ProviderFallbackAPIImpl implements ProviderApi{
  
      @Override
      public String invokerProviderController(String message) {
          return "invokerProviderController fallback message="+message;
      }
  
  }
  ```

  ```java
  //使用fallback属性。这样当invokerProviderController方法发生错误时会自动降级至ProviderFallbackAPIImpl实现类中。
  @FeignClient(name = "hello-service-provider",
          path = "/provider",
          fallback = ProviderFallbackAPIImpl.class
   )
  public interface ProviderApi {
      String invokerProviderController(String message);
  }
  ```
  
+ **fallbackFactory：**指定相应的降级方法的工厂实现类，在create方法中完成fallback子类实现。使用fallbackFactory的实现类可以获取异常的详细信息。

  ```java
  //实现 feign.hystrix.FallbackFactory工厂接口
  @Service
  public class FallbackFactory implements feign.hystrix.FallbackFactory {
  
      @Override
      public ProviderApi create(Throwable throwable) {
          return new ProviderApi() {
              @Override
              public String invokerProviderController(String message) {
                  return "invokerProviderController FallbackFactory message="+message;
              }
          };
      }
  
  }
  ```

  ```java
  //指定fallbackFactory属性
  @FeignClient(name = "hello-service-provider",
          path = "/provider",
          fallbackFactory = FallbackFactory.class
   )
  public interface ProviderApi {
  }
  ```
  
# 继承特性

  在服务提供方与服务消费者之间，他们的服务接口都是一样的，往往需要从服务提供方将方法声明复制过来。为了解决这种繁琐的操作可以使用继承特性。

  单独定义一个服务用来存放接口声明相关信息与可以复用的DTO对象。服务提供方与服务消费方引入这个公用的api模块。当服务提供方需要定义服务接口的实现时，只需要实现相应接口。而服务消费方需要使用时，只需要单独定义一个接口继承来自公用api模块相应接口。

  这样可以实现接口定义从controller中剥离。
