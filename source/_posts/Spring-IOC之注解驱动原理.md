---
title: Spring-IOC之注解驱动原理
author: XIA
categories:
  - null
tags:
  - null
date: 2020-02-02 19:58:17
---

# <context:component-scan base-package="com.*">`

让我们回到spring解析xml的流程，在`DefaultBeanDefinitionDocumentReader.parseBeanDefinitions`方法中，会根据名字空间将解析分为默认名字空间的解析和自定义标签的解析，显然这里要走的是自定义标签的解析。所以我们找到了这个这个标签解析的入口，下面让本鸟一步步跟进去看看到底注解驱动的原理是什么？

关于自定义标签的解析流程在上面已经进行了总结，简单说就是根据自定义标签的名字空间uri查询spring.handlers文件找到解析方法并执行其中定义的init方法，init方法中一般都是对localName（localName是个什么东西呢，比如对于context:annotation-config标签就是annotation-config）的解析器注册。接下来根据localName获取对应解析器执行解析方法parse。

![1570979542838](https://xbxblog.bj.bcebos.com/%E5%A4%96%E9%83%A8%E8%A7%A3%E6%9E%90.png)

**注册各种localName的解析器**

```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {
   @Override
   public void init() {
      registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
      registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
      registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
      registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
      registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
      registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
      registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
      registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
   }
}
```

所以我们找到了解析这个标签的入口方法：ComponentScanBeanDefinitionParser.parse。

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
   //获取 base-package属性
   String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
   //解析占位符
   basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
   //分割为数组
   String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
         ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

   // Actually scan for bean definitions and register them.
   //初始化扫描器
   ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
   //执行扫描
   Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
   registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

   return null;
}
```

在初始化扫描器之前所做的工作都是对base-package属性进行处理，包括解析占位符和分割为数组形式。

## 初始化扫描器

`ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);`

```java
protected ClassPathBeanDefinitionScanner configureScanner(ParserContext parserContext, Element element) {
   //设置为否将不会扫描@Component、@Repository、@Service和@Controller
   boolean useDefaultFilters = true;
   if (element.hasAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE)) {
      useDefaultFilters = Boolean.parseBoolean(element.getAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE));
   }

   // Delegate bean definition registration to scanner class.
   //将bean定义注册到扫描仪类。
   ClassPathBeanDefinitionScanner scanner = createScanner(parserContext.getReaderContext(), useDefaultFilters);
   scanner.setBeanDefinitionDefaults(parserContext.getDelegate().getBeanDefinitionDefaults());
   scanner.setAutowireCandidatePatterns(parserContext.getDelegate().getAutowireCandidatePatterns());

   //用以配置扫描器扫描的路径，默认**/*.class
   if (element.hasAttribute(RESOURCE_PATTERN_ATTRIBUTE)) {
      scanner.setResourcePattern(element.getAttribute(RESOURCE_PATTERN_ATTRIBUTE));
   }

   //设置名字生成策略
   try {
      parseBeanNameGenerator(element, scanner);
   }
   catch (Exception ex) {
      parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
   }


   try {
      parseScope(element, scanner);
   }
   catch (Exception ex) {
      parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
   }

   //处理过滤器
   parseTypeFilters(element, scanner, parserContext);

   return scanner;
}
```

## 执行扫描

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   //逐包扫描
   for (String basePackage : basePackages) {
       //主要方法
       //扫描basePackage下的class，将其包装为BeanDefinition
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         //设置bean的scope,解析@Scope注解
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         //生成beanName
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);

         if (candidate instanceof AbstractBeanDefinition) {
            //为BeanDefinition设置默认属性值
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         //其他注解解析
         if (candidate instanceof AnnotatedBeanDefinition) {
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         //将BeanDefinition添加到仓库
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```

### 扫描basePackage

