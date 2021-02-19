---
title: Spring-IOC之XML解析
author: XIA
categories:
  - null
tags:
  - null
date: 2020-01-31 13:21:02
---

# XML配置

## `<beans>`

xml配置文件中的最顶层元素，它下面包含0或1个`<description>`以及多个`<bean>`、`<import>`、`<alias>`元素。

`<beans>`支持的属性：

- default-lazy-init。其值可以指定为true或者false，默认值为false。用来标志是否对所
  有的进行延迟初始化。
- default-autowire。可以取值为no、byName、byType、constructor以及autodetect。默
  认值为no，如果使用自动绑定的话，用来标志全体bean使用哪一种默认绑定方式。
- default-dependency-check。可以取值none、objects、simple以及all，默认值为none，
  即不做依赖检查。
- default-init-method。如果所管辖的按照某种规则，都有同样名称的初始化方法的
  话，可以在这里统一指定这个初始化方法名，而不用在每一个上都重复单独指定。
- default-destroy-method。与default-init-method相对应，如果所管辖的bean有按照某种
  规则使用了相同名称的对象销毁方法，可以通过这个属性统一指定。

`<beans>`下的子元素含义：

- `<description>`：配置一些描述信息
- `<import>`: 合并一些spring的配置文件
- `<alias>`：为bean进行起别名
- `<bean>`: 就是定义spring的bean啊

## `<bean>`

**`<bean>的属性`**

- id：定义bean的id，作为bean的唯一标识
- class：指定该bean的class类型
- depends-on: 当我们需要在对象中使用另一个对象时，可以通过构造方法或者变量进行注入，以保证它是存在的并且已经准备完成。而有时我们并不需要在对象声明时候对它进行注入，但是还需要在创建这个对象时检查它是否已经存在，这是就是用depends-on属性了。
- autowire：为bean的依赖提供自动注入的功能。支持以下值：no，不采用自动绑定；byName，使用beanName作为注入的依据；byType，根据类型作为注入的依据；constructor，前两种都是为property进行注入，这个是为构造函数进行注入，匹配方式为byType；autodetect，是byType和constructor的结合体，会为构造函数和property使用byType模式进行注入。还可以再`<beans>`元素中设置default-autowire属性指定该beans下的bean默认自动注入行为。
- lazy-init：主要针对ApplicationContext容器的bean，因为它在启动时候会默认加载所有singleton的bean，设置该属性后会进行懒加载，在使用时候在进行加载bean。
- parent：有时候会存在两个bean间使用相同的属性，而且注入的属性值也相同的情况。为了减少代码书写量，可以使用parent来指定从一个bean中继承相同的属性的注入。
- abstract：作为一个bean的模板，通常和parent属性共同属性，使用的abstract属性后可以不指定class属性。因为使用了abstract属性的bean不会被实例化。
- scope：限定bean的作用域。
- 工厂方法：如果一个bean是通过工厂方法获得的，可以在bean定义时指定工厂方法的属性，工厂方法又分为静态工厂方法和非静态工厂方法之分。配置时要分开配置。
  - 配置静态工厂方法时，指定bean的class属性为静态工厂方法所在的类名，factory-method为工厂方法的名称。如：`<bean id="stu" class="....StudentFactory" factory-method="getStudent"/>`
  - 配置非静态工厂方法时，需要使用工厂方法所在的bean的引用，然后再声明工厂方法。`<bean id="stu" factory-bean="stuFactory" factory-method="getStu">`
  - 当工厂方法存在需要参数的时候，可以使用`<constructor-arg>`来指定方法调用的参数。

**`<bean>的子元素`**

- `<constructor-arg>`：为构造方法的参数进行注入。该元素可以使用如下属性：type，要注入的属性的类型；index，所传入的依赖在构造方法顺序的第几位；
- `<property>`: 为bean中的变量进行注入。该元素可以使用如下属性：name，用来指定将要注入的变量名称。
- `<constructor-arg>`和`<property>`的可配置项：
  - `<value>`: 可以注入一个简单数据类型
  - `<ref>`：用来引用容器中其他对象实例。该元素有三个属性可供选择：local，parent，bean。分别代表被引用的实例来自：与当前配置对象在同一配置文件，只能位于当前容器的父容器，既可以在同一文件也可以在父容器。
  - `<idref>`：如果注入一个所依赖的对象名称，可以使用该元素。它与`<vlaue>`元素的区别在于，它可以检查所指定的所依赖的对象名称对应的对象是否存在。
  - 内部`<bean>`：有时我们所依赖的对象只有当前一个对象引用，或者不想别的对象通过`<ref>`引用他，这是就可以使用内部`<bean>`，内部`<bean>`可以不指定id属性
  - `<list>`：如果注入的对象是java.util.List类型或者是数组类型，使用该元素
  - `<set>`：如果注入的对象是java.util.Set类型则使用该元素
  - `<map>` : 如果注入的对象是java.util.Map类型则使用该元素
  - `<props>`:如果注入的对象是java.util.Properties类型则使用该元素，其key和value只能是String类型
  - `<null/>`:注入一个空对象
