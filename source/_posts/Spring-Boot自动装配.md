---
title: Spring-Boot自动装配
author: XIA
categories:
  - null
tags:
  - null
date: 2020-02-15 17:21:32
---

# @SpringBootApplication结构

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
  .......
}
```

可见该注解由三个元注解组成  @SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan。

**@SpringBootConfiguration**就是@Configuration加了一个外壳。其意义为@SpringBootApplication注解的类可以被@ComponentScan扫描到。

```java
@Configuration
public @interface SpringBootConfiguration {

}
```

**@EnableAutoConfiguration**

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```

@Import(AutoConfigurationImportSelector.class)

spring内部使用@Import注解来实现Enable功能。所以AutoConfigurationImportSelector类的selectImports方法将返回将要进行导入的类。

**todo:@AutoConfigurationPackage**

![image-20200215180610219](C:\Users\xiabx\AppData\Roaming\Typora\typora-user-images\image-20200215180610219.png)

# Spring Boot自动装配原理

AutoConfigurationImportSelector#selectImports

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
			.loadMetadata(this.beanClassLoader); //加载自动装配的元信息
	AnnotationAttributes attributes = getAttributes(annotationMetadata); //获取@EnableAutoConfiguration标注类的元信息
	List<String> configurations = getCandidateConfigurations(annotationMetadata,
			attributes); //返回导入类名集合
	configurations = removeDuplicates(configurations); //移除重复对象
	Set<String> exclusions = getExclusions(annotationMetadata, attributes); // 获取排除名单
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	configurations = filter(configurations, autoConfigurationMetadata); //过滤
	fireAutoConfigurationImportEvents(configurations, exclusions); //触发自动装配的导入事件
	return StringUtils.toStringArray(configurations);
}
```

## 读取候选组件

**AutoConfigurationImportSelector#getCandidateConfigurations**该方法将返回定义在MATE-INF/spring.factories文件中key为`org.springframework.boot.autoconfigure.EnableAutoConfiguration`的实现类全类名列表。

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
		AnnotationAttributes attributes) {
    //getSpringFactoriesLoaderFactoryClass() ---> EnableAutoConfiguration.class;
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
			getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
	return configurations;
}
```

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}
```

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    //获取classLoader下所有MATE-INF/spring.factories文件url
    Enumeration<URL> urls = (classLoader != null ?
                             classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
    result = new LinkedMultiValueMap<>();
    while (urls.hasMoreElements()) {
        URL url = urls.nextElement();
        UrlResource resource = new UrlResource(url);
        Properties properties = PropertiesLoaderUtils.loadProperties(resource);
        for (Map.Entry<?, ?> entry : properties.entrySet()) {
            List<String> factoryClassNames = Arrays.asList(
                StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
            result.addAll((String) entry.getKey(), factoryClassNames);
        }
    }

    return result;
		
}
```

MATE-INF/spring.factories是一个properties文件，key为接口全名，value为实现类全类名列表：

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
```

**移除重复对象**

在获取实现类全类名列表后接下来就是去重了，去重原理是利用set的特性。

```java
protected final <T> List<T> removeDuplicates(List<T> list) {
    return new ArrayList<>(new LinkedHashSet<>(list));
}
```

## 排除自动装配组件

`AutoConfigurationImportSelector#getExclusions`方法执行后会获得一个自动装配class的排除名单。

将注解@EnableAutoConfiguration配置类的注解属性exclude和excludeName以及spring.autoconfigure.exclude配置累加到excluded集合。

```java
protected Set<String> getExclusions(AnnotationMetadata metadata,
                                    AnnotationAttributes attributes) {
    Set<String> excluded = new LinkedHashSet<>();
    excluded.addAll(asList(attributes, "exclude"));
    excluded.addAll(Arrays.asList(attributes.getStringArray("excludeName")));
    excluded.addAll(getExcludeAutoConfigurationsProperty());
    return excluded;
}
```

接着对**排除名单进行校验**。

`AutoConfigurationImportSelector#checkExcludedClasses`

```java
private void checkExcludedClasses(List<String> configurations,
                                  Set<String> exclusions) {
    List<String> invalidExcludes = new ArrayList<>(exclusions.size());
    for (String exclusion : exclusions) {
        //classLoader中存在被排除class && 自动加载名单里没有
        if (ClassUtils.isPresent(exclusion, getClass().getClassLoader())
            && !configurations.contains(exclusion)) {
            invalidExcludes.add(exclusion);
        }
    }
    //存在不合法的  抛异常
    if (!invalidExcludes.isEmpty()) {
        handleInvalidExcludes(invalidExcludes);
    }
}
```