最终调用此方法对basePackage进行扫描注册

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
   Set<BeanDefinition> candidates = new LinkedHashSet<>();
   try {
      String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
       //根据匹配模式扫描basePackage下的class文件，将其封装为Resource类型
      Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
      boolean traceEnabled = logger.isTraceEnabled();
      boolean debugEnabled = logger.isDebugEnabled();
      for (Resource resource : resources) {
         if (resource.isReadable()) {
            try {
                //根据resource获取MatedataReader。
                //模式注解的多层派生性，原理就是从这里。涉及asm库的内容。
               MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                //1
               if (isCandidateComponent(metadataReader)) {
                   //创建BeanDefiniton对象，ScannedGenericBeanDefinition是AnnotatedBeanDefinition子类
                  ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                  sbd.setResource(resource);
                  sbd.setSource(resource);
                   //2
                  if (isCandidateComponent(sbd)) {
                     candidates.add(sbd);
                  }
               }
      }
   }
   return candidates;
}
```

一个被扫描的类能否作为组件有两个isCandidateComponent方法决定：

```java
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
				return false;
			}
		}
        //这里includeFilters在ClassPathBeanDefinitionScanner实例化时被设置，
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
				return isConditionMatch(metadataReader);
			}
		}
		return false;
	}

	//设置includeFilters
	protected void registerDefaultFilters() {
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) cl.loadClass("javax.annotation.ManagedBean")), false));
			logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) cl.loadClass("javax.inject.Named")), false));
			logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```

#### 主要涉及类

ClassMetadata     AnnotationMetadata    MetadataReader    

<img src="https://xbxblog2.bj.bcebos.com/springioc注解驱动原理/image-20200205143616060.png" alt="image-20200205143616060"  />

**ClassMetadata：**表示class的元数据类，此接口有两个重要实现类：StandardClassMetadata和ClassMetadataReadingVisitor。这两个实现类的区别在于前者使用jdk实现后者使用asm库，使用jdk的优势是实现起来较简单，但是需要将class加载到内存，而asm的优点则是无需将class加载到内存，这样可以提高性能。

```java
public interface ClassMetadata {

	// 返回类名（注意返回的是最原始的那个className）
	String getClassName();
	boolean isInterface();
	// 是否是注解
	boolean isAnnotation();
	boolean isAbstract();
	// 是否允许创建  不是接口且不是抽象类  这里就返回true了
	boolean isConcrete();
	boolean isFinal();
	// 是否是独立的(能够创建对象的)  比如是Class、或者内部类、静态内部类
	boolean isIndependent();
	// 是否有内部类之类的东东
	boolean hasEnclosingClass();
	@Nullable
	String getEnclosingClassName();
	boolean hasSuperClass();
	@Nullable
	String getSuperClassName();
	// 会把实现的所有接口名称都返回  具体依赖于Class#getSuperclass
	String[] getInterfaceNames();

	// 基于：Class#getDeclaredClasses  返回类中定义的公共、私有、保护的内部类
	String[] getMemberClassNames();
}
```

**AnnotatedTypeMetadata：**对注解元素的封装，什么叫注解元素(AnnotatedElement)？比如我们常见的Class、Method、Constructor、Parameter等等都属于它的子类都属于注解元素。简单理解：只要能在上面标注注解都属于这种元素。Spring4.0新增的这个接口提供了对注解统一的、便捷的访问，使用起来更加的方便高效了。

```java
public interface AnnotatedTypeMetadata {

	// 此元素是否标注有此注解
	// annotationName：注解全类名
	boolean isAnnotated(String annotationName);
	
	// 取得指定类型注解的所有的属性 - 值（k-v）
	// annotationName：注解全类名
	// classValuesAsString：若是true表示 Class用它的字符串的全类名来表示。这样可以避免Class被提前加载
	@Nullable
	Map<String, Object> getAnnotationAttributes(String annotationName);
	@Nullable
	Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString);

	// 参见这个方法的含义：AnnotatedElementUtils.getAllAnnotationAttributes
	@Nullable
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName);
	@Nullable
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString);
}
```

**AnnotationMetadata：**这是理解`Spring`注解编程的必备知识，它是`ClassMetadata`和`AnnotatedTypeMetadata`的子接口，具有两者共同能力，并且新增了访问注解的相关方法。可以简单理解为它是对注解的抽象。

该接口有两个实现，同样也是分别基于反射和asm库。

```java
public interface AnnotationMetadata extends ClassMetadata, AnnotatedTypeMetadata {

	//拿到当前类上所有的注解的全类名（注意是全类名）
	Set<String> getAnnotationTypes();
	// 拿到指定的注解类型
	//annotationName:注解类型的全类名
	Set<String> getMetaAnnotationTypes(String annotationName);
	
