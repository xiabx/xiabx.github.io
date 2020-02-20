---
title: SpringBoot之SpringApplication
author: XIA
categories:
  - null
tags:
  - null
date: 2020-02-18 16:58:45
---

SpringBoot应用的整个生命周期可以分为以下四部分：

+ SpringApplication 初始化阶段
+ SpringApplication 运行阶段
+ SpringApplication 结束阶段
+ Spring Boot应用退出

# SpringApplication 初始化阶段

初始化阶段可以分为两个阶段：构造阶段和配置阶段。构造阶段负责SpringApplication对象的创建，配置阶段则负责SpringApplication对象的属性设置。

## 构造阶段

使用SpringBoot时往往都是通过`SpringApplication.run;`方法启动。跟进该方法后发现其背后是通过new一个SpringApplication对象然后调用run方法。

```java
public static ConfigurableApplicationContext run(Class<?> primarySource,
                                                 String... args) {
    return run(new Class<?>[] { primarySource }, args);
}


public static ConfigurableApplicationContext run(Class<?>[] primarySources,
                                                 String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

这里的primarySource参数并不一定需要为SpringBoot启动类，它只需要是一个Configuration class即可。意思是只要一个类上有@Configuration注解即可被用做primarySource。除了在构造器指定primarySource还可以使用SpringApplication#addPrimarySources方法进行追加。

接下来就是SpringApplication的构造函数了。

```java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //推断web应用类型
    this.webApplicationType = deduceWebApplicationType();
    //加载spring应用上下文初始化器
    setInitializers((Collection) getSpringFactoriesInstances(
        ApplicationContextInitializer.class));
    //加载spring应用事件监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //推断引导类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 推断web应用类型

原理就是查找应用中是否存在指定类。

```java
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
    + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

private static final String REACTIVE_WEB_ENVIRONMENT_CLASS = "org.springframework."
    + "web.reactive.DispatcherHandler";

private static final String MVC_WEB_ENVIRONMENT_CLASS = "org.springframework."
    + "web.servlet.DispatcherServlet";


private WebApplicationType deduceWebApplicationType() {
    if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
        && !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```

### 加载spring应用上下文初始化器

加载spring应用上下文初始化器时会用到META-INF/spring.factories，这次使用该文件中的`org.springframework.context.ApplicationContextInitializer`接口。在获取该接口的实现类对象后设置到SpringApplication的相关属性中。

```
public void setInitializers(Collection<? extends ApplicationContextInitializer<?>>     initializers) {
    this.initializers = new ArrayList<>();
    this.initializers.addAll(initializers);
}
```

### 加载spring应用事件监听器

该部分与上部分类似，都是使用META-INF/spring.factories文件进行加载的，这次的接口是`org.springframework.context.ApplicationListener`。之后也是将相应实现类设置到属性中。

```java
public void setListeners(Collection<? extends ApplicationListener<?>> listeners) {
    this.listeners = new ArrayList<>();
    this.listeners.addAll(listeners);
}
```

### 推断引导类

引导类就是启动SpringBoot应用的main方法所在的类。

推断的原理是根据当前线程的执行栈中哪个雷包含main方法。

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```

## 配置阶段

配置阶段位于构造阶段和运行阶段之间。也就是run方法之前。通过SpringApllication对象的一些set方法和SpringApplicationBuilder的一些方法可以改变springboot的运行方式和一些参数配置。如banner、WEBType等。

SpringApplicationBuilder是为了简化SpringApllication配置繁琐而产生的类，它内部维护着一个SpringApllication，它提供一些流式的配置api。

# SpringApplication 运行阶段

运行阶段指的就是run方法的内容了。它包含三部分：SpringApplication准备阶段、ApplicationContext启动阶段、ApplicationContext启动后阶段。

**SpringApplication#run**

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                                                                 applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        context = createApplicationContext();
        exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
        prepareContext(context, environment, listeners, applicationArguments,
                       printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                .logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```



## SpringApplication准备阶段

准备阶段的范围是从run方法开始到`refreshContext(context);`之前。

```java
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			
            ..........
		}
		catch ........
	}

```

此过程涉及到几个重要对象：SpringApplicationRunListeners、ApplicationArguments、ConfigurableEnvironment、Banner、ConfigurableApplicationContext、SpringBootExceptionReporter。

### SpringApplicationRunListeners

该对象是一个组合对象，内部维护一个SpringApplicationRunListener的list容器。

`SpringApplicationRunListeners listeners = getRunListeners(args);`

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
        SpringApplicationRunListener.class, types, this, args));
}
```

SpringApplicationRunListener的对象创建也是使用的META-INF/spring.factories。接口为`org.springframework.boot.SpringApplicationRunListener`。

### 装配ApplicationArguments

`ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);`

ApplicationArguments是一个简化SpringBoot应用启动参数封装的接口，他的底层实现基于命令行配置源SimpleCommandLinePropertySource。

```java
public class DefaultApplicationArguments implements ApplicationArguments {

	private final Source source;

	private final String[] args;

	public DefaultApplicationArguments(String[] args) {
		Assert.notNull(args, "Args must not be null");
		this.source = new Source(args);
		this.args = args;
	}	
    
    .......

	private static class Source extends SimpleCommandLinePropertySource {
		Source(String[] args) {
			super(args);
		}

		@Override
		public List<String> getNonOptionArgs() {
			return super.getNonOptionArgs();
		}

		@Override
		public List<String> getOptionValues(String name) {
			return super.getOptionValues(name);
		}

	}
}
```

该对象的作用是对命令行启动参数进行封装，如--name=xia会解析为name:xia的形式。

### 创建Spring应用上下文

`context = createApplicationContext();`

根据web类型创建相应的容器。

```java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
                case SERVLET:
                    contextClass = Class.forName(DEFAULT_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Unable create a default ApplicationContext, "
                + "please specify an ApplicationContextClass",
                ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

 ### Spring应用上下文运行前准备

该过程根据SpringApplicationRunListener的声明周期又分为：Spring应用上下文准备阶段和Spring应用上下文装载阶段。

```java
private void prepareContext(ConfigurableApplicationContext context,
                            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
                            ApplicationArguments applicationArguments, Banner printedBanner) {
    //设置Spring上下文ConfigurationableEnvironment
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    //执行Initializers中的方法，这些对象在创建SpringApplication时被加载
    applyInitializers(context);
    //以上为准备阶段
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // 注册Spring Boot Bean
    context.getBeanFactory().registerSingleton("springApplicationArguments",
                                               applicationArguments);
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }

    // Load the sources
    //合并Spring应用上下文配置源
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    //将sources中的类装载为BeanDefinition
    load(context, sources.toArray(new Object[0]));
    //以上为装载阶段
    listeners.contextLoaded(context);
}
```

## ApplicationContext启动阶段

直接调用的ApplicationContext的refresh方法。

## ApplicationContext启动后阶段

SpringApplication#afterRefresh方法未实现，留给开发自己扩展。







