随后自动加载类名单将删除配排除名单`configurations.removeAll(exclusions);`

## 过滤自动装配组件

这个方式的过滤与上一种的不同之处在于，上一种通过手动指定排除组件，这一种则是通过springboot内置的过滤机制进行过滤。

`AutoConfigurationImportSelector#filter`

```java
private List<String> filter(List<String> configurations,
			AutoConfigurationMetadata autoConfigurationMetadata) {
    long startTime = System.nanoTime();
    String[] candidates = StringUtils.toStringArray(configurations);
    boolean[] skip = new boolean[candidates.length];
    boolean skipped = false;
    //获取过滤器
    for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
        //当filter实现各种Aware接口时，为其注入所需依赖
        invokeAwareMethods(filter);
        //主要方法，执行过滤
        boolean[] match = filter.match(candidates, autoConfigurationMetadata);
        for (int i = 0; i < match.length; i++) {
            if (!match[i]) {
                skip[i] = true;
                skipped = true;
            }
        }
    }
    if (!skipped) {
        return configurations;
    }
    List<String> result = new ArrayList<>(candidates.length);
    for (int i = 0; i < candidates.length; i++) {
        if (!skip[i]) {
            result.add(candidates[i]);
        }
    }
    if (logger.isTraceEnabled()) {
        int numberFiltered = configurations.size() - result.size();
        logger.trace("Filtered " + numberFiltered + " auto configuration class in "
                     + TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)
                     + " ms");
    }
    return new ArrayList<>(result);
}
```

**获取过滤器**

在MATE-INF/spring.factories中获取key为org.springframework.boot.autoconfigure.AutoConfigurationImportFilter的实例集合。

```java
protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
    return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class,
                                               this.beanClassLoader);
}
```

SpringBoot中只内置了一个实现类：OnClassCondition。

**执行过滤**

这里用到了autoConfigurationMetadata属性，它是在AutoConfigurationImportSelector#selectImports中被初始化。`AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader      .loadMetadata(this.beanClassLoader);`

```java
//path -->  META-INF/spring-autoconfigure-metadata.properties
static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {

    Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(path)
                             : ClassLoader.getSystemResources(path));
    Properties properties = new Properties();
    while (urls.hasMoreElements()) {
        properties.putAll(PropertiesLoaderUtils
                          .loadProperties(new UrlResource(urls.nextElement())));
    }
    return loadMetadata(properties);
}
```

`META-INF/spring-autoconfigure-metadata.properties`是一个properties文件，这里所用的内容为自动装配类后加`.ConditionalOnClass`的内容，其含义根据后续分析为该自动装配类所依赖类的全类名，后续的`OnClassCondition#match`方法会根据依赖的类是否存在决定是否将该自动装配类过滤：

```properties
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration.ConditionalOnClass=javax.sql.DataSource,org.springframework.jdbc.core.JdbcTemplate
```

`OnClassCondition#match`是执行过滤的主要方法。

```java
public boolean[] match(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata) {
    
    //委托给getOutcomes方法获取处理结果
    ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses,
                                              autoConfigurationMetadata);
    boolean[] match = new boolean[outcomes.length];
    for (int i = 0; i < outcomes.length; i++) {
        match[i] = (outcomes[i] == null || outcomes[i].isMatch());
        if (!match[i] && outcomes[i] != null) {
            logOutcome(autoConfigurationClasses[i], outcomes[i]);
           
        }
    }
    return match;
}
```

```java
private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
                                       AutoConfigurationMetadata autoConfigurationMetadata) {
    // 将autoConfigurationClasses分为两份，用两个线程来执行，提高性能
    int split = autoConfigurationClasses.length / 2;
    //会创建新线程，用来分析前一半autoConfigurationClasses
    OutcomesResolver firstHalfResolver = createOutcomesResolver(
        autoConfigurationClasses, 0, split, autoConfigurationMetadata);
    //用main线程分析后一半
    OutcomesResolver secondHalfResolver = new StandardOutcomesResolver(
        autoConfigurationClasses, split, autoConfigurationClasses.length,
        autoConfigurationMetadata, this.beanClassLoader);
    //分析的方法
    ConditionOutcome[] secondHalf = secondHalfResolver.resolveOutcomes();
    ConditionOutcome[] firstHalf = firstHalfResolver.resolveOutcomes();
    ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];
    System.arraycopy(firstHalf, 0, outcomes, 0, firstHalf.length);
    System.arraycopy(secondHalf, 0, outcomes, split, secondHalf.length);
    return outcomes;
}
```