	// 是否包含指定注解 （annotationName：全类名）
	boolean hasAnnotation(String annotationName);
	//这个厉害了，用于判断注解类型自己是否被某个元注解类型所标注
	//依赖于AnnotatedElementUtils#hasMetaAnnotationTypes
	boolean hasMetaAnnotation(String metaAnnotationName);
	
	// 类里面只有有一个方法标注有指定注解，就返回true
	//getDeclaredMethods获得所有方法， AnnotatedElementUtils.isAnnotated是否标注有指定注解
	boolean hasAnnotatedMethods(String annotationName);
	// 返回所有的标注有指定注解的方法元信息。注意返回的是MethodMetadata 原理基本同上
	Set<MethodMetadata> getAnnotatedMethods(String annotationName);
}

```

**MetadataReader接口：**获取指定类的ClassMetadata和AnnotationMetadata的接口。他的实现类为default包访问权限。所以还需要使用一个工厂MetadataReaderFactory接口获取实例。

```java
public interface MetadataReader {
	// 返回此Class文件的来源（资源）
	Resource getResource();
	// 返回此Class的元数据信息
	ClassMetadata getClassMetadata();
	// 返回此类的注解元信息（包括方法的）
	AnnotationMetadata getAnnotationMetadata();
}
```

**参考：**<https://blog.csdn.net/f641385712/article/details/88765470>

### 生成beanName

```java
public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
    //扫描出来的BeanDefinition都是AnnotatedBeanDefinition类型
   if (definition instanceof AnnotatedBeanDefinition) {
      String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition);
      if (StringUtils.hasText(beanName)) {
         // Explicit bean name found.
         return beanName;
      }
   }
   // Fallback: generate a unique default bean name.
   return buildDefaultBeanName(definition, registry);
}
```

生成beanName首先根据注解信息生成，如果注解中没有指定，再使用默认策略生成。

**根据注解解析beanName**

```java
protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) {
   AnnotationMetadata amd = annotatedDef.getMetadata();
    //拿到当前类上所有的注解的全类名
   Set<String> types = amd.getAnnotationTypes();
   String beanName = null;
   for (String type : types) {
       //获取type类型注解的属性集合
      AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(amd, type);
       //amd.getMetaAnnotationTypes(type)，获取type类型的元注解
      if (attributes != null && isStereotypeWithNameValue(type, amd.getMetaAnnotationTypes(type), attributes)) {
          //获取value属性
         Object value = attributes.get("value");
         if (value instanceof String) {
            String strVal = (String) value;
            if (StringUtils.hasLength(strVal)) {
               if (beanName != null && !strVal.equals(beanName)) {
                  throw new IllegalStateException("Stereotype annotations suggest inconsistent " +
                        "component names: '" + beanName + "' versus '" + strVal + "'");
               }
                //注解的value属性就是beanName
               beanName = strVal;
            }
         }
      }
   }
   return beanName;
}

