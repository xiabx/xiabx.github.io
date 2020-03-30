---

title: RestTemplate
author: XIA
categories:
  - null
tags:
  - null
date: 2020-03-22 20:21:00
---

学习Ribbon时候看到了RestTemplate，对其拦截机制不了解，所以看到哪里算哪里吧。

# RestTemplate组件

RestTemplate大致包含五个组件，其中一个组件ClientHttpRequestFactory在其父类中，另外四个在RestTemplate属性。

```java
//消息转换器
private final List<HttpMessageConverter<?>> messageConverters = new ArrayList<>();
//错误处理器
private ResponseErrorHandler errorHandler = new DefaultResponseErrorHandler();
//用于URL的构建
private UriTemplateHandler uriTemplateHandler;
//返回值处理器
private final ResponseExtractor<HttpHeaders> headersExtractor = new HeadersExtractor();
```

## ClientHttpRequestFactory

ClientHttpRequestFactory是一个接口，用来封装各种Http客户端，屏蔽他们之间的差异。做到RestTemplate所使用的Http客户端可拔插。当需要使用JDK的HttpURLConnection负责请求处理时就就用SimpleClientHttpRequestFactory，当需要使用OKHttp时就用OkHttp3ClientHttpRequestFactory。

该接口只有一个方法createRequest，该方法返回的ClientHttpRequest对象屏蔽了各个Http客户端之间请求的差异性。

**ClientHttpRequestFactory**

```java
public interface ClientHttpRequestFactory {

	ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException;

}
```

![image-20200322210651113](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322210651113.png)

**ClientHttpRequest**

```java
public interface ClientHttpRequest extends HttpRequest, HttpOutputMessage {

	ClientHttpResponse execute() throws IOException;

}
```

![image-20200322211455264](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322211455264.png)

![image-20200322211629185](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322211629185.png)

------

**ClientHttpRequestFactory与ClientHttpRequest在RestTemplate中的使用：**

在RestTemplate中所有的请求方法，最终都会调用doExecute方法来执行请求操作。而在doExecute中则是先根据ClientHttpRequestFactory创建获取ClientHttpRequest，然后执行ClientHttpRequest#execute方法

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
			@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
		ClientHttpResponse response = null;
		try {
            //创建ClientHttpRequest
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
				requestCallback.doWithRequest(request);
			}
            //执行请求操作
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

---

**InterceptingClientHttpRequestFactory**

ClientHttpRequestFactory有一个很重要的子类InterceptingClientHttpRequestFactory，其内部维护一个拦截器集合`List<ClientHttpRequestInterceptor> interceptors`。它创建的ClientHttpRequest支持在请求执行前进行拦截操作。

```java
//AbstractClientHttpRequestFactoryWrapper一个装饰器类，内部维护一个ClientHttpRequestFactory对象。
public class InterceptingClientHttpRequestFactory extends AbstractClientHttpRequestFactoryWrapper {
	//拦截器集合
	private final List<ClientHttpRequestInterceptor> interceptors;

	public InterceptingClientHttpRequestFactory(ClientHttpRequestFactory requestFactory,
			@Nullable List<ClientHttpRequestInterceptor> interceptors) {

		super(requestFactory);
		this.interceptors = (interceptors != null ? interceptors : Collections.emptyList());
	}


	//创建具有拦截器功能的InterceptingClientHttpRequest
	protected ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod, ClientHttpRequestFactory requestFactory) {
		return new InterceptingClientHttpRequest(requestFactory, this.interceptors, uri, httpMethod);
	}

}
```

**InterceptingClientHttpRequest**

```java
protected final ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
    //创建执行器
    InterceptingRequestExecution requestExecution = new InterceptingRequestExecution();
    return requestExecution.execute(this, bufferedOutput);
}


private class InterceptingRequestExecution implements ClientHttpRequestExecution {
	
    private final Iterator<ClientHttpRequestInterceptor> iterator;

    public InterceptingRequestExecution() {
        //当前拦截器集合的迭代器
        this.iterator = interceptors.iterator();
    }

    @Override
    public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
        //遍历拦截器集合
        if (this.iterator.hasNext()) {
            ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
            //这里将InterceptingRequestExecution传入进去了，在interceptor中会执行InterceptingRequestExecution.execute递归，以达到遍历的目的
            return nextInterceptor.intercept(request, body, this);
        }
        else {
            HttpMethod method = request.getMethod();
            //使用AbstractClientHttpRequestFactoryWrapper中包装的requestFactory，创建ClientHttpRequest
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
            //执行请求
            return delegate.execute();
        }
    }
}
```

## ExtractingResponseErrorHandler

当请求相应错误时用来处理请求。RestTemplate默认使用的是DefaultResponseErrorHandler。

DefaultResponseErrorHandler实现的hasError方法简单来说就是请求响应状态码是否由4、5开头。其实现的handleError方法就是根据不同的状态码抛出不同的错误。

```java
public interface ResponseErrorHandler {
	//是否有错误产生
	boolean hasError(ClientHttpResponse response) throws IOException;
	//处理错误
	void handleError(ClientHttpResponse response) throws IOException;
	
	default void handleError(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
		handleError(response);
	}

}
```

