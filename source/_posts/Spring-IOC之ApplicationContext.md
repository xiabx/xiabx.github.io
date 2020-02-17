---
title: Spring-IOC之ApplicationContext
author: XIA
categories:
  - null
tags:
  - null
date: 2020-02-02 19:01:01
---

# ClassPathXmlApplicationContext

```java
public ClassPathXmlApplicationContext(
      String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
      throws BeansException {
   //设置父容器
   super(parent);
    //设置配置文件
   setConfigLocations(configLocations);
   if (refresh) {
      refresh();
   }
}
```

# super(parent);

沿着调用链一直到AbstarctApplicationContext类。

```java
public AbstractApplicationContext(@Nullable ApplicationContext parent) {
   //设置资源文件解析器
   this();
   //设置父容器
   setParent(parent);
}
```

```java
//this();
public AbstractApplicationContext() {
   this.resourcePatternResolver = getResourcePatternResolver();
}

// getResourcePatternResolver();
protected ResourcePatternResolver getResourcePatternResolver() {
    //支持Ant风格的路径解析
	return new PathMatchingResourcePatternResolver(this);
}
```

```java
public void setParent(@Nullable ApplicationContext parent) {
   //设置父容器
   this.parent = parent;
   if (parent != null) {
       //父容器的环境设置到parentEnvironment属性中
       Environment parentEnvironment = parent.getEnvironment();
      if (parentEnvironment instanceof ConfigurableEnvironment) {
         getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
      }
   }
}
```

# setConfigLocations(configLocations)

传入的路径可能包含占位符等，需要进行解析，对占位符进行替换。

```java
public void setConfigLocations(@Nullable String... locations) {
   if (locations != null) {
      Assert.noNullElements(locations, "Config locations must not be null");
      this.configLocations = new String[locations.length];
      for (int i = 0; i < locations.length; i++) {
         //解析路径
         this.configLocations[i] = resolvePath(locations[i]).trim();
      }
   }
   else {
      this.configLocations = null;
   }
}
```

```java
protected String resolvePath(String path) {
   //获取Environment，进行解析，将路径中的占位符进行解析
   return getEnvironment().resolveRequiredPlaceholders(path);
}
```

getEnvironment()中如果this.environment为空则调用createEnvironment()方法创建新的StandardEnvironment对象。

```java
protected ConfigurableEnvironment createEnvironment() {
    return new StandardEnvironment();
}
```

# 大名鼎鼎的refresh()

该方法是对ApplicationContext加载的主函数。

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备刷新上下文环境
			prepareRefresh();

			// 初始化BeanFactory，并进行xml文件读取
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 对BeanFactory进行各种功能填充
			prepareBeanFactory(beanFactory);

			try {
				// 子类覆盖父类方法做额外处理
				postProcessBeanFactory(beanFactory);

				// 激活各种BeanFactory处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册拦截Bean创建的Bean处理器，这里只是注册，真正的调用是在getBean的时候
				registerBeanPostProcessors(beanFactory);

				// 为上下文初始化Message源，国际化处理
				initMessageSource();

				// 初始化应用消息广播器，并入放applicationEventMulticaster中
				initApplicationEventMulticaster();

				// 留给子类来初始化其他的Bean
				onRefresh();

				// 在所有注册的bean中查找Listener bean，注册到消息广播中
				registerListeners();

				// 初始化剩下的单例（非惰性的）
				finishBeanFactoryInitialization(beanFactory);

				// 完成刷新过程，通知生命周期处理器lifecycleRrocessor刷新过程
				// 同时发出ContextRefreshEvent通知别人
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

```


   1. 初始化前的准备工作，例如对系统属性或者环境变量进行准备及验证。

      在某种情况下项目的使用需要读取某些系统变量,而这个变量的设置很可能会影响着系统的正确性，那么ClassPathXmlIApplicationContext 为我们提供的这个准备函数就显得非常必要，它可以在Spring启动的时候提前对必需的变量进行存在性验证。

2. 初始化BeanFactory,并进行XML文件读取。

   之前有提到ClassPathXmlApplicationContext包含着BeanFactory所提供的一切特征，那么在这一步骤中将会复用BeanFactory 中的配置文件读取解析及其他功能，这-一步之后，ClassPathXmlApplicationContext实际上就已经包含了BeanFactory 所提供的功能，也就是可以进行bean的提取等基础操作了。

3. 对BeanFactory进行各种功能填充。

​      @Qualifer与@Autowired这两个非常熟悉的注解,正是在这一步骤中增加的支持。

4. 子类覆盖方法做额外的处理。

   一个扩展点，提供了一个空的函数实现postProcessBeanFactory来方便程序员在业务上做进一步扩展。

5. 激活各种BeanFactory处理器。

6. 注册拦截bean创建的bean处理器，这里只是注册，真正的调用是在getBean时候。

7. 为上下文初始化Message源，即对不同语言的消息体进行国际化处理。

8. 初始化应用消息广播器，并放入“applicationEventMulticaster” bean 中。

9. 留给子类来初始化其他的bean。

10. 在所有注册的bean中查找listener bean,注册到消息广播器中。

11. 初始化剩下的单实例(非惰性的)。

12. 完成刷新过程，通知生命周期处理器lifecycleProcessor 刷新过程，同时发出Context-RefreshEvent通知别人。

## 环境准备：prepareRefresh()

```java
protected void prepareRefresh() {
   // Switch to active.
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   // 留给子类覆盖
   initPropertySources();

   // 验证需要的属性文件是否都已经加入到环境中
   getEnvironment().validateRequiredProperties();

   // Store pre-refresh ApplicationListeners...
   if (this.earlyApplicationListeners == null) {
      this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
   }
   else {
      // Reset local application listeners to pre-refresh state.
      this.applicationListeners.clear();
      this.applicationListeners.addAll(this.earlyApplicationListeners);
   }

   // Allow for the collection of early ApplicationEvents,
   // to be published once the multicaster is available...
   this.earlyApplicationEvents = new LinkedHashSet<>();
}

```

 initPropertySources方法并没有实际的方法体，他是spring框架开放式的表现，当用户需要在spring真正启动前做一些设置，可以继承并重写该方法。

## 初始化BeanFactory：obtainFreshBeanFactory()

ApplicationContext包含了全部的BeanFactory功能，正是通过这个方法实现的。

```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		//初始化BeanFactory，并进行xml文件读取，并将得到的BeanFactory记录到当前实体的属性中
		refreshBeanFactory();
		//返回当前实体的BeanFactory属性
		return getBeanFactory();
	}