//判断此注解是否可以用来生成beanName
protected boolean isStereotypeWithNameValue(String annotationType,
      Set<String> metaAnnotationTypes, @Nullable Map<String, Object> attributes) {
	//是否符合，注解为component||元注解为component||注解类型为ManagedBean||注解类型为Named
   boolean isStereotype = annotationType.equals(COMPONENT_ANNOTATION_CLASSNAME) ||
         metaAnnotationTypes.contains(COMPONENT_ANNOTATION_CLASSNAME) ||
         annotationType.equals("javax.annotation.ManagedBean") ||
         annotationType.equals("javax.inject.Named");
	//他们的value属性就是beanName
   return (isStereotype && attributes != null && attributes.containsKey("value"));
}
```

所以@Component、以@Component作为元注解的注解、@ManagedBean、@Named的value属性就是beanName

**默认策略生成beanName**

类名转首字母小写驼峰

```java
protected String buildDefaultBeanName(BeanDefinition definition) {
   String beanClassName = definition.getBeanClassName();
   Assert.state(beanClassName != null, "No bean class name set");
   String shortClassName = ClassUtils.getShortName(beanClassName);
   return Introspector.decapitalize(shortClassName);
}
```

### 其他注解解析

解析@Lazy、@Primary等注解

```java
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
   AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
   if (lazy != null) {
      abd.setLazyInit(lazy.getBoolean("value"));
   }
   else if (abd.getMetadata() != metadata) {
      lazy = attributesFor(abd.getMetadata(), Lazy.class);
      if (lazy != null) {
         abd.setLazyInit(lazy.getBoolean("value"));
      }
   }

   if (metadata.isAnnotated(Primary.class.getName())) {
      abd.setPrimary(true);
   }
   AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
   if (dependsOn != null) {
      abd.setDependsOn(dependsOn.getStringArray("value"));
   }

   AnnotationAttributes role = attributesFor(metadata, Role.class);
   if (role != null) {
      abd.setRole(role.getNumber("value").intValue());
   }
   AnnotationAttributes description = attributesFor(metadata, Description.class);
   if (description != null) {
      abd.setDescription(description.getString("value"));
   }
}
```

### registerComponents

将`<context:annotation-config>`配置需要注册的组件进行注册。所以使用component-scan就不必使用annotation-config了

# `<context:annotation-config>`

## parse

AnnotationConfigBeanDefinitionParser.parse

```java
public class AnnotationConfigBeanDefinitionParser implements BeanDefinitionParser {
   @Override
   @Nullable
   public BeanDefinition parse(Element element, ParserContext parserContext) {
      Object source = parserContext.extractSource(element);

      // Obtain bean definitions for all relevant BeanPostProcessors.
      Set<BeanDefinitionHolder> processorDefinitions =
            AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);

      // Register component for the surrounding <context:annotation-config> element.
      CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
      parserContext.pushContainingComponent(compDefinition);

      // Nest the concrete beans in the surrounding component.
      for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
         parserContext.registerComponent(new BeanComponentDefinition(processorDefinition));
      }

      // Finally register the composite component.
      parserContext.popAndRegisterContainingComponent();

      return null;
   }

}
```

以上代码的关键语句为：`Set<BeanDefinitionHolder> processorDefinitions =
 AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);`其作用为将对注解进行解析的组件的类的beanDefinition注册到容器中。这些组件有的为BeanFactoryPostProcessor类型，有的为BeanPostProcessor类型。在refresh方法的后续代码中将回对他们分别执行或注册。

AnnotationConfigUtils.registerAnnotationConfigProcessors：

```java
//注册所有和注解有关的后处理器，在给定的registry上
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
      BeanDefinitionRegistry registry, @Nullable Object source) {
   //转化为DefaultListableBeanFactory
   DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);

   if (beanFactory != null) {
      if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
         //解析@Order，用来调整配置类加载顺序
         beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
      }
      if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
         //此类用以决定一个bean是否可以当作一个依赖的候选者
         beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
      }
   }

   Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
   //此类用于处理标注了@Configuration注解的类
   if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   }
   //此类便用于对标注了@Autowire等注解的bean或是方法进行注入
   if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
   if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition();
      try {
         def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
               AnnotationConfigUtils.class.getClassLoader()));
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
      }
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   }
   //提供对于注解@EventListener的支持，此注解在Spring4.2被添加，用于监听ApplicationEvent事件
   if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   }
   //此类应该是和上面的配合使用，用以产生EventListener对象
   if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   }

   return beanDefs;
}
```

这里注册的组件有：

- AnnotationAwareOrderComparator：用来解析 @Order与@Priority注解，用来定义类的加载顺序
- ContextAnnotationAutowireCandidateResolver：用来决定一个bean是否可以当作一个依赖的候选者
- ConfigurationClassPostProcessor：解析@Configuration注解
- AutowiredAnnotationBeanPostProcessor：解析@Autowire等注解
- RequiredAnnotationBeanPostProcessor：解析@Require注解
- PersistenceAnnotationBeanPostProcessor：用于提供对JPA的支持
- EventListenerMethodProcessor:提供对于注解@EventListener的支持，此注解在Spring4.2被添加，用于监听ApplicationEvent事件
- DefaultEventListenerFactory：此类应该是和上面的配合使用，用以产生EventListener对象，也是从Spring4.2加入

### AutowiredAnnotationBeanPostProcessor

该组件的入口有两个，一个是在AbstractAutowireCapableBeanFactory.doCreateBean中，由于AutowiredAnnotationBeanPostProcessor是MergedBeanDefinitionPostProcessor子类，所以会调用一次后处理其中方法来扫描bean对应的calss将包含自动注入的注解的属性和方法进行缓存。第二个入口就是在属性注入的populateBean方法中将缓存的需要自动注入的属性或方法通过反射注入或调用。

**第一个入口：**

找到需要进行注入的属性，并进行标记

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {
    
   ****省略代码*****
       
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         // 第一个入口！！！！
         applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         mbd.postProcessed = true;
      }
   }
    
   ****省略代码*****

}

protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
                //入口方法！！！！！
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
```