```java
//StandardOutcomesResolver#resolveOutcomes
public ConditionOutcome[] resolveOutcomes() {
    return getOutcomes(this.autoConfigurationClasses, this.start, this.end,
                       this.autoConfigurationMetadata);
}


private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
                                       int start, int end, AutoConfigurationMetadata autoConfigurationMetadata) {
    ConditionOutcome[] outcomes = new ConditionOutcome[end - start];
    for (int i = start; i < end; i++) {
        String autoConfigurationClass = autoConfigurationClasses[i];
        //获取autoConfigurationMetadata中的值，然后转化为set集合，此处的key为autoConfigurationClass.ConditionalOnClass
        //如autoConfigurationClass为org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration那么对应的key就为org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration.ConditionalOnClass     
        Set<String> candidates = autoConfigurationMetadata
            .getSet(autoConfigurationClass, "ConditionalOnClass");
        if (candidates != null) {
            //获取结果，就是看看candidates中的类是不是都存在。。。
            outcomes[i - start] = getOutcome(candidates);
        }
    }
    return outcomes;
}
```

```java
//OnClassCondition#getOutcome
private ConditionOutcome getOutcome(Set<String> candidates) {

    List<String> missing = getMatches(candidates, MatchType.MISSING,
                                      this.beanClassLoader);
    if (!missing.isEmpty()) {
        return ConditionOutcome.noMatch(
            ConditionMessage.forCondition(ConditionalOnClass.class)
            .didNotFind("required class", "required classes")
            .items(Style.QUOTE, missing));
    }

    return null;
}


private List<String> getMatches(Collection<String> candidates, MatchType matchType,
                                ClassLoader classLoader) {
    List<String> matches = new ArrayList<>(candidates.size());
    for (String candidate : candidates) {
        //当不存在该candidate时，加入matches
        if (matchType.matches(candidate, classLoader)) {
            matches.add(candidate);
        }
    }
    return matches;
}

//以下是怎么判断该类是否存在的方法。利用classLoader加载，抛异常了就是不存在的。
MISSING {

    @Override
    public boolean matches(String className, ClassLoader classLoader) {
        return !isPresent(className, classLoader);
    }

};

private static boolean isPresent(String className, ClassLoader classLoader) {
    if (classLoader == null) {
        classLoader = ClassUtils.getDefaultClassLoader();
    }
    try {
        forName(className, classLoader);
        return true;
    }
    catch (Throwable ex) {
        return false;
    }
}

private static Class<?> forName(String className, ClassLoader classLoader)
    throws ClassNotFoundException {
    if (classLoader != null) {
        return classLoader.loadClass(className);
    }
    return Class.forName(className);
}
```

所以这里自动过滤的流程主要分为以下几步：获取过滤器，从AutoConfigurationMetadata 获取自动配置类的依赖类，根据依赖类是否都存在来确定是否过滤掉该自动装配类。

## 自动装配事件

AutoConfigurationImportSelector#fireAutoConfigurationImportEvents

```java
private void fireAutoConfigurationImportEvents(List<String> configurations,
                                               Set<String> exclusions) {
    //获取监听器，同样是通过MATE-INF/spring.factories文件，此时key为org.springframework.boot.autoconfigure.AutoConfigurationImportListener
    List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();
    if (!listeners.isEmpty()) {
        //创建事件对象
        AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this,
                                                configurations, exclusions);
        for (AutoConfigurationImportListener listener : listeners) {
            //实现Aware接口的话进行注入
            invokeAwareMethods(listener);
            //执行事件方法
            listener.onAutoConfigurationImportEvent(event);
        }
    }
}
```

# DeferredImportSelector

AutoConfigurationImportSelector实现DeferredImportSelector接口。该接口与ImportSelector的区别在于延迟功能，在ConfigurationClassParser#processImports方法中，当被导入的类是DeferredImportSelector类型时，则会将该类先加入一个集合中，然后在所有配置类处理完成后再处理这个延迟集合中的配置类。

![image-20200216204932115](C:\Users\xiabx\AppData\Roaming\Typora\typora-user-images\image-20200216204932115.png)

加入延迟处理集合：

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,

       ................
                            
   if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
      this.deferredImportSelectors.add(
         new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
   } 
     ..................
}
```

处理延迟处理集合：

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<>();

    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if 。。。。。。。
    }

    //处理延迟集合
    processDeferredImportSelectors();
}
```





