- `<lookup-method>`：方法注入，可以为一个方法注入一个bean。配置时name属性表示所要进行注入的方法，bean属性表示要被注入的bean的名称。
- `<replace-method>`：方法替换，可以将bean中的一个方法替换为另一个方法。使用时需要一个类实现MethodReplace接口，然后在将要被替换的方法所在的bean定义中添加replace-method元素，name属性表示将要被替换的方法，replacer属性表示实现了MethodReplace接口的类的bean的名称。

# XML文件解析

在实际开发中很少会使用BeanFactory，而是直接使用ApplicationContext。ApplicationContext作为更高级的容器但是其ioc的核心功能依然是委托给BeanFactory来实现。所以这里使用BeanFactory的实现类XmlBeanFactory入手来看xml文件在spring中的解析。

## XmlBeanFactory使用实例

XmlBeanFactory是BeanFactory接口的一种最终的实现，如下是使用XmlBeanFactory的demo：

```java
/**
定义一个pojo
*/
@Data //lombok的注解
public class Student {
    private String name;
    private int age;
}
```

```xml
/**
spring的配置文件
*/
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="student" class="demo.Student">
        <property name="name" value="xbx"/>
        <property name="age" value="23"/>
    </bean>
</beans>
```

```java
       
/**
XmlBeanFactory的使用
*/
public class XBFTest {
    public static void main(String[] args) {
        XmlBeanFactory xmlBeanFactory = new XmlBeanFactory(new ClassPathResource("config/applicationContext.xml"));
        Object student = xmlBeanFactory.getBean("student");
        System.out.println(student);
    }
}
```

从上面这段代码来看，spring所做的事情有三部分：1. 读取配置文件；2. 根据配置文件的的配置信息找到相应的类，然后实例化它；3. 获取实例化后的实例。

## XmlBeanFactory结构

![XmlBeanFactory](https://xbxblog.bj.bcebos.com/XmlBeanFactory.png)

根据上图可以看到，XmlBeanFactory直接继承自DefaultListableBeanFactory。DefaultListableBeanFactory是BeanFactory接口的一个比较通用的实现类，DefaultListableBeanFactory除了间接实现了BeanFactory接口，还间接实现了BeanDefinitionRegistry接口。BeanFactory接口负责Bean容器的访问操作，BeanDefinitionRegistry接口负责容器内Bean的注册管理操作。BeanDefinitionRegistry抽象出bean的注册逻辑，而BeanFactory则抽象出了bean的管理逻辑

XmlBeanFactory中的关于bean的操作都是直接继承自DefaultListableBeanFactory类，区别在XmlBeanFactory增加了一个XmlBeanDefinitionReader类型的reader属性，用于对资源文件进行读取和注册。

如果将BeanFactory比作一座图书馆，BeanDefinitionRegistry就是一个书架，而书架上的书就是BeanDefinition。在spring中BeanDefinition负责保存一个Bean的必要信息，如class类型、是否是抽象类、构造方法参数等信息。它的主要实现类RootBeanDefinition、ChildBeanDefinition、GenericBeanDefinition。

> 各种BeanDefinition关系详解：https://blog.csdn.net/f641385712/article/details/88683596

**一些类的主要作用：**

+ AliasRegistry: 定义对alias的简单增删改等操作
+ SimpleAiasRegistry: 主要使用map作为alias的缓存,并对接口AliasRegistry进行实现
+ SingletonBeanRegistry:定义对单例的注册及获取
+ BeanFactory：定义获取bean及bean的各种属性
+ DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry各函数的实现
+ HierarchicalBeanFactory ：继承 BeanFactory，也就是在BeanFactory定义的功能的基础上增加了对 parentFactory的支持
+ BeanDefinitionRegistry：定义对 BeanDefinition 的各种增删改操作
+ FactoryBeanRegstrySupport ：在 DefaultSingletonBeanRegist 基础上增加了对 FactoryBean的特殊处理功能
+ ConfigurableBeanFactory ：提供配置Factory的各种方法
+ ListableBeanFactory ：根据各种条件获取bean的配直清单
+ AbstractBeanFactory ：综合 FactoryBeanRegistrySupport和ConfigurableBeanFactory功能
+ AutowireCapableBeanFactory ：提供创建bean、自动注入、初始化以及应用 bean 的后处理器
+ AbstractAutowireCapabeBeanFactory ：综合AbstractBeanFactory并对接口AutowireCapableBeanFactory进行实现
+ ConfigurableListableBeanFactory : Beanfactory配置清单，指定忽略类型及接口等
+ DefaultListableBeanFactory ：综合上面所有功能，主要是对 bean 注册后的处理

从以上类结构来看，spring将每个类的职责进行了分割，最后通过一个类来进行综合这些功能并对外提供接口供使用。

## XmlBeanDefinitionReader结构

XmlBeanFactory将xml文件解析为BeanDefinition的工作委托给XmlBeanDefinitionReader处理。所以接下来的很多工作都是在XmlBeanDefinitionReader中。

````java
public class XmlBeanFactory extends DefaultListableBeanFactory {

	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
        //解析与注册BeanDefinition
		this.reader.loadBeanDefinitions(resource);
	}

}
````