接下来是AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition

```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    //在bean对应的class中寻找需要自动注入的属性或方法
   InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
   metadata.checkConfigMembers(beanDefinition);
}
```

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
   // Fall back to class name as cache key, for backwards compatibility with custom callers.
   String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
   // Quick check on the concurrent map first, with minimal locking.
   InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
   if (InjectionMetadata.needsRefresh(metadata, clazz)) {
      synchronized (this.injectionMetadataCache) {
         metadata = this.injectionMetadataCache.get(cacheKey);
         if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            if (metadata != null) {
               metadata.clear(pvs);
            }
             //将需要自动注入的Field和Method进行缓存
            metadata = buildAutowiringMetadata(clazz);
             //加入缓存，当从第二个入口进来时不需要再次寻找
            this.injectionMetadataCache.put(cacheKey, metadata);
         }
      }
   }
   return metadata;
}
```

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
   if (!AnnotationUtils.isCandidateClass(clazz, this.autowiredAnnotationTypes)) {
      return InjectionMetadata.EMPTY;
   }

   List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
   Class<?> targetClass = clazz;
   //循环检测父类
   do {
      final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
	//对field进行解析
      ReflectionUtils.doWithLocalFields(targetClass, field -> {
          //寻找field是否包含自动注入的注解
         MergedAnnotation<?> ann = findAutowiredAnnotation(field);
         if (ann != null) {
            if (Modifier.isStatic(field.getModifiers())) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static fields: " + field);
               }
               return;
            }
            boolean required = determineRequiredStatus(ann);
             //保存
            currElements.add(new AutowiredFieldElement(field, required));
         }
      });
       //对method进行解析
      ReflectionUtils.doWithLocalMethods(targetClass, method -> {
         Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
         if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
            return;
         }
         MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
         if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
            if (Modifier.isStatic(method.getModifiers())) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static methods: " + method);
               }
               return;
            }
            if (method.getParameterCount() == 0) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation should only be used on methods with parameters: " +
                        method);
               }
            }
            boolean required = determineRequiredStatus(ann);
            PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
            currElements.add(new AutowiredMethodElement(method, required, pd));
         }
      });

      elements.addAll(0, currElements);
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);

   return InjectionMetadata.forElements(elements, clazz);
}


private MergedAnnotation<?> findAutowiredAnnotation(AccessibleObject ao) {
		//获取field上的注解
		MergedAnnotations annotations = MergedAnnotations.from(ao, SearchStrategy.INHERITED_ANNOTATIONS);
		//autowiredAnnotationTypes是一个Set，在AutowiredAnnotationBeanPostProcessor实例化时进行了初始化，保存了@Autowire和@Value的class类
		for (Class<? extends Annotation> type : this.autowiredAnnotationTypes) {
			MergedAnnotation<?> annotation = annotations.get(type);
			//有自动注入的标志，返回注解
			if (annotation.isPresent()) {
				return annotation;
			}
		}
		return null;
	}
```

至此，第一个入口的对class中@Autowired注解的扫描已经完成

**第二个入口：**

开始对第一个入口中标记的字段等进行注入。

在populateBean方法对xml声明的属性注入完成后，会执行一个后置处理器：

```java
for (BeanPostProcessor bp : getBeanPostProcessors()) {
   if (bp instanceof InstantiationAwareBeanPostProcessor) {
      InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
      PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
      if (pvsToUse == null) {
         if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
         }
          //第二个入口 ！！！！
         pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
         if (pvsToUse == null) {
            return;
         }
      }
      pvs = pvsToUse;
   }
}

```

AutowiredAnnotationBeanPostProcessor.postProcessProperties

