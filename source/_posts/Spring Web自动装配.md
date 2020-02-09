---
title: Spring Web自动装配
author: XIA
categories:
  - null
tags:
  - null
date: 2020-02-09 20:56:33
---

# 什么是web自动装配

Servlet 3.0之前我们搭建Spring MVC项目需要在web.xml中手动配置ContextLoaderListener和DispatchServlet，这么直接写死的方式着实缺乏灵活性。自Servlet 3.0后spring通过相应API实现了在项目运行时自动将项目框架需要的Servlet、Filter注入。这就是web自动注入。

**示例：**

通过在项目中任意位置添加一个AbstractAnnotationConfigDispatcherServletInitializer实现类，就可以实现将ContextLoaderListener和DispatchServlet进行自动装配，而无需web.xml。

```java
public class SpringWebMvcServletInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[0];
    }

    @Override
    protected Class<?>[] getServletConfigClasses() { // DispatcherServlet 配置Bean
        return of(SpringWebMvcConfiguration.class);
    }

    @Override
    protected String[] getServletMappings() {        // DispatcherServlet URL Pattern 映射
        return of("/*");
    }

    private static <T> T[] of(T... values) {         // 便利 API ，减少 new T[] 代码
        return values;
    }
}
```



# 原理

spring web自动装配的原理是对Servlet 3.0技术的应用。Servlet 3.0以后推出了“ServletContext配置方法”和“运行时插拔”两个技术更新，spring通过对这两者的合理运行实现了web自动装配功能。

## ServletContext配置方法

在传统web应用中使用web.xml来配置应用中的Servlet、Filter、Listener。显然这种通过web.xml配置的方式不够灵活，运行时无法调整。

为了解决这个问题Servlet3.0推出了通过ServletContext配置的方法，通过相应addServlet、addFilter、addListener方法动态装配Servlet、Filter、Listener。

## 运行时插拔

在ServletContext动态装配了Servlet、Filter、Listener后可以在两个时刻调用：ServletContextListener#contextInitialized或ServletContainerInitializer#onStartup方法。这两个方法一个是在ServletContext容器初始化结束后调用，一个是在web容器启动或应用启动时调用。

spring通过ServletContainerInitializer#onStartup的方式来调用这些动态加载Servlet、Filter、Listener的方法。因为spring mvc需要注册ServletContextListener，它是ServletContextListener实例，要在ServletContext初始化完成前注册。

当容器启动时ServletContainerInitializer#onStartup(Set<Class<?>>,ServletContext)方法被调用，第一个参数的Set集合通过ServletContainerInitializer实现类上的@HandlesTypes注解进行过滤，指定类型及其子类将将组合为第一个参数Set集合。然后需要将ServletContainerInitializer的一个或多个实现类存放在一个名为`javax.servlet.ServletContinerInitializer`的文本文件中，该文件存放在独立JAR包中的`META-INF/services`目录下。

# 自动web装配实现过程

## ServletContainerInitializer的实现类

在spring中ServletContainerInitializer的实现类为SpringServletContainerInitializer。该实现类也在`javax.servlet.ServletContinerInitializer`进行了配置。

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
	
    //第一个参数为WebApplicationInitializer子类集合
	public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {

				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer) waiClass.newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		AnnotationAwareOrderComparator.sort(initializers);
		servletContext.log("Spring WebApplicationInitializers detected on classpath: " + initializers);

		for (WebApplicationInitializer initializer : initializers) {
            //分别执行WebApplicationInitializer接口的onStartup方法
			initializer.onStartup(servletContext);
		}
	}

}

```

通过`@HandlesTypes(WebApplicationInitializer.class)`注解，结合规范可以得知WebApplicationInitializer的子类的集合将会作为第一个入参。

关于web自动装配spring提供了三种WebApplicationInitializer的实现

![image-20200209214511295](https://xbxblog2.bj.bcebos.com/springweb%E8%87%AA%E5%8A%A8%E8%A3%85%E9%85%8D%2Fimage-20200209214511295.png)

+ AbstractContextLoaderInitializer：替代web.xml注册ContextLoaderListener
+ AbstractDispatcherServletInitializer：替代web.xml注册DispatchServlet
+ AbstractAnnotationConfigDispatcherServletInitializer：具备Annotation配置驱动能力的AbstractDispatcherServletInitializer

## 三种WebApplicationInitializer实现

**AbstractContextLoaderInitializer**

```java
public abstract class AbstractContextLoaderInitializer implements WebApplicationInitializer {

	protected final Log logger = LogFactory.getLog(getClass());


	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		registerContextLoaderListener(servletContext);
	}

	protected void registerContextLoaderListener(ServletContext servletContext) {
		WebApplicationContext rootAppContext = createRootApplicationContext();
		if (rootAppContext != null) {
			ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
			listener.setContextInitializers(getRootApplicationContextInitializers());
			servletContext.addListener(listener);
		}
		else {
			logger.debug("No ContextLoaderListener registered, as " +
					"createRootApplicationContext() did not return an application context");
		}
	}

    
	//模板方法，创建WebApplicationContext
	protected abstract WebApplicationContext createRootApplicationContext();

	@Nullable
	protected ApplicationContextInitializer<?>[] getRootApplicationContextInitializers() {
		return null;
	}

}
```

可见该实现类并没有实现创建WebApplicationContext的方法。

**AbstractDispatcherServletInitializer**

```java
public abstract class AbstractDispatcherServletInitializer extends AbstractContextLoaderInitializer {
    
	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		super.onStartup(servletContext);
		registerDispatcherServlet(servletContext);
	}

	protected void registerDispatcherServlet(ServletContext servletContext) {
		String servletName = getServletName();
        
        .......
            
		WebApplicationContext servletAppContext = createServletApplicationContext();

		FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);

        .......
        
		ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);

		registration.setLoadOnStartup(1);
		registration.addMapping(getServletMappings());
		registration.setAsyncSupported(isAsyncSupported());

		Filter[] filters = getServletFilters();
		if (!ObjectUtils.isEmpty(filters)) {
			for (Filter filter : filters) {
				registerServletFilter(servletContext, filter);
			}
		}

		customizeRegistration(registration);
	}
```

AbstractDispatcherServletInitializer实现了对DispatcherServlet的动态添加，但是没有实现创建相应ioc容器的方法。

**AbstractAnnotationConfigDispatcherServletInitializer**

前面的AbstractDispatcherServletInitializer既没有实现createRootApplicationContext也没有实现createServletApplicationContext方法。而AbstractAnnotationConfigDispatcherServletInitializer相对来说更像是完全体了，不光实现了该实现的方法而且还对注解驱动进行了支持。

```java
public abstract class AbstractAnnotationConfigDispatcherServletInitializer
		extends AbstractDispatcherServletInitializer {

	protected WebApplicationContext createRootApplicationContext() {
		Class<?>[] configClasses = getRootConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
			context.register(configClasses);
			return context;
		}
		else {
			return null;
		}
	}

	protected WebApplicationContext createServletApplicationContext() {
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		Class<?>[] configClasses = getServletConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			context.register(configClasses);
		}
		return context;
	}

	protected abstract Class<?>[] getRootConfigClasses();

	protected abstract Class<?>[] getServletConfigClasses();

}
```

AbstractAnnotationConfigDispatcherServletInitializer主要实现了两个创建容器的方法。注解驱动是通过内建AnnotationConfigWebApplicationContext来支持。

# 总结

就这样，原来通过web.xml配置的ContextLoaderListener和DispatchServlet两个组件，悄咪咪就被注册了。