![image-20200322214735505](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322214735505.png)

---

**RestTemplate中ExtractingResponseErrorHandler调用时机**

同样是

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

    ClientHttpResponse response = null;
    try {
        ClientHttpRequest request = createRequest(url, method);
        if (requestCallback != null) {
            requestCallback.doWithRequest(request);
        }
        response = request.execute();
        //这里，当获取请求相应后便会去执行handleResponse处理是否产生错误
        handleResponse(url, method, response);
        
       //。。。。。。。。
}
```

```java
protected void handleResponse(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
    ResponseErrorHandler errorHandler = getErrorHandler();
    //是否有错误
    boolean hasError = errorHandler.hasError(response);
    if (logger.isDebugEnabled()) {
        try {
            int code = response.getRawStatusCode();
            HttpStatus status = HttpStatus.resolve(code);
            logger.debug("Response " + (status != null ? status : code));
        }
        catch (IOException ex) {
            // ignore
        }
    }
    //有的话，处理错误
    if (hasError) {
        errorHandler.handleError(url, method, response);
    }
}
```

## ResponseExtractor

响应提取器：从`Response`中提取数据。`RestTemplate`请求完成后，都是通过它来从`ClientHttpResponse`提取出指定内容（比如请求头、请求Body体等）

```java
public interface ResponseExtractor<T> {

	T extractData(ClientHttpResponse response) throws IOException;

}
```

主要实现类为HttpMessageConverterExtractor。其提取流程为先获取相应的Content-Type，然后遍历HttpMessageConverter列表，列表的初始化位于RestTemplate空参构造中。当某个converter匹配后会利用Converter转换为相应对象。

# 继承结构

RestTemplate继承自InterceptingHttpAccessor和RestOperations。

```java
public class RestTemplate extends InterceptingHttpAccessor implements RestOperations
```

## InterceptingHttpAccessor

```java
public abstract class InterceptingHttpAccessor extends HttpAccessor{
```

**其父类HttpAccessor**主要定义了管理ClientHttpRequestFactory的方法。并对ClientHttpRequestFactory设置初始值,`private ClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();`

![image-20200322203927357](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322203927357.png)

而**InterceptingHttpAccessor**主要用于管理请求的拦截器们。

```java
public abstract class InterceptingHttpAccessor extends HttpAccessor {
	//拦截器集合
	private final List<ClientHttpRequestInterceptor> interceptors = new ArrayList<>();

	private volatile ClientHttpRequestFactory interceptingRequestFactory;

	public void setInterceptors(List<ClientHttpRequestInterceptor> interceptors) {
		// Take getInterceptors() List as-is when passed in here
		if (this.interceptors != interceptors) {
			this.interceptors.clear();
			this.interceptors.addAll(interceptors);
			AnnotationAwareOrderComparator.sort(this.interceptors);
		}
	}


    //如果拦截器集合不为空，则返回具有拦截功能的InterceptingClientHttpRequestFactory实例代替原来的interceptingRequestFactory对象。
	public ClientHttpRequestFactory getRequestFactory() {
		List<ClientHttpRequestInterceptor> interceptors = getInterceptors();
		if (!CollectionUtils.isEmpty(interceptors)) {
			ClientHttpRequestFactory factory = this.interceptingRequestFactory;
			if (factory == null) {
				factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
				this.interceptingRequestFactory = factory;
			}
			return factory;
		}
		else {
			return super.getRequestFactory();
		}
	}

}

```

## RestOperations

类似于JDBCTemplate中的JdbcOperations。

定义了RestTemplate支持的方法。

![image-20200322203130186](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322203130186.png)

方法分类：

---

**GET请求有关**

RestTemplate对GET请求支持的方法分为两类：getForEntity和getForObject。

getForEntity的返回对象为Spring封装的HTTP响应对象ResponseEntity。

![image-20200322142104337](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322142104337.png)

getForObject返回的是对HTTP响应中body中的封装。

![image-20200322142222614](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322142222614.png)

---------

**POST请求有关**

POST请求的支持方法可以分为三类：postForEntity、postForObject和postForLocation。

postForEntity、postForObject与get的方法类似，他们通过发送post请求，然后将相应内容封装为ResponseEntity对象和将响应中的body封装为指定类型对象。

postForLocation方法则会返回新资源的URI。

![image-20200322143025167](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322143025167.png)

----------

**PUT请求有关**

RestTemplate对PUT请求的支持通过三个重载的put方法实现，他们的返回值都是void。

![image-20200322143233748](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322143233748.png)

---

**DELETE请求有关**

RestTemplate对DELETE请求的支持同样是通过三个重载的方法实现。

![image-20200322143525345](https://xbxblog2.bj.bcebos.com/RestTemplate%2Fimage-20200322143525345.png)

----

**exchange方法**：更通用的请求方法。它入参必须接受一个`RequestEntity`，从而可以设置请求的路径、头等等信息，最终全都是返回一个`ResponseEntity`（可以发送Get、Post、Put等所有请求）

最终所有的这些请求方法都会调用**execute**方法。。。