```java
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    //从缓存中直接取出来需要租入的field和method
   InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
   try {
       //开始注入了。。。
      metadata.inject(bean, beanName, pvs);
   }
   catch (BeanCreationException ex) {
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
   }
   return pvs;
}

```

```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    //这里的this.checkedElements，在入口一的metadata.checkConfigMembers(beanDefinition);中进行赋值
   Collection<InjectedElement> checkedElements = this.checkedElements;
   Collection<InjectedElement> elementsToIterate =
         (checkedElements != null ? checkedElements : this.injectedElements);
   if (!elementsToIterate.isEmpty()) {
      for (InjectedElement element : elementsToIterate) {
         if (logger.isTraceEnabled()) {
            logger.trace("Processing injected element of bean '" + beanName + "': " + element);
         }
          //
         element.inject(target, beanName, pvs);
      }
   }
}

```

```Java
protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
      throws Throwable {

   if (this.isField) {
      Field field = (Field) this.member;
      ReflectionUtils.makeAccessible(field);
       //field注入
      field.set(target, getResourceToInject(target, requestingBeanName));
   }
   else {
      if (checkPropertySkipping(pvs)) {
         return;
      }
      try {
         Method method = (Method) this.member;
         ReflectionUtils.makeAccessible(method);
          ///method直接invoke？？？
         method.invoke(target, getResourceToInject(target, requestingBeanName));
      }
      catch (InvocationTargetException ex) {
         throw ex.getTargetException();
      }
   }
}
```

### ConfigurationClassPostProcessor

解析@Configuration注解

ConfigurationClassPostProcessor的入口位于refresh方法的`this.invokeBeanFactoryPostProcessors(beanFactory);`。ConfigurationClassPostProcessor是BeanDefinitionRegistryPostProcessor子类，所以会先执行`postProcessBeanDefinitionRegistry`方法。

ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry:

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
   //对registry获取hash值，防止重复解析
   int registryId = System.identityHashCode(registry);
   if (this.registriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
   }
   if (this.factoriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
   }
   this.registriesPostProcessed.add(registryId);

   //主方法
   processConfigBeanDefinitions(registry);
}
```

执行主方法`processConfigBeanDefinitions(registry);`

该方法对registry中的BeanDefinition进行遍历，当BeanDefinition存在@Configuration或@Component，@ComponentScan，@Import，@ImportResource注解，将BeanDefinition加入到集合中为进一步解析。

然后实例化ConfigurationClassParser用来解析。

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
   //标注有配置类相关注解的BeanDefinitionHolder集合
   List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
   //获取已经注册的bean的name
   String[] candidateNames = registry.getBeanDefinitionNames();

   //遍历beanNames
   for (String beanName : candidateNames) {
      BeanDefinition beanDef = registry.getBeanDefinition(beanName);
      //存在指定属性则意味着已经处理过了,直接跳过
      if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
         if (logger.isDebugEnabled()) {
            logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
         }
      }
      //判断对应bean是否为配置类，然后加入到configCandidates中
      //当存在@Configuration或者@Component等注解时
      else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
         configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
      }
   }

   // Return immediately if no @Configuration classes were found
   if (configCandidates.isEmpty()) {
      return;
   }

   // Sort by previously determined @Order value, if applicable
   //对configCandidates按照@order进行排序
   configCandidates.sort((bd1, bd2) -> {
      int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
      int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
      return Integer.compare(i1, i2);
   });

   // Detect any custom bean name generation strategy supplied through the enclosing application context
   SingletonBeanRegistry sbr = null;
   if (registry instanceof SingletonBeanRegistry) {
      sbr = (SingletonBeanRegistry) registry;
      // 如果localBeanNameGeneratorSet 等于false 并且SingletonBeanRegistry 中有 id 为 org.springframework.context.annotation.internalConfigurationBeanNameGenerator
      // 的bean .则将componentScanBeanNameGenerator,importBeanNameGenerator 赋值为 该bean.
      if (!this.localBeanNameGeneratorSet) {
         BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
               AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
         if (generator != null) {
            this.componentScanBeanNameGenerator = generator;
            this.importBeanNameGenerator = generator;
         }
      }
   }

   if (this.environment == null) {
      this.environment = new StandardEnvironment();
   }

   // Parse each @Configuration class
   // 4. 实例化ConfigurationClassParser 为了解析 各个配置类
   ConfigurationClassParser parser = new ConfigurationClassParser(
         this.metadataReaderFactory, this.problemReporter, this.environment,
         this.resourceLoader, this.componentScanBeanNameGenerator, registry);

   // 实例化2个set,candidates 用于将之前加入的configCandidates 进行去重
   // alreadyParsed 用于判断是否处理过
   Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
   Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
   //进行解析
   do {
      parser.parse(candidates);
      parser.validate();

      Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
      configClasses.removeAll(alreadyParsed);

      // Read the model and create bean definitions based on its content
      if (this.reader == null) {
         this.reader = new ConfigurationClassBeanDefinitionReader(
               registry, this.sourceExtractor, this.resourceLoader, this.environment,
               this.importBeanNameGenerator, parser.getImportRegistry());
      }
      this.reader.loadBeanDefinitions(configClasses);
      alreadyParsed.addAll(configClasses);

      candidates.clear();
      if (registry.getBeanDefinitionCount() > candidateNames.length) {
         String[] newCandidateNames = registry.getBeanDefinitionNames();
         Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
         Set<String> alreadyParsedClasses = new HashSet<>();
         for (ConfigurationClass configurationClass : alreadyParsed) {
            alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
         }
         for (String candidateName : newCandidateNames) {
            if (!oldCandidateNames.contains(candidateName)) {
               BeanDefinition bd = registry.getBeanDefinition(candidateName);
               if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                     !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                  candidates.add(new BeanDefinitionHolder(bd, candidateName));
               }
            }
         }
         candidateNames = newCandidateNames;
      }
   }
   while (!candidates.isEmpty());

   // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
   if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
      sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
   }

   if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
      // Clear cache in externally provided MetadataReaderFactory; this is a no-op
      // for a shared cache since it'll be cleared by the ApplicationContext.
      ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
   }
}
```