![image-20200131145137111](https://xbxblog2.bj.bcebos.com/spring-ioc%E4%B9%8Bxml%E8%A7%A3%E6%9E%90%2Fimage-20200131145137111.png)

**主要类与字段：**

+ EnvironmentCapable：表示包含并公开 Environment 引用的组件的接口。

  ```java
  public interface EnvironmentCapable {
  
  	/**
  	 * Return the {@link Environment} associated with this component.
  	 */
  	Environment getEnvironment();
  
  }
  ```

+ BeanDefinitionReader：用于bean定义读取的接口。根据接口的内容可以看出来其主要方法为loadBeanDefinitions，用于将指定的Resources或location定义的BeanDefinition读取出来。

  ```java
  public interface BeanDefinitionReader {
  
  	BeanDefinitionRegistry getRegistry();
  
  	@Nullable
  	ResourceLoader getResourceLoader();
      
  	@Nullable
  	ClassLoader getBeanClassLoader();
  
  	BeanNameGenerator getBeanNameGenerator();
  
  	int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
  
  	int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;
  
  	int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;
  
  	int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;
  
  }
  ```

## 资源加载过程

spring通过Resource和ResourceLoader接口对资源进行了重新的封装。Resource接口主要提供资源的访问和操作的功能，ResourceLoader提供对资源的定位功能。

从XmlBeanFactory构造函数开始：

```java
	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}
//都会调用这个构造函数
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource); //对resource进行解析
	}
```

可以看到XmlBeanDefinitionReader类型的reader属性调用loadBeanDefinitions方法开始对resource进行解析。

![1568591560605](https://xbxblog.bj.bcebos.com/loadBeanDefnition%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

1.1-1.2：根据resource获取EncodedResource类型实例，该实例封装了Resource对象的编码等信息，spring需要根据这些信息作为输入流的编码。

获取了EncodedResource后调用XmlBeanDefinitionReader内部的`loadBeanDefinitions(new EncodedResource(resource));`方法，这个方法才是真正的数据准备阶段。

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}
		//记录已经加载的资源
    	//重复加载的验证，如果发现已经对当前的资源加载过了，则抛出异常。
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			//从封装的encodedResource对象中后去Resource对象获取其中的InputStream
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				//根据InputStream获取InputSource对象，用来构造之后的Document对象
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				//进入逻辑的核心部分
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

在上面代码中调用了`doLoadBeanDefinitions(inputSource, encodedResource.getResource())`方法，这个方法就是加载和解析xml文件的核心

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
            //加载xml文件，获取document对象
			Document doc = doLoadDocument(inputSource, resource);
            //根据document对其中注册的Bean信息进行解析
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch........
	}
