---
title: Ribbon
author: XIA
categories:
  - null
tags:
  - null
date: 2020-03-22 13:44:18
---

# 负载均衡

负载均衡通常可以分为两类：服务端负载均衡和客户端负载均衡。

服务端负载均衡通常都会维护一个可用的服务端清单，通过心跳检测清单下的服务是否可以正常访问，当请求来到时，负载均衡设备会按照某种算法转发到服务清单中的一台设备上。

![image-20200322140236824](https://xbxblog2.bj.bcebos.com/Ribbon%2Fimage-20200322140236824.png)

客户端负载均衡与服务端负载均衡的不同点在于服务端清单所在位置不同。客户端负载均衡，所有客户端节点都维护着自己要访问的服务端清单，而服务端清单来自于注册中心，当然也可以自己手动配置。同服务端负载均衡类似，客户端负载均衡也需要维护心跳去保证服务端清单中服务的健康性，这个可以通过注册中心的协助，也可以自己进行实现。Ribbon属于客户端负载均衡。

使用Rebbion只需要两步即可：

1. 服务提供者启动多个实例，将自己注册到注册中心
2. 服务消费者使用@LoadBalanced注解修饰的RestTemplate来实现面向服务的接口调用

RestTemplate对象会使用Rebbion的自动化配置，同时通过@LoadBalanced还能开启客户端负载均衡。

# 原理

## Interceptor添加

Rebbion通过RestTemplate的拦截器机制来实现负载均衡。所以分析的入口就在Interceptor的设置处。

同样是通过SpringBoot自动配置机制。LoadBalancerAutoConfiguration是一个SpringBoot自动配置所导入的类。

```java
public class LoadBalancerAutoConfiguration {

	@LoadBalanced
	@Autowired(required = false)
    //所有被@LoadBalanced注解过的RestTemplate集合
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

	@Bean
    //SmartInitializingSingleton接口实例将会在spring预加载了所有单例bean后触发
    //在触发时就使用customizers来配置restTemplates了
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		});
	}

	@Bean
	@ConditionalOnMissingBean
    //TODO
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
    //两个Bean都与Interceptor有关
	static class LoadBalancerInterceptorConfig {

		@Bean
        //创建用来拦截请求的Interceptor
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
        //封装将Interceptor设置到restTemplate的逻辑
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(
						restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
				restTemplate.setInterceptors(list);
			};
		}

	}
    

}

```

先是注册了LoadBalancerInterceptor的Bean，然后注册用于将LoadBalancerInterceptor添加到RestTemplate的Custmizer的Bean，之后利用SmartInitializingSingleton执行的触发时机，执行Custmizer。这样就将拦截器添加到了RestTemplate中了。

## 拥有Interceptor的RestTemplate执行方式

所有的RestTemplate的请求方法底层都是执行doExecute()方法。所以从这个方法入手

**RestTemplate#doExecute**

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback, @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

    ClientHttpResponse response = null;
    try {
        //因为设置了Interceptor，所以这里创建的request为InterceptingClientHttpRequest
        ClientHttpRequest request = createRequest(url, method);
        if (requestCallback != null) {
            requestCallback.doWithRequest(request);
        }
        //执行请求
        response = request.execute();
        handleResponse(url, method, response);
        return (responseExtractor != null ? responseExtractor.extractData(response) : null);
    }
    catch (IOException ex) {
        String resource = url.toString();
        String query = url.getRawQuery();
        resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
        throw new ResourceAccessException("I/O error on " + method.name() +
                                          " request for \"" + resource + "\": " + ex.getMessage(), ex);
    }
    finally {
        if (response != null) {
            response.close();
        }
    }
}
```

**InterceptingRequestExecution#execute**

```java
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
    if (this.iterator.hasNext()) {
        ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
        //这里执行拦截
        return nextInterceptor.intercept(request, body, this);
    }
    else {
        HttpMethod method = request.getMethod();
        ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
        request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
        if (body.length > 0) {
            if (delegate instanceof StreamingHttpOutputMessage) {
                StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
            }
            else {
                StreamUtils.copy(body, delegate.getBody());
            }
        }
        return delegate.execute();
    }
}
```

## LoadBalancerInterceptor

LoadBalancerInterceptor这个就是Rebbion为实现负载均衡而使用的拦截器，里面应该有我们想知道的关于负载均衡的很多东西。不过在了解他之前还要了解一个接口：LoadBalancerClient

LoadBalancerClient表示负载均衡的客户端的接口。

+ choose方法：根据传入的serviceId，从负载均衡中挑选出一个对应服务的实例
+ execute方法：使用从负载均衡中挑选出来的服务实例执行请求
+ reconstructURI方法：根据选出的服务实例，构建出一个合适的URI

![image-20200323202527126](https://xbxblog2.bj.bcebos.com/Ribbon%2Fimage-20200323202527126.png)

接下来到了LoadBalancerInterceptor。

**LoadBalancerInterceptor#intercept**

```java
@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
final ClientHttpRequestExecution execution) throws IOException {
    final URI originalUri = request.getURI();
    String serviceName = originalUri.getHost();
    //this.loadBalancer为LoadBalancerClient类实例
    //这里创建的Request实例，注意其方法。后期将用来执行重构URI
    return this.loadBalancer.execute(serviceName,
                                     this.requestFactory.createRequest(request, body, execution));
}
```

这里的LoadBalancerClient的Bean也是由springBoot自动配置机制加载。默认加载的实现类为RibbonLoadBalancerClient。配置类位于RibbonAutoConfiguration。

接下来进入**RibbonLoadBalancerClient#execute**

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
    throws IOException {
    //这里是获取负载均衡的策略，
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    //根据负载均衡策略选择出合适的服务实例
    Server server = getServer(loadBalancer, hint);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    //使用RibbonServer包装Server，包含更多属性，如：是否使用HTTPS等
    RibbonServer ribbonServer = new RibbonServer(serviceId, server,
     isSecure(server, serviceId),serverIntrospector(serviceId).getMetadata(server));
	
    return execute(serviceId, ribbonServer, request);
}
```