ConfigurationClassParser.parse:

根据不同的BeanDefinition类型调用不同的重载方法，目的时通过不同的条件获取MetadataReader

```java
	public void parse(Set<BeanDefinitionHolder> configCandidates) {
		//遍历BeanDefinitionHolder
		for (BeanDefinitionHolder holder : configCandidates) {
			BeanDefinition bd = holder.getBeanDefinition();
			try {

				if (bd instanceof AnnotatedBeanDefinition) {
					parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
				}
				else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
					parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
				}
				else {
					parse(bd.getBeanClassName(), holder.getBeanName());
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
			}
		}

		this.deferredImportSelectorHandler.process();
	}
```

一个parse方法，获取了MetadataReader后再调用processConfigurationClass方法

```java
protected final void parse(@Nullable String className, String beanName) throws IOException {
   Assert.notNull(className, "No bean class name for configuration class bean definition");
   MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
   processConfigurationClass(new ConfigurationClass(reader, beanName));
}

```

processConfigurationClass方法。首先判断该类有没有@Conditional注解，判断是否可以跳过；判断该class是否被解析过。判断完成后进行真正的解析方法。

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
   //处理类上的@Conditional注解，判断是否需要跳过
   if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
      return;
   }
   //判断该class是否被解析过
   ConfigurationClass existingClass = this.configurationClasses.get(configClass);
   if (existingClass != null) {
      if (configClass.isImported()) {
         if (existingClass.isImported()) {
            existingClass.mergeImportedBy(configClass);
         }
         // Otherwise ignore new imported config class; existing non-imported class overrides it.
         return;
      }
      else {
         // Explicit bean definition found, probably replacing an import.
         // Let's remove the old one and go with the new one.
         this.configurationClasses.remove(configClass);
         this.knownSuperclasses.values().removeIf(configClass::equals);
      }
   }

   // Recursively process the configuration class and its superclass hierarchy
   // 搞不懂SourceClass与ConfigurationClass区别。。。
   //一个是最外层？一个是当前？
   SourceClass sourceClass = asSourceClass(configClass);
   do {
      sourceClass = doProcessConfigurationClass(configClass, sourceClass);
   }
   while (sourceClass != null);

   //缓存，表示该类已经被解析过
   this.configurationClasses.put(configClass, configClass);
}