```

这里做了来两件事：

1. 加载xml文件，获取Document对象，这部分工作委托给DefaultDocumentLoader类进行；
2. 根据获取的Document对象，对Bean的信息进行解析，这部分工作委托给DefaultBeanDefinitionDocumentReader类进行。

### 加载xml文件，获取Document对象

加载xml文件时，进一步调用了doLoadDocument方法。在doLoadDocument方法中spring并没有亲自对xml文件进行加载，而是交给了documentLoader对象进行加载，documentLoader默认实现为DefaultDocumentLoader类型。

```java
	protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
        //交给documentLoader进行加载
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}
```

在documentLoader的loadDocument方法中有四个参数，分别表示：xml文件的inputSource；getEntityResolver()返回一个EntityResolver对象，该对象的作用是在项目中查找dtd或xsd文件；异常辅助类；xml文件的验证模式，dtd或xsd；解析器是否应该支持XML命名空间

接下来就是获取Document对象了,获取的步骤都是老一套，先创建DocumentBuilderFactory，再通过DocumentBuilderFactory创建DocumentBuilder，进而解析inputSource来返回Document对象。

### 解析及注册BeanDefinition

在获取了document对象之后，接下来就是对docmuent对象的解析了。

在doLoadBeanDefinitions方法中`int count = registerBeanDefinitions(doc, resource);`，这句代码是解析document的开始。

```java
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		//真正对doc进行解析的类，默认实现为DefaultBeanDefinitionDocumentReader类型
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
        //开始负责解析的主类DefaultBeanDefinitionDocumentReader的表演，createReaderContext(resource)获取XmlReaderContext，它是xml文档解析期间的上下文，贯穿整个解析过程，封装所有相关的配置以及状态
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        //新增的beanDefinition数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

接下来主要进入**DefaultBeanDefinitionDocumentReader**类了。

DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法：

```java
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext; 
        //获取doc的根节点，也就是<beans></beans>
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
```

进一步调用doRegisterBeanDefinitions方法，获取profile属性，这里可以看出来profile属性的原理~

执行真正的解析动作

```java
protected void doRegisterBeanDefinitions(Element root) {
    	//BeanDefinitionParserDelegate是负责解析<bean></bean>标签的委托类
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
            //获取profile属性
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
                
                //这里Environment是在XmlBeanDefinitionReader中获取的，还记得吗，它实现了EnvironmentCapable接口
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		//设计模式中的模板方法，没有实现，由子类继承后覆盖实现
		preProcessXml(root);
    	//开始真正的解析了
		parseBeanDefinitions(root, this.delegate);
    	//设计模式中的模板方法，没有实现，由子类继承后覆盖实现
		postProcessXml(root);

		this.delegate = parent;
	}
```

遍历根节点的子节点们，根据是否为自定义类型执行不同的解析方法。

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
                    //默认命名空间的元素
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
                        //自定义命名空间的元素
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```

#### 解析默认命名空间的元素

默认命名空间的元素解析通过parseDefaultElement方法作为入口，根据阶段元素标签的类型对：import、alias、bean、beans标签分别执行不同的解析逻辑。

```java
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            //只有这里用了delegate
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse 又执行了doRegisterBeanDefinitions方法，ele作为root重新来过
			doRegisterBeanDefinitions(ele);
		}
	}
```

##### 对`<bean>`元素进行解析

对bean标签执行解析的processBeanDefinition的方法,将解析的BeanDefinition注册到registry中。

```java
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //该方法返回的BeanDefinitionHolder对象包含BeanDefinition、beanName、alias属性，该方法的详细解析过程在下面
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
            //解析默认标签中的自定义标签元素，这里其实只是自定义属性
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

**BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele)**

delegate.parseBeanDefinitionElement(ele)进一步调用`parseBeanDefinitionElement(ele, null)`方法

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		//获取class属性
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		//获取parent属性
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
			//创建GenericBeanDefinition对象，它是AbstractBeanDefinition子类.自spring2.5推荐使用一站式的GenericBeanDefinition。
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			//解析各种bean属性，将其保存到bd中
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

			parseMetaElements(ele, bd);
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
			//解析构造参数
			parseConstructorArgElements(ele, bd);
			//解析property元素
            parsePropertyElements(ele, bd);
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch .......
            
		return null;
	}
