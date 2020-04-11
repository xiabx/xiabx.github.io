---
title: Spring-MVC
author: XIA
categories:
  - null
tags:
  - 所有人都想拯救世界，却没人帮妈妈洗碗
date: 2020-01-19 11:42:47
---

# 总览

Spring MVC使用两层Controller设计，DispatcherServlet作为对外开放的Front Controller来接受请求，并根据配置规则转发给对应的实际处理请求的Controller。

当请求到达DispatchServlet后根据HandlerMapping配置的映射规则映射到实际处理请求的Controller实现类。当Controller处理完成后会返回一个ModelAndView实例，ModelAndView包含两部分信息：视图的逻辑名称（DispatchServlet将根据该名称决定用户显示哪个视图），模型数据（视图渲染过程中需要将这些模型数据并入视图显示中）。当Controller处理完成后DispatchServlet将根据ViewResolver配置获取相应的View实现类，View接口是一个处理模板渲染的抽象接口。之后就可以根据返回的View实例将ModelAndView中的数据渲染到模板中。

Spring MVC会创建两个ApplicationContext，一个在作为根容器负责定义业务中使用的组件，如：数据源、service层定义等。另一个则是负责spring mvc框架所需组件的定义工作，如：HandlerMapping、ViewResolver等，继承自根容器。

# 主要组件

## HandlerMapping

HandlerMapping负责根据url映射到具体处理的Controller。

一些HandlerMapping的实现：

+ BeanNameUrlHandlerMapping：根据请求url的路径确定Controller类BeanName，url与BeanName耦合性较大。
+ SimpleUrlHandlerMapping：支持手动配置url与处理Controller类BeanName的映射。
+ DefaultAnnotationHandlerMapping：基于注解配置的方式进行映射。

## Controller

controller是用于处理具体web请求的handler类型之一。

Controller接口作为所有controller的顶层抽象，handleRequest方法将在DispatchServlet中被调用，返回ModelAndView实例。

框架提供了抽象类AbstractController对请求处理的通用逻辑进行统一处理，模板方法模式。

```java
public interface Controller {

	ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception;

}
```

## ModelAndView

在Controller处理请求完成后将会返回一个ModelAndView实例。它包括两部分内容：视图名称和模型数据。

模型数据在ModelAndView中使用一个ModelMap对象保存。ModelMap是LinkedHashMap子类。

## ViewResolver

在Controller处理请求后返回ModelAndView实例，ViewResolver职责就是根据ModelAndView中视图名返回一个可用的View实例。

ViewResolver接口很简单~~spring好像所有组件的接口都很简单0.0

```java
public interface ViewResolver {

	View resolveViewName(String viewName, Locale locale) throws Exception;

}
```