//解析@Conditional***************************
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
		//类上是否包含@Conditional注解
		if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
			return false;
		}

		if (phase == null) {
			if (metadata instanceof AnnotationMetadata &&
					ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
				return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
			}
			return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
		}

		List<Condition> conditions = new ArrayList<>();
		//获取@Conditional注解的value值，即包含matches方法的className
		for (String[] conditionClasses : getConditionClasses(metadata)) {
			for (String conditionClass : conditionClasses) {
				//获取condition对象，并加入集合
				Condition condition = getCondition(conditionClass, this.context.getClassLoader());
				conditions.add(condition);
			}
		}

		AnnotationAwareOrderComparator.sort(conditions);

		for (Condition condition : conditions) {
			ConfigurationPhase requiredPhase = null;
			if (condition instanceof ConfigurationCondition) {
				requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
			}
			//执行matches方法
			if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
				return true;
			}
		}

		return false;
	}
```

对各种注解进行逐个解析。

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
      throws IOException {

   //包含@Component注解
   if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
      // Recursively process any member (nested) classes first
      //处理内部类，如果包含内部类则遍历进行解析，通过processConfigurationClass方法
      processMemberClasses(configClass, sourceClass);
   }

   // Process any @PropertySource annotations
   //处理@PropertySource注解
   for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), PropertySources.class,
         org.springframework.context.annotation.PropertySource.class)) {
      if (this.environment instanceof ConfigurableEnvironment) {
         //处理propertySource，将@PropertySource注解中标注的路径解析成PropertySource对象加载到环境中
         processPropertySource(propertySource);
      }
      else {
         logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
               "]. Reason: Environment must implement ConfigurableEnvironment");
      }
   }

   // Process any @ComponentScan annotations
   //处理@ComponentScan注解
   Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
   if (!componentScans.isEmpty() &&
         !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
      for (AnnotationAttributes componentScan : componentScans) {
         // The config class is annotated with @ComponentScan -> perform the scan immediately
         //进行扫描
         Set<BeanDefinitionHolder> scannedBeanDefinitions =
               this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
         // Check the set of scanned definitions for any further config classes and parse recursively if needed
         //依次遍历扫描的配置类进行解析
         for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
            BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
            if (bdCand == null) {
               bdCand = holder.getBeanDefinition();
            }
            if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
               parse(bdCand.getBeanClassName(), holder.getBeanName());
            }
         }
      }
   }

   // Process any @Import annotations
   //处理@Import
   processImports(configClass, sourceClass, getImports(sourceClass), true);

   // Process any @ImportResource annotations
   //处理@ImportResource注解
   AnnotationAttributes importResource =
         AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
   if (importResource != null) {
      String[] resources = importResource.getStringArray("locations");
      Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
      for (String resource : resources) {
         String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
         configClass.addImportedResource(resolvedResource, readerClass);
      }
   }

   // Process individual @Bean methods
   //处理内置@Bean方法
   Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
   for (MethodMetadata methodMetadata : beanMethods) {
      configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
   }

   // Process default methods on interfaces
   processInterfaces(configClass, sourceClass);

   // Process superclass, if any
   if (sourceClass.getMetadata().hasSuperClass()) {
      String superclass = sourceClass.getMetadata().getSuperClassName();
      if (superclass != null && !superclass.startsWith("java") &&
            !this.knownSuperclasses.containsKey(superclass)) {
         this.knownSuperclasses.put(superclass, configClass);
         // Superclass found, return its annotation metadata and recurse
         return sourceClass.getSuperClass();
      }
   }

   // No superclass -> processing is complete
   return null;
}
```

在@Component注解的类上，可能存在用来配置的内部类。

@PropertySource解析思路就是将注解上指定的property文件注册到Environment对象中。

@ComponentScan注解解析，同component-scan。

@Import解析，获取属性的值，然后作为配置类再解析。

@Bean的解析。其实就是根据方法解析为BeanDefinition，然后将方法注册为工厂方法，如果是静态的就是静态工厂方法，非静态就是实例化工厂方法。

**@Enable原理**

自spring3.0以后开始引入了@Enable注解模式，例如@EnableWebMVC等。其实现原理就是通过@Import注解。

在ConfigurationClassPostProcessor执行过程中当解析到@Import注解，会递归的进行解析以达到通过委托类来完成某一领域相关Bean的集中注册功能。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