```

在上面代码中，生成的AbstractBeanDefinition类型对象，将`<bean></bean>`元素所可能包含的属性都以硬编码的方式作为类的属性。

解析构造参数：parseConstructorArgElements(ele, bd);

```java
	public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
		String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
		String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		//该constructor-arg存在index属性
		if (StringUtils.hasLength(indexAttr)) {
			try {
				int index = Integer.parseInt(indexAttr);
				if (index < 0) {
					error("'index' cannot be lower than 0", ele);
				}
				else {
					try {
						this.parseState.push(new ConstructorArgumentEntry(index));
						//解析该构造参数的值，value
						Object value = parsePropertyValue(ele, bd, null);
						//将获取的value保存到 ConstructorArgumentValues.ValueHolder中
						ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
						if (StringUtils.hasLength(typeAttr)) {
							valueHolder.setType(typeAttr);
						}
						if (StringUtils.hasLength(nameAttr)) {
							valueHolder.setName(nameAttr);
						}
						valueHolder.setSource(extractSource(ele));
						//bd中通过一个LinkedHashMap维护构造函数的参数，index为key，valueHolder为值
						if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
							error("Ambiguous constructor-arg entries for index " + index, ele);
						}
						else {
							bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
						}
					}
					finally {
						this.parseState.pop();
					}
				}
			}
			catch (NumberFormatException ex) {
				error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
			}
		}
		else { //不存在index
			try {
				this.parseState.push(new ConstructorArgumentEntry());
				Object value = parsePropertyValue(ele, bd, null);
				ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
				if (StringUtils.hasLength(typeAttr)) {
					valueHolder.setType(typeAttr);
				}
				if (StringUtils.hasLength(nameAttr)) {
					valueHolder.setName(nameAttr);
				}
				valueHolder.setSource(extractSource(ele));
				//不存在index的构造函数，使用ArrayList对值进行保存
				bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
			}
			finally {
				this.parseState.pop();
			}
		}
	}
```

解析构造参数中的值：parsePropertyValue(ele, bd, null);

```java
public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
		String elementName = (propertyName != null ?
				"<property> element for property '" + propertyName + "'" :
				"<constructor-arg> element");

		// Should only have one child element: ref, value, list, etc.
		NodeList nl = ele.getChildNodes();
		Element subElement = null;
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
					!nodeNameEquals(node, META_ELEMENT)) {
				// Child element is what we're looking for.
				if (subElement != null) {
					error(elementName + " must not contain more than one sub-element", ele);
				}
				else {
					subElement = (Element) node;
				}
			}
		}

		boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
		boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
    	//ref和value标签只能存在一个
		if ((hasRefAttribute && hasValueAttribute) ||
				((hasRefAttribute || hasValueAttribute) && subElement != null)) {
			error(elementName +
					" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
		}

		if (hasRefAttribute) {
			String refName = ele.getAttribute(REF_ATTRIBUTE);
			if (!StringUtils.hasText(refName)) {
				error(elementName + " contains empty 'ref' attribute", ele);
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName);
			ref.setSource(extractSource(ele));
			return ref;
		}
		else if (hasValueAttribute) {
			TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
			valueHolder.setSource(extractSource(ele));
			return valueHolder;
		}
		else if (subElement != null) {
			//当不是value也不是ref时，调用这个方法继续解析。后面代码太长了，不贴了。
			return parsePropertySubElement(subElement, bd);
		}
		else {
			// Neither child element nor "ref" or "value" attribute found.
			error(elementName + " must specify a ref or value", ele);
			return null;
		}
	}
```

对property元素的解析：parsePropertyElements(ele, bd);

与解析构造函数的原理类似，依次遍历property元素，通过parsePropertyValue方法解析内容。然后将其name和解析后的结果封装为PropertyValue。BeanDefinition中通过一个list维护这个bean中的property。

**bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);**

```java
	public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef) {
		return decorateBeanDefinitionIfRequired(ele, originalDef, null);
	}
```

解析非默认元素时将属性和节点分开处理：

```java
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
			Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

		BeanDefinitionHolder finalDefinition = originalDef;

		// Decorate based on custom attributes first.
		//装饰非默认属性
		NamedNodeMap attributes = ele.getAttributes();
		for (int i = 0; i < attributes.getLength(); i++) {
			Node node = attributes.item(i);
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}

		// Decorate based on custom nested elements.
		//装饰非默认节点
		NodeList children = ele.getChildNodes();
		for (int i = 0; i < children.getLength(); i++) {
			Node node = children.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
			}
		}
		return finalDefinition;
	}
