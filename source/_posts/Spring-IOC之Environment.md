---
title: Spring-IOC之Environment
author: XIA
categories:
  - null
tags:
  - null
date: 2020-01-31 17:10:40
---

# 概览

Environmen接口**代表了当前应用所处的环境。**其主要和profile、Property相关。

StandardEnvironment为Environment接口的核心实现类。

# StandardEnvironment结构

![image-20200131171345020](https://xbxblog2.bj.bcebos.com/Spring-IOC%E4%B9%8BEnvironment%2Fimage-20200131171345020.png)

在非web应用中常用StandardEnvironment实现类，在web应用中常用StandardServletEnvironment实现类。其区别为StandardServletEnvironment多了些web相关的propertySource。

+ PropertyResolver：用于properties的操作，他提供了Environment关于properties功能的定义。

  ```java
  public interface PropertyResolver {
  
  	boolean containsProperty(String key);
  
  	String getProperty(String key);
  
  	String getRequiredProperty(String key) throws IllegalStateException;
  
  	<T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException;
  
  	String resolvePlaceholders(String text);
  
  	String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;
  
  }
  ```

+ Environment：提供profile操作的能力。

  ```java
  public interface Environment extends PropertyResolver {
  
  	String[] getActiveProfiles();
  
  	String[] getDefaultProfiles();
  
  	boolean acceptsProfiles(Profiles profiles);
  
  }
  ```

+ ConfigurablePropertyResolver：提供访问和定义ConversionService和设置占位符的方法。

  ```java
  public interface ConfigurablePropertyResolver extends PropertyResolver {
  
  	ConfigurableConversionService getConversionService();
  
  	void setConversionService(ConfigurableConversionService conversionService);
  
  	void setPlaceholderPrefix(String placeholderPrefix);
  
  	void setPlaceholderSuffix(String placeholderSuffix);
  
  	void setValueSeparator(@Nullable String valueSeparator);
  
  	void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders);
  
  	void setRequiredProperties(String... requiredProperties);
  
  	void validateRequiredProperties() throws MissingRequiredPropertiesException;
  
  }
  ```

+ 提供设置active和default Profile和操作PropertySources的方法。

  ```java
  public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {
  
  	void setActiveProfiles(String... profiles);
  
  	void addActiveProfile(String profile);
  
  	void setDefaultProfiles(String... profiles);
  
  	MutablePropertySources getPropertySources();
  
  	Map<String, Object> getSystemProperties();
  
  	Map<String, Object> getSystemEnvironment();
  
  	void merge(ConfigurableEnvironment parent);
  
  }
  
  ```

+ AbstractEnvironment：Environment的抽象实现，基本实现了完整的Environment的frofile和properties功能，提供了模板方法customizePropertySources来设置MutablePropertySources包含的propertySources。

+ StandardEnvironment：直接继承自AbstractEnvironment，重写模板方法customizePropertySources，其他功能直接使用AbstractEnvironment中的实现。

  ````java
  public class StandardEnvironment extends AbstractEnvironment {
  
  	/** System environment property source name: {@value}. */
  	public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";
  
  	/** JVM system properties property source name: {@value}. */
  	public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";
  
  	@Override
  	protected void customizePropertySources(MutablePropertySources propertySources) {
  		propertySources.addLast(
  				new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
  		propertySources.addLast(
  				new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
  	}
  
  }
  ````

# MutablePropertySources与PropertySource

AbstractEnvironment中维护着一个MutablePropertySources对象，MutablePropertySources负责维护PropertySource，内部维护着一个CopyOnWriteArrayList用来存储PropertySource对象。

![image-20200131192211396](https://xbxblog2.bj.bcebos.com/Spring-IOC%E4%B9%8BEnvironment%2Fimage-20200131192211396.png)

```java
public interface PropertySources extends Iterable<PropertySource<?>> {

	default Stream<PropertySource<?>> stream() {
		return StreamSupport.stream(spliterator(), false);
	}

	boolean contains(String name);

	@Nullable
	PropertySource<?> get(String name);

}
```

**PropertySource**是一个KV形式的对象。保存着Property。

```java
public abstract class PropertySource<T> {

	protected final Log logger = LogFactory.getLog(getClass());

	protected final String name;

	protected final T source;
	
    ............
}
```

总的来说就是：

AbstractEnvironment中维护着一个MutablePropertySources对象，MutablePropertySources维护着一个保存PropertySource的列表。



# 总结

Environmen总的接口就是这样，用来保存profile和property。