![image-20200120154514017](https://xbxblog2.bj.bcebos.com/springmvc%2Fimage-20200120154514017.png)

AbstractCachingViewResolver提供了缓存功能，可以将根据视图名解析的View进行缓存，以提高性能。

ViewResolver可以分为两种：面向单一视图类型的ViewResolver和面向多视图类型的ViewResolver。

面向单一视图类型的ViewResolver都直接或间接地继承UrlBasedViewResolver类。主要的实现类有：

+ InternalResourceViewResolver：负责jsp模板的映射
+ FreeMarkerViewResolver：负责FreeMarker模板的映射

## View

在ViewResolver根据视图名解析返回指定的View实例。

DispatchServlet将调用View实例的render方法，该方法并不会将数据与模板View进行结合，通常DispatchServlet会将ModelMap转换为模板引擎需要的类型，然后将数据渲染到模板的工作委托给模板引擎。JSP的模板引擎则直接委托给tomcat，FreeMarker的渲染则委托给FreeMarker的模板引擎。

```java
public interface View {

	String RESPONSE_STATUS_ATTRIBUTE = View.class.getName() + ".responseStatus";

	String PATH_VARIABLES = View.class.getName() + ".pathVariables";

	String SELECTED_CONTENT_TYPE = View.class.getName() + ".selectedContentType";

	@Nullable
	default String getContentType() {
		return null;
	}

	void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
			throws Exception;

}
```

![image-20200120205103900](https://xbxblog2.bj.bcebos.com/springmvc%2Fimage-20200120205103900.png)

这里有一个特殊的View实现RedirectView，它负责请求的重定向。

# 其他组件

## MultipartResolver

MultipartResolver组件用于spring mvc中文件上传。 其原理是在DispatchServlet中，当请求到达时会首先调用MultipartResolver实现类中的isMultipart方法，判断原理为按照ContextType是否以`multipart\`开头。

若判断为true则会对上传文件进行处理，然后返回MultipartHttpServletRequest用以替代之前的HttpServletRequest来进行后续请求处理，然后就可以像处理普通表单字段那样处理上传的文件了。

```java
public interface MultipartResolver {

	boolean isMultipart(HttpServletRequest request);

	MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;

	void cleanupMultipart(MultipartHttpServletRequest request);

}

```

```java
public interface MultipartHttpServletRequest extends HttpServletRequest, MultipartRequest {

	HttpMethod getRequestMethod();

	HttpHeaders getRequestHeaders();

	HttpHeaders getMultipartHeaders(String paramOrFileName);

}



public interface MultipartRequest {

	Iterator<String> getFileNames();

	MultipartFile getFile(String name);

	List<MultipartFile> getFiles(String name);
	
    ...........
    
	String getMultipartContentType(String paramOrFileName);

}
```

## Handler与HandlerAdaptor

在上面我们知道当请求到达DispatchServlet后会委托给HandlerMapping获取处理请求的Controller。但是这只是处理Controller类型作为处理器的一种简化，实际上spring mvc支持多种Handler。spring是如何实现这种结构的呢？没有问题是加一个中间层解决不了的。。。所以spring mvc添加了一个中间层，这个中间层就是HandlerAdaptor，当通过HandlerMapping返回Handler后，会遍历所有HandlerAdaptor根据supports验证是否支持，然后调用handle进行实际处理。

```java
public interface HandlerAdapter {

	boolean supports(Object handler);

	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	long getLastModified(HttpServletRequest request, Object handler);

}
```

当我们要使用一个自定义的Handler时需要提供一整套从HandlerMapping、HandlerAdaptor、Handler的实现。其中Handler的实现没有任何要求，只是一个POJO。HandlerMapping要根据自定义的Handler特征返回处理请求的Handler实例，接下来DispatchServlet则会根据Handler实例选取合适的HandlerAdaptor执行处理方法了~

## HandlerInterceptor

在HandlerMapping返回用于处理具体请求的Handler对象，是通过一个HandlerExectionChain进行封装的，HandlerExectionChain除了包含用于处理请求的Handler外还有一个HandlerInterceptor数组。其中HandlerInterceptor的作用就是在Handler执行前后进行处理流程的拦截处理。

```java
public interface HandlerInterceptor {

	default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return true;
	}

	default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable ModelAndView modelAndView) throws Exception {
	}

	default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable Exception ex) throws Exception {
	}

}
```

+ preHandle方法将在相应的的HandlerAdaptor调用具体的Handler处理web请求之前执行。该方法的返回值决定是否执行后续处理流程。
+ postHandle该方法在Handler处理完请求后，并且在视图解析和渲染前。该方法可以对Handler返回的ModelAndView进行处理。
+ afterCompletion该方法在流程结束之后，也就是视图渲染之后执行。

## HandlerExceptionResolver

HandlerExceptionResolver可以在Handler处理请求抛出异常时，此时他不会按照原定流程返回ModelAndView实例，而是会将该请求委托给HandlerExceptionResolver，他的实现类将根据异常类型构造相应新的ModelAndView并导向新的异常页面。

其实HandlerExceptionResolver和Handler是同级的关系，当Handler出现异常后HandlerExceptionResolver将直接接管后续异常处理工作。

# 基于注解的Controller

现在使用spring mvc基本上都是使用注解的形式，其实注解形式的Controller与之前的通过继承实现的Controller并没有本质的区别。如果将注解形式的Conroller当作一种自定义Handler来看的话就会很明了了。

之前我们说过，如果自定义Handler需要将HandlerMapping与HandlerAdaptor一同定义了。现在自定义Handler的格式已经有了，所以只要搞懂了HandlerMapping与HandlerAdaptor的实现思路一切就明了了。

## 基于注解的Controller的HandlerMapping

在基于注解的Controller实现类中，当前类将用于哪个Web请求的处理是由相应的注解标注的，如：`@RequestMapping("\hello")`。这些注解信息将通过反射来读取。

要实现一个基于注解的Controller的HandlerMapping思路就是，便利所有可用的基于注解的Controller实现类，然后根据请求的路径信息，与实现类中标注的映射信息进行比对。如果请求路径信息与实现类中标注的映射信息比对成功则返回当前基于注解的Controller实现类。

**伪代码：**

![image-20200121000317544](https://xbxblog2.bj.bcebos.com/springmvc%2Fimage-20200121000317544.png)

spring官方提供的处理基于注解的Controller的HandlerMapping是DefaultAnnotationHandlerMapping。

## 基于注解的Controller的HandlerAdaptor

基于注解的Controller并不会像通过继承来实现Controller那样有很大的“契约关系”。

其实现也是根据反射来拿到所有方法，然后通过比对映射路径找到正确的方法，然后通过反射执行该处理方法。

其实在自定义基于注解的Controller的HandlerAdaptor会面临很多问题：

+ 如何根据POST、GET方法，调用相应的处理方法
+ 数据绑定时，如何决定将哪个请求参数半丁到方法的哪个参数上。
+ 如何处理方法参数上的@RequestParam注解
+ 基于注解的Controller中如何对HttpSession进行管理

spring为我们提供的实现类为：AnnotaionMethodHandlerAdaptor