```

```java
public BeanDefinitionHolder decorateIfRequired(
			Node node, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {
		//获取名字空间的uri
		String namespaceUri = getNamespaceURI(node);
		//不是默认的
		if (namespaceUri != null && !isDefaultNamespace(namespaceUri)) {
			//根据命名空间找到处理器
			NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
			if (handler != null) {
                //进行装饰
				BeanDefinitionHolder decorated =
						handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
				if (decorated != null) {
					return decorated;
				}
			}
			else if (namespaceUri.startsWith("http://www.springframework.org/schema/")) {
				error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
			}
			else {
				// A custom namespace, not to be handled by Spring - maybe "xml:...".
				if (logger.isDebugEnabled()) {
					logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
				}
			}
		}
		return originalDef;
	}
```

##### 注册BeanDefinition

这里的register就是XmlBeanFactory。

```java
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {
        
        //通过beanName注册BeanDefinition
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
        
        //通过alias注册BeanDefinition
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

这两种注册方式原理上大同小异，都是将beanDefinition保存到ConcurrentHashMap中。不同的是通过beanName方式进行注册的保存的kv为beanName:BeanDefinition，而通过alias方式注册保存的kv为alias:beanName。

##### 对`<alias>`标签的解析

在parseDefaultElement方法中调用processAliasRegistration(Element ele)方法进行对alias标签的解析，其实他的解析最终调用的就是，在对BeanDefinition进行进行alias注册时调用的registry.registerAlias(beanName, alias)方法。

````java
	protected void processAliasRegistration(Element ele) {
		String name = ele.getAttribute(NAME_ATTRIBUTE);
		String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
		boolean valid = true;
		if (!StringUtils.hasText(name)) {
			getReaderContext().error("Name must not be empty", ele);
			valid = false;
		}
		if (!StringUtils.hasText(alias)) {
			getReaderContext().error("Alias must not be empty", ele);
			valid = false;
		}
		if (valid) {
			try {
                //看这里。。。似曾相似。。。所以。。。就这样了
				getReaderContext().getRegistry().registerAlias(name, alias);
			}
			catch (Exception ex) {
				getReaderContext().error("Failed to register alias '" + alias +
						"' for bean with name '" + name + "'", ele, ex);
			}
			getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
		}
	}
````

##### 对`<import>`标签的解析

在parseDefaultElement方法中调用importBeanDefinitionResource(ele)方法进行进一步解析，这段代码太长了，就不贴了，其实思路就是获取到被导入的xml文件的地址，然后通过`getReaderContext().getReader().loadBeanDefinitions(location, actualResources);`对被导入的文件进行解析。

#### 自定义标签的解析

使用自定义标签的步骤：

- 定义一个需要扩展的组件，定义一个xsd文件描述组件内容
- 创建一个实现BeanDefinnitionParser接口的类，用来解析xsd文件中的定义和组件的定义
- 创建一个继承自NamespaceHandlerSupport类的类，用来将自定义的名字空间和自定义的解析器进行关联，注册到spring容器
- 编写Spring.handlers和Spring.schemas。一个用来根据名字空间的url定位Handler类，一个根据名字空间url确定xsd文件的位置

解析过程：

```java
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        //根据名字空间获取uri
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
        //获取自定义标签对应的handler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
        //执行解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```

**NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);**

在readContext初始化时就将属性初始化为DefaultNamespaceHandlerResolver。resolve()方法的作用就是根据spring.handlers文件，将文件中的类名通过反射转换成class对象，然后**执行自定义handler的init方法**进行初始化，最后将自定义的handler对象返回。

**handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));**

这个方法是为了根据localName获取解析器。解析器的注册是在NamespaceHandler 的init方法中。

这里handler调用的parse方法其实是NamespaceHandlerSupport类的

```java
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}
```

```java
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
        //获取element元素的名称，如<myname:name>,的名称就是name
		String localName = parserContext.getDelegate().getLocalName(element);
        //根据名称找到对应的解析器，也就是在继承自NamespaceHandlerSupport类的类的init方法中定义的
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}
```

然后就是调用解析器的parse方法了。

# 小结

至此已经完成了xml文件的加载和解析过程。我们从XmlBeanFactory类开始解析过程主要涉及XmlBeanDefinitionReader、DefaultDocumentLoader、DefaultBeanDefinitionDocumentReader、BeanDefinitionParserDelegate这几个类。

再XmlBeanFactory中包含一个XmlBeanDefinitionReader对象，负责xml文件的加载。XmlBeanDefinitionReader又将把xml文件包装成Document对象的任务交给了DefaultDocumentLoader类，将document的解析交给DefaultBeanDefinitionDocumentReader类，该类又对spring默认命名空间的元素和自定义命名空间的元素进行区分，交给BeanDefinitionParserDelegate进行解析。最后解析结束将解析得到的BeanDefintion注册到Register中，注册时分为name和alias类型进行注册。而那个Register就是XmlBeabFactory。。。

虽然现在都是用注解驱动，但是了解一下没坏处。