> ILoadBalancer这个接口是netflix中的，他的实现类才是各种负载均衡的实现，其中负载均衡实现中会包含负载均衡策略，还牵扯到与Eureka集成。所以。。。以后再看吧。这个接口与LoadBalancerClient接口可以看作构建了springCloud与Ribbon集成的桥梁。
>
> 除了ILoadBalancer接口，Ribbon还有两个接口可对其进行配置。IRule：抽象服务选择的接口。IPing：对服务列表中的服务存活状态进行判断的接口，他的两个主要实现，一个使用Eureka提供的服务列表，另一个自己通过指定URL进行判断。
>
> 在ILoadBalancer实现中，维护两个ServerList，保存全部服务的List与可用状态的服务List。

**RibbonLoadBalancerClient#execute**

```java
public <T> T execute(String serviceId, ServiceInstance serviceInstance,
                     LoadBalancerRequest<T> request) throws IOException {
    Server server = null;
    if (serviceInstance instanceof RibbonServer) {
        server = ((RibbonServer) serviceInstance).getServer();
    }
    //
    RibbonLoadBalancerContext context = this.clientFactory
        .getLoadBalancerContext(serviceId);
    RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

		//执行Request的回调，根据apply方法的内容，可知该方法用于执行reconstructURI。用于重构URI
        T returnVal = request.apply(serviceInstance);
        statsRecorder.recordStats(returnVal);
        return returnVal;

    
    return null;
}
```

**LoadBalancerRequestFactory#createRequest**

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(
    final HttpRequest request, final byte[] body,
    final ClientHttpRequestExecution execution) {
    //回调方法
    return instance -> {
        //这个包装类中个getURI方法就是执行了重构URI的方法
        HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
                                                               this.loadBalancer);
        if (this.transformers != null) {
            for (LoadBalancerRequestTransformer transformer : this.transformers) {
                serviceRequest = transformer.transformRequest(serviceRequest,
                                                              instance);
            }
        }
        //然后继续执行别的拦截器或者进行请求操作了。
        //ClientHttpRequestExecution execution RestTemplate相关的
        return execution.execute(serviceRequest, body);
    };
}
```

**ServiceRequestWrapper#getURI**

```java
//根据所选服务实例重构URI
public URI getURI() {
    URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
    return uri;
}
```