```

进一步调用refreshBeanFactory()方法。

```java
protected final void refreshBeanFactory() throws BeansException {
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
      //创建DefaultListableBeanFactory
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      //设置序列化id
      beanFactory.setSerializationId(getId());
      //定制beanFactory，设置相关属性，包含是否允许覆盖同名的不同定义的对象，
      //以及是否允许循环依赖
      customizeBeanFactory(beanFactory);
      //初始化DocumentReader，并进行xml文件读取和解析，加载BeanDefinition
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}

```

### 定制化BeanFactory：customizeBeanFactory(beanFactory)

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
   //允许重名覆盖
   if (this.allowBeanDefinitionOverriding != null) {
      beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   }
   //允许循环依赖
   if (this.allowCircularReferences != null) {
      beanFactory.setAllowCircularReferences(this.allowCircularReferences);
   }
}

```

### 加载BeanDefinition：loadBeanDefinitions(beanFactory)

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   // 创建beanDefinitionReader,这里的beanFactory就是registry。。。
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // 对beanDefinitionReader进行环境变量设置
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // 初始化beanDefinitionReader，子类可以覆盖进行扩展
   initBeanDefinitionReader(beanDefinitionReader);
   //调用beanDefinitionReader的loadBeanDefinitions方法，然后就是xmlBeanFactory解析的过程了
   loadBeanDefinitions(beanDefinitionReader);
}
```

首先获取XmlBeanDefinitionReader对象，然后对其进行必要的设置，最后调用loadBeanDefinitions方法，将bean解析后注册到DefaultListableBeanFactory中。

## 功能扩展：prepareBeanFactory(beanFactory)

在此方法执行之前beanFactory已经初始化完成。所以这个方法主要是对BeanFactory进行一些功能上的扩展。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // 设置classLoader
   beanFactory.setBeanClassLoader(getClassLoader());
   //设置SPEL解析器
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   //为beanFactory增加一个propertyEditor
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // 添加BeanPostProcessor
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   //设置几个忽略自动装配的接口
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // 设置几个自动装配的特殊实现
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // 添加BeanPostProcessor
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // 增加AspectJ支持
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // 增加默认的Environment Bean
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```

1. 增加SpEL语言的支持
2. 增加对属性编辑器的支持
3. 增加一些内置类，比如：EnvironmentAware，MessageSourceAware的信息注入。
4. 设置依赖功能可忽略的接口
5. 注册一些固定依赖的属性
6. 增加AspectJ的支持
7. 将相关环境变量及属性注册以单例模式注册

### 属性编辑器的注册

`beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));`

进行默认属性编辑器注册的方法是ResourceEditorRegistrar中的registerCustomEditors方法。

```java
	public void registerCustomEditors(PropertyEditorRegistry registry) {
		ResourceEditor baseEditor = new ResourceEditor(this.resourceLoader, this.propertyResolver);
		doRegisterEditor(registry, Resource.class, baseEditor);
		doRegisterEditor(registry, ContextResource.class, baseEditor);
		doRegisterEditor(registry, InputStream.class, new InputStreamEditor(baseEditor));
		doRegisterEditor(registry, InputSource.class, new InputSourceEditor(baseEditor));
		doRegisterEditor(registry, File.class, new FileEditor(baseEditor));
		doRegisterEditor(registry, Path.class, new PathEditor(baseEditor));
		doRegisterEditor(registry, Reader.class, new ReaderEditor(baseEditor));
		doRegisterEditor(registry, URL.class, new URLEditor(baseEditor));

		ClassLoader classLoader = this.resourceLoader.getClassLoader();
		doRegisterEditor(registry, URI.class, new URIEditor(classLoader));
		doRegisterEditor(registry, Class.class, new ClassEditor(classLoader));
		doRegisterEditor(registry, Class[].class, new ClassArrayEditor(classLoader));

		if (this.resourceLoader instanceof ResourcePatternResolver) {
			doRegisterEditor(registry, Resource[].class,
					new ResourceArrayPropertyEditor((ResourcePatternResolver) this.resourceLoader, this.propertyResolver));
		}
	}
```

该放在AbstractBeanFactory类的initBeanWrapper中被调用。

### 增加一些内置类

`beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));`

ApplicationContextAwareProcessor是一个BeanPostProcessor。

对各种Aware接口的支持。

```java
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
			return bean;
		}

		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                //增加资源支持
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

```

```java
//为bean增加各种支持
private void invokeAwareInterfaces(Object bean) {
   if (bean instanceof EnvironmentAware) {
      ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
   }
   if (bean instanceof EmbeddedValueResolverAware) {
      ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
   }
   if (bean instanceof ResourceLoaderAware) {
      ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
   }
   if (bean instanceof ApplicationEventPublisherAware) {
      ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
   }
   if (bean instanceof MessageSourceAware) {
      ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
   }
   if (bean instanceof ApplicationContextAware) {
      ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
   }
}

```

## BeanFactoryPostProcessor的调用

`this.invokeBeanFactoryPostProcessors(beanFactory);`对spring中注册的BeanFactoryPostProcessors进行invoke。

该方法通过委托PostProcessorRegistrationDelegate类的invokeBeanFactoryPostProcessors方法进行执行。主要思路为先遍历硬编码的BeanFactoryPostProcessors，即通过AbstractApplicationContext的addBeanFactoryPostProcessors方法添加的。

然后就是自定义的BeanFactoryPostProcessors执行了，主要原理是通过对beanFactory中注册beanDefinition进行遍历，只要是实现了对应接口就会执行。。。

执行时如果实现了BeanDefinitionRegistryPostProcessor接口，那么会先执行postProcessBeanDefinitionRegistry方法，然后才是BeanFactoryPostProcessors接口中的方法。

## 注册BeanPostProcessors

`this.registerBeanPostProcessors(beanFactory);`

原理与上面的差不多，但是这个只是注册不涉及到调用所以没有对硬编码的处理。

注册原理也是对beanDefinitions进行遍历，将对应类型的的beanName进行注册。

## 容器内事件原理

在refresh方法中`initApplicationEventMulticaster();`，对事件广播器进行初始化。

当调用applicationContext.publishEvent(event)方法时，内部会调用ApplicationEventMulticaster的事件广播方法，遍历所有监听器执行监听方法。

监听器的注册是在refresh方法的`registerListeners();`中，原理也是对BeanDefinitions进行遍历，根据类型匹配进行注册。

## 初始化非延迟单例

`finishBeanFactoryInitialization(beanFactory);`

该方法将所有非延迟加载的单例bean进行加载。除了加载bean的功能还有进行ConversionService配置和冻结bean定义配置。

```java
public void preInstantiateSingletons() throws BeansException {
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }

   // Iterate over a copy to allow for init methods which in turn register new bean definitions.
   // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   // Trigger initialization of all non-lazy singleton beans...
   for (String beanName : beanNames) {
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
         if (isFactoryBean(beanName)) {
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                              ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
             //看着里。。。。。。。
            getBean(beanName);
         }
      }
   }

   // Trigger post-initialization callback for all applicable beans...
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

# 