---
title: spring-core源码阅读笔记
author: XIA
categories:
  - null
tags:
  - null
date: 2019-09-15 20:03:23
---

# Spring IOC 相关知识点回顾

## XML配置

### `<beans>`

xml配置文件中的最顶层元素，它下面包含0或1个<description>以及多个<bean>、<import>、<alias>元素。

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

- <description>：配置一些描述信息
- <import>: 合并一些spring的配置文件
- <alias>：为bean进行起别名
- <bean>: 就是定义spring的bean啊

### `<bean>`

**`<bean>的属性`**

- id：定义bean的id，作为bean的唯一标识
- class：指定该bean的class类型
- depends-on: 当我们需要在对象中使用另一个对象时，可以通过构造方法或者变量进行注入，以保证它是存在的并且已经准备完成。而有时我们并不需要在对象声明时候对它进行注入，但是还需要在创建这个对象时检查它是否已经存在，这是就是用depends-on属性了。
- autowire：为bean的依赖提供自动注入的功能。支持以下值：no，不采用自动绑定；byName，使用beanName作为注入的依据；byType，根据类型作为注入的依据；constructor，前两种都是为property进行注入，这个是为构造函数进行注入，匹配方式为byType；autodetect，是byType和constructor的结合体，会为构造函数和property使用byType模式进行注入。还可以再<beans>元素中设置default-autowire属性指定该beans下的bean默认自动注入行为。
- lazy-init：主要针对ApplicationContext容器的bean，因为它在启动时候会默认加载所有singleton的bean，设置该属性后会进行懒加载，在使用时候在进行加载bean。
- parent：有时候会存在两个bean间使用相同的属性，而且注入的属性值也相同的情况。为了减少代码书写量，可以使用parent来指定从一个bean中继承相同的属性的注入。
- abstract：作为一个bean的模板，通常和parent属性共同属性，使用的abstract属性后可以不指定class属性。因为使用了abstract属性的bean不会被实例化。
- scope：限定bean的作用域。
- 工厂方法：如果一个bean是通过 工厂方法获得的，可以在bean定义时指定工厂方法的属性，工厂方法又分为静态工厂方法和非静态工厂方法之分。配置时要分开配置。
  - 配置静态工厂方法时，指定bean的class属性为静态工厂方法所在的类名，factory-method为工厂方法的名称。如：`<bean id="stu" class="....StudentFactory" factory-method="getStudent"/>`
  - 配置非静态工厂方法时，需要使用工厂方法所在的bean的引用，然后再声明工厂方法。`<bean id="stu" factory-bean="stuFactory" factory-method="getStu">`
  - 当工厂方法存在需要参数的时候，可以使用<constructor-arg>来指定方法调用的参数。

**`<bean>的子元素`**

- <constructor-arg>：为构造方法的参数进行注入。该元素可以使用如下属性：type，要注入的属性的类型；index，所传入的依赖在构造方法顺序的第几位；
- <property>: 为bean中的变量进行注入。该元素可以使用如下属性：name，用来指定将要注入的变量名称。
- <constructor-arg>和<property>的可配置项：
  - <value>: 可以注入一个简单数据类型
  - <ref>：用来引用容器中其他对象实例。该元素有三个属性可供选择：local，parent，bean。分别代表被引用的实例来自：与当前配置对象在同一配置文件，只能位于当前容器的父容器，既可以在同一文件也可以在父容器。
  - <idref>：如果注入一个所依赖的对象名称，可以使用该元素。它与<vlaue>元素的区别在于，它可以检查所指定的所依赖的对象名称对应的对象是否存在。
  - 内部<bean>：有时我们所依赖的对象只有当前一个对象引用，或者不想别的对象通过<ref>引用他，这是就可以使用内部<bean>，内部<bean>可以不指定id属性
  - <list>：如果注入的对象是java.util.List类型或者是数组类型，使用该元素
  - <set>：如果注入的对象是java.util.Set类型则使用该元素
  - `<map>` : 如果注入的对象是java.util.Map类型则使用该元素
  - <props>:如果注入的对象是java.util.Properties类型则使用该元素，其key和value只能是String类型
  - <null/>:注入一个空对象
- <lookup-method>：方法注入，可以为一个方法注入一个bean。配置时name属性表示所要进行注入的方法，bean属性表示要被注入的bean的名称。
- <replace-method>：方法替换，可以将bean中的一个方法替换为另一个方法。使用时需要一个类实现MethodReplace接口，然后在将要被替换的方法所在的bean定义中添加replace-method元素，name属性表示将要被替换的方法，replacer属性表示实现了MethodReplace接口的类的bean的名称。

## Spring IOC 执行流程

Spring IOC启动流程总的可以分为两大步骤，容器启动阶段和容器实例化阶段。

![1569061930193](C:\Users\xia\AppData\Roaming\Typora\typora-user-images\1569061930193.png)

容器的加载阶段主要是加载配置的xml文件到内存中，对xml文件的结构与定义信息进行分析，将xml中定义的信息注册到一个BeanDefinition对象中。至于上图中的其他后处理主要指的是BeanFactoryPostProcessor接口的功能，它在bean实例化之前执行，可以用来进行修改已经定义BeanDefinition信息等操作。我认为这个处理应该放在Bean实例化阶段，因为只有调用了getBean()方法才会对这个方法进行调用。。。

而Bean实例化阶段主要是指IOC容器对象调用了getBean方法后执行的对Bean的实例化过程。

### BeanFactoryPostProcessor接口

BeanFactoryPostProcessor是spring的一种容器扩展机制。它允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做相应的修改。这就相当于在容器实现的第一阶段最后加入一道工序，让我们对最终的BeanDefinition做一些额外的操作，比如修改其中bean定义的某些属性，为bean定义增加其他信息等。

如果要自定义实现BeanFactoryPostProcessor，通常我们需要实现org.springframework.
beans.factory.config.BeanFactoryPostProcessor接口。同时，因为一个容器可能拥有多个BeanFactoryPostProcessor，这个时候可能需要实现类同时实现Spring的org.springframework.core.
Ordered接口，以保证各个BeanFactoryPostProcessor可以按照预先设定的顺序执行。

Spring已经提供了几个现成的BeanFactoryPostProcessor实现类，所以，大多时候，我们很少自己去实现某个BeanFactoryPostProcessor。其中org.springframework.beans.factory.config.PropertyPlaceholderConfigurer和org.springframework.beans.factory.config.Property OverrideConfigurer是两个比较常用的BeanFactoryPostProcessor。另外，为处理配置文件中的数据类型与真正的业务对象所定义的数据类型转换，Spring还允许我们通过org.springframework.beans.factory.config.CustomEditorConfigurer来注册自定义的PropertyEditor以补助容器中默认的PropertyEditor。

BeanFactory使用BeanFactoryPostProcessor需要手动注册，而ApplicationContext会自动识别配置文件中的BeanFactoryPostProcessor并应用他。

#### PropertyPlaceholderConfigurer

通常我们不想将系统相关的信息同业务相关的配置信息混在到XML配置文件中，这样可能会出现更改xml文件时出现问题。我们可以把系统相关的信息如：数据库连接信息等。配置到一个单独的properties文件中，这样系统信息发生变动时只需要更改properties文件就行了。

使用方法很简单，只需要再配置xml文件时将需要从properties文件中读取的数据项使用占位符`${placeHolder}`占位即可。然后将PropertyPlaceholderConfigurer注册到IOC容器中，就可以实现对XML配置文件中的占位符进行替换的功能。

#### PropertyOverrideConfigurer

该类的作用是对xml中定义的bean的属性值进行覆盖。比如在xml中定义的一个bean的属性值a为100，通过该类可以将该属性值更改为200。原理也是通过创建一个properties文件，在文件中以beanName.propertyName形式指定将要进行替换的bean的属性。

使用时将PropertyOverrideConfigurer指定properties文件路径，然后注册到IOC容器中。

#### CustomEditorConfigurer

当我们在xml配置了一个属性的值的时候，他在xml中只是一个String类型。就像在pojo类中指定一个变量的类型为FIle类，而在xml文件为其注入时只需要value属性填写文件路径在pojo类中就可以自动转化为File类型。那这是怎么实现的呢？PropertyEditor就提供了这种转化功能，通过实现PropertyEditor接口（通常都是继承PropertyEditorSupport）来实现转化的逻辑，然后将PropertyEditor接口的实现类注册到CustomEditorConfigurer这个后处理器中。然后再将这个后处理器注册到IOC容器中去。

### Bean的生命周期

在BeanFactory中默认是使用延迟加载的，只有ioc容器对象显式调用getBean方法或者对象间依赖进行的隐式调用时bean才开始进行创建。如下图就是bean的生命周期：

![1569079638040](C:\Users\xia\AppData\Roaming\Typora\typora-user-images\1569079638040.png)

#### Bean的实例化与BeanWrapper

>  容器在内部实现的时候，采用“策略模式”来决定采用何种方式初始化bean实例。通常，可以通过反射或者CGLIB动态字节码生成来初始化相应的bean实例或者动态生成其子类。
>
> org.springframework.beans.factory.support.InstantiationStrategy定义是实例化策略的抽象接口，其直接子类SimpleInstantiationStrategy实现了简单的对象实例化功能，可以通过反射来实例化对象实例，但不支持方法注入方式的对象实例化。CglibSubclassingInstantiationStrategy继承了SimpleInstantiationStrategy的以反射方式实例化对象的功能，并且通过CGLIB的动态字节码生成功能，该策略实现类可以动态生成某个类的子类，进而满足了方法注入所需的对象实例化需求。默认情况下，容器内部采用的是CglibSubclassingInstantiationStrategy。
>
> 容器只要根据相应bean定义的BeanDefintion取得实例化信息，结合CglibSubclassingInstantiationStrategy以及不同的bean定义类型，就可以返回实例化完成的对象实例。但是，返回方式上有些“点缀”。不是直接返回构造完成的对象实例进行包裹，返回相应的BeanWrapper实例。

BeanWrapper接口通常在Spring框架内部使用，它有一个BeanWrapperImpl的实现类，其作用就是对某个bean进行包裹，然后对这个包裹的bean进行操作，比如设置或者获取bean的相关属性值。在Bean的实例化过程中返回的不是原先的对象实例而是一个BeanWrapper对象，就是为了方便对其进行对象属性的设置。

> BeanWrapper定义继承了org.springframework.beans.PropertyAccessor接口，可以以统一的方式对对象属性进行访问；
>
> BeanWrapper定义同时又直接或者间接继承了PropertyEditorRegistry和TypeConverter接口。
>
> 不知你是否还记得CustomEditorConfigurer？当把各种PropertyEditor注册给容器时，知道后面谁用到这些PropertyEditor吗？对，就是BeanWrapper！
>
> 在第一步构造完成对象之后，Spring会根据对象实例构造一个BeanWrapperImpl实例，然后将之前CustomEditorConfigurer注册的PropertyEditor复制一份给BeanWrapperImpl实例（这就是BeanWrapper同时又是PropertyEditorRegistry的原因）。这样，当BeanWrapper转换类型、设置对象属性值时，就不
> 会无从下手了。

#### 各种Aware接口

在完成了对象实例化和相关属性以及依赖设置完成之后，spring容器就会检查当前对象是否实现了一系列名称以Aware结尾的接口。如果实现了这些接口，那么spring就会将Aware接口中规定的依赖注入到当前对象实例。

对BeanFactory而言有以下几个Aware接口：

+ BeanNameWare：将bean对应的beanName注入到当前对象实例
+ BeanClassLoaderAware：将加载该bean的ClassLoader注入当前对象
+ BeanFactoryAware：将BeanFactory容器注入到当前对象中

在ApplicationContext中还存在一些Aware接口，不过这些接口的注入是在BeanPostProcessor中进行的。

+ ResourceLoaderAware：ApplicationContext 实现了Spring的ResourceLoader接口。当容器检测到当前对象实例实现了ResourceLoaderAware接口之后，会将当前ApplicationContext自身设置到对象实例，这样当前对象实例就拥有了其所在ApplicationContext容器的一个引用。
+ ApplicationEventPublisherAware。ApplicationContext作为一个容器，同时还实现了ApplicationEventPublisher接口，这样，它就可以作为ApplicationEventPublisher来使用。所以，当前ApplicationContext容器如果检测到当前实例化的对象实例声明了ApplicationEventPublisherAware接口，则会将自身注入当前对象。
+ MessageSourceAware。ApplicationContext通过MessageSource接口提供国际化的信息支持，即I18n。它自身就实现了MessageSource接口，所以当检测到当前对象实例实现了MessageSourceAware接口，则会将自身注入当前对象实例。
+ ApplicationContextAware。 如果ApplicationContext容器检测到当前对象实现了ApplicationContextAware接口，则会将自身注入当前对象实例。

# BeanFactory与ApplicationContext

spring提供了两种容器类型：BeanFactory与ApplicationContext

* BeanFactory:基础类型的ioc容器，提供完整的ioc服务。如果没有特殊指定默认使用懒加载模式，只有当客户端访问某个受管的对象时才会对该对象进行初始化和依赖注入的操作。所以，相对来说BeanFactory的启动速度较快，所需要的资源有限。

* ApplicationContext：ApplicationContext是在BeanFactory的基础上构建的，是相对高级的容器实现。除了Beanfactory提供的特性之外还提供了事件发布、国际化支持等。ApplicationContext在启动时默认就会对所管理的对象进行初始化和依赖注入。所以ApplicationContext比BeanFactory消耗更多的资源。

* 如图表示这两者间的关系：

  ![beanfactory和applicationContext](C:\Users\xia\Desktop\beanfactory和applicationContext.png)

# 从BeanFactory开始吧

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

从上面这段代码来看，猜测spring所做的事情有三部分：1. 读取配置文件；2. 根据配置文件的的配置信息找到相应的类，然后实例化它；3. 获取实例化后的实例。

其实spring中的实现的思路也是这个样子的。。。接下来我就要开始看看spring是怎样优雅的实现这个过程的。加油(ง •_•)ง

## XmlBeanFactory继承结构与主要类的作用

![XmlBeanFactory](C:\Users\xia\Desktop\XmlBeanFactory.png)

根据上图可以看到，XmlBeanFactory直接继承自DefaultListableBeanFactory。DefaultListableBeanFactory是BeanFactory接口的一个比较通用的实现类，DefaultListableBeanFactory除了间接实现了BeanFactory接口，还间接实现了BeanDefinitionRegistry接口。BeanFactory接口负责Bean容器的访问操作，BeanDefinitionRegistry接口负责容器内Bean的注册管理操作。

XmlBeanFactory中的关于bean的操作都是直接继承自DefaultListableBeanFactory类，区别在XmlBeanFactory增加了一个XmlBeanDefinitionReader类型的reader属性，用于对资源文件进行读取和注册。

如果将BeanFactory比作一座图书馆，BeanDefinitionRegistry就是一个书架，而书架上的书就是BeanDefinition。在spring中BeanDefinition负责保存一个Bean的必要信息，如class类型、是否是抽象类、构造方法参数等信息。它有两个主要实现类RootBeanDefinition和ChildBeanDefinition。

**一些类的主要作用：**

AliasRegistry: 定义对alias的简单增删改等操作
SimpleAiasRegistry: 主要使用map作为alias的缓存,并对接口AliasRegistry进行实现
SingletonBeanRegistry:定义对羊例的注册及获取
BeanFactory：定义获取bean及bean的各种属性
DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry各函数的实现
HierarchicalBeanFactory ：继承 BeanFactory，也就是在BeanFactory定义的功能的基础上增加了对 parentFactory的支持
BeanDefinitionRegistry：定义对 BeanDefinition 的各种增删改操作
FactoryBeanRegstrySupport ：在 DefaultSingletonBeanRegist 基础上增加了对 FactoryBean的特殊处理功能
ConfigurableBeanFactory ：提供配置Factory的各种方法
ListableBeanFactory ：根据各种条件获取bean的配直清单
AbstractBeanFactory ：综合 FactoryBeanRegistrySupport和ConfigurableBeanFactory功能
AutowireCapableBeanFactory ：提供创建bean、自动注入、初始化以及应用 bean 的后处理器
AbstractAutowireCapabeBeanFactory ：综合AbstractBeanFacto1y并对接口AutowireCapableBeanFactory进行实现
ConfigurableListableBeanFactory : Beanfactory配置清单，指定忽略类型及接口等
DefaultListableBeanFactory ：综合上面所有功能，主要是对 bean 注册后的处理

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

![1568591560605](C:\Users\xia\AppData\Roaming\Typora\typora-user-images\1568591560605.png)

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
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```

上面的代码看似很长，其实只做了来两件事：1.加载xml文件，获取Document对象；2.根据获取的Document对象，对Bean的信息进行解析。

### 一步步来，加载xml文件，获取Document对象。

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
        //开始负责解析的主类DefaultBeanDefinitionDocumentReader的表演，createReaderContext(resource)获取XmlReaderContext，它是xml文档解析期间的上下文，贯穿整个解析过程
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        //新增的beanDefinition数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法：

```java
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext; 
        //获取doc的根节点，也就是<beans></beans>
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
```

进一步调用doRegisterBeanDefinitions方法，获取profile属性，执行真正的解析动作

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

### 解析默认命名空间的元素

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

#### 对`<bean>`元素进行解析

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
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

在上面代码中，生成的AbstractBeanDefinition类型对象，将<bean></bean>元素所可能包含的属性都以硬编码的方式作为类的属性。

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

#### 注册BeanDefinition

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

这两种注册方式原理上大同小异，都是将beanDefinition保存到ConcurrentHashMap中。不同的是通过beanName方式进行注册的保存的值为BeanDefinition，而通过alias方式注册保存的值为beanName。

#### 对`<alias>`标签的解析

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

#### 对`<import>`标签的解析

在parseDefaultElement方法中调用importBeanDefinitionResource(ele)方法进行进一步解析，这段代码太长了，就不贴了，其实思路就是获取到被导入的xml文件的地址，然后通过`getReaderContext().getReader().loadBeanDefinitions(location, actualResources);`对被导入的文件进行解析。

### 自定义标签的解析

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

在readContext初始化时就将属性初始化为DefaultNamespaceHandlerResolver。resolve()方法的作用就是根据spring.handlers文件，将文件中的类名通过反射转换成class对象，然后执行自定义handler的init方法进行初始化，最后将自定义的handler对象返回。

**handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));**

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

```java
	public final BeanDefinition parse(Element element, ParserContext parserContext) {
		//对BeanDefinition进行数据准备，如class，scope等，然后执行子类重写重写的doParse方法执行自定义的解析逻辑
        AbstractBeanDefinition definition = parseInternal(element, parserContext);
		if (definition != null && !parserContext.isNested()) {
			try {
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
                //将AbstractBeanDefinition转换为BeanDefinitionHolder
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				String msg = ex.getMessage();
				parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
				return null;
			}
		}
		return definition;
	}
```

### 小结

至此已经完成了xml文件的加载和解析过程。我们从XmlBeanFactory类开始解析过程主要涉及XmlBeanDefinitionReader、DefaultDocumentLoader、DefaultBeanDefinitionDocumentReader、BeanDefinitionParserDelegate这几个类。

再XmlBeanFactory中包含一个XmlBeanDefinitionReader对象，负责xml文件的加载。XmlBeanDefinitionReader又将把xml文件包装成Document对象的任务交给了DefaultDocumentLoader类，将document的解析交给DefaultBeanDefinitionDocumentReader类，该类又对spring默认命名空间的元素和自定义命名空间的元素进行区分，交给BeanDefinitionParserDelegate进行解析。最后解析结束将解析得到的BeanDefintion注册到Register中，注册时分为name和alias类型进行注册。而那个Register就是XmlBeabFactory。。。

所以。。。这三天就搞了怎么把xml中定义的bean封装为BeanDefinition注册到Register中这点事。。。

有时候想想很多事情真的想一想是很简单事，我学了两年的java了，回想一下好像并不会很多东西，但是就这点东西就花了我两年的时间啊。。。再往前想一想，高中三年，物理课，平均一天一节。等到我高考结束时候，仿佛也就是背下来几个公式，明白些物理定理。其实一张纸就可以把四年的知识总结下来。。。

为什么学习的过程那么慢，学习的过程是一个探索的过程 ，前面对于我来说就是一个黑箱。每走一步都有可能走错、走弯路、掉坑里。所以学习还是要有一位好的老师，他可以是一位前辈（当然我没有。。。）、可以是一本书、可以是网上对这个知识的总结。。。一个方向太重要了，这可能就是教育的问题了，为什么老师好的学校学生就普遍优秀，方向对了，路对了就可以节省至少百分之八十的时间。所以，方向是很重要的，好的方向省下来时间，省下来的时间可以做更多的尝试，体验更多的东西。就像我现在，我非常急切的想把java掌握好，这样我就可以把我吃饭的工具掌握好，然后有更多的时间更多的心情去做更多我好奇、我感兴趣的事情。如果真的有那一天。。。真好。。。

其实很同情现在的学生，很多老师都是个大混子，回想这么多年的求学经历，很少能见到老师救了一个孩子，更多的见了老师毁掉一个孩子，老师如何否定一个孩子。现在的老师真的都是一堆大学毕业招不到工作的，不知道自己可以干什么的，希望有个工作可以混下去的选择去做老师。这么搞下去真的可能不知道有多少孩子会被毁掉。自己的三观不正硬要育人，自己流连于形式主义要让孩子们跟他们一起。真的是精神空虚到何种地步。最近看到网上关于教师节送礼物的讨论，感觉真的是越垃圾的人越当老师。

一个国家的富强第一步就是教育，老师的质量就是第一位，真的为那些真心希望投入教育的老师感到不公，对那些混子老师感到愤慨。真的应该让优秀的人当老师，提高教师门槛，提高教师待遇。让优秀的人带领让下一代变优秀，把只想着混日子，心里压根没有对教师这个职业敬畏的人全部扫除出去。。。。

## bean的加载

### BeanFactory和FactoryBean

一般情况下，Spring通过反射机制利用<bean>的class属性指定实现类实例化Bean，在某些情况下，实例化Bean过程比较复杂，如果按照传统的方式，则需要在<bean>中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring为此提供了一个org.springframework.bean.factory.FactoryBean的工厂类接口，用户可以通过实现该接口定制实例化Bean的逻辑。FactoryBean接口对于Spring框架来说占用重要的地位，Spring自身就提供了70多个FactoryBean的实现。它们隐藏了实例化一些复杂Bean的细节，给上层应用带来了便利。从Spring3.0开始，FactoryBean开始支持泛型，即接口声明改为FactoryBean<T>的形式。

```java
package org.springframework.beans.factory;  
public interface FactoryBean<T> {  
    T getObject() throws Exception; //返回创建的对象 
    Class<?> getObjectType();  //返回创建的对象的类型
    boolean isSingleton();  //是否为单例
}   
```

使用方法如下：

```java
//目标类实现FactoryBean接口
@Data
public class Student implements FactoryBean<Student> {
    private String name;
    private int age;

    public Student() {
    }
    
    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }
	//返回对象
    public Student getObject() throws Exception {
        return new Student("xia",23);
    }

    public Class<?> getObjectType() {
        return Student.class;
    }
}
```

在xml文件中声明bean

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="student" class="demo.Student">
    </bean>
</beans>
```

从spring中取出实现FactoryBean接口的bean时会自动执行getObject方法，获取的Bean就是getObject方法的返回值。如果要取出实现FactoryBean的类的对象，则需要在beanName前面加上`&`符号。

```java
public class IOCTest {
    public static void main(String[] args) {
        ClassPathResource resource = new ClassPathResource("config/applicationContext.xml");
        XmlBeanFactory xmlBeanFactory = new XmlBeanFactory(resource);
        //不加&符号，取出的bean为getObject方法的返回值
        Object student = xmlBeanFactory.getBean("student");
        //加上&符号，取出的bean为实现FactoryBean的类的对象
        Object student = xmlBeanFactory.getBean("student");
        System.out.println(student);
    }
}
```

那么BeanFactory是什么呢？BeanFactory是一个IOC容器啊。。。

### XML配置和Bean的生命周期

在看源码过程中发现对spring的常用配置，bean的生命周期的掌握还是有所欠缺。所以打算回头复习巩固一下这两方面知识。

### XML配置

#### `<beans>`
xml配置文件中的最顶层元素，它下面包含0或1个<description>以及多个<bean>、<import>、<alias>元素。

`<beans>`支持的属性：

* default-lazy-init。其值可以指定为true或者false，默认值为false。用来标志是否对所
  有的进行延迟初始化。
* default-autowire。可以取值为no、byName、byType、constructor以及autodetect。默
  认值为no，如果使用自动绑定的话，用来标志全体bean使用哪一种默认绑定方式。
* default-dependency-check。可以取值none、objects、simple以及all，默认值为none，
  即不做依赖检查。
* default-init-method。如果所管辖的按照某种规则，都有同样名称的初始化方法的
  话，可以在这里统一指定这个初始化方法名，而不用在每一个上都重复单独指定。
* default-destroy-method。与default-init-method相对应，如果所管辖的bean有按照某种
  规则使用了相同名称的对象销毁方法，可以通过这个属性统一指定。

`<beans>`下的子元素含义：

* <description>：配置一些描述信息
* <import>: 合并一些spring的配置文件

* <alias>：为bean进行起别名
* <bean>: 就是定义spring的bean啊

#### `<bean>`

**`<bean>的属性`**

* id：定义bean的id，作为bean的唯一标识
* class：指定该bean的class类型
* depends-on: 当我们需要在对象中使用另一个对象时，可以通过构造方法或者变量进行注入，以保证它是存在的并且已经准备完成。而有时我们并不需要在对象声明时候对它进行注入，但是还需要在创建这个对象时检查它是否已经存在，这是就是用depends-on属性了。
* autowire：为bean的依赖提供自动注入的功能。支持以下值：no，不采用自动绑定；byName，使用beanName作为注入的依据；byType，根据类型作为注入的依据；constructor，前两种都是为property进行注入，这个是为构造函数进行注入，匹配方式为byType；autodetect，是byType和constructor的结合体，会为构造函数和property使用byType模式进行注入。还可以再<beans>元素中设置default-autowire属性指定该beans下的bean默认自动注入行为。
* lazy-init：主要针对ApplicationContext容器的bean，因为它在启动时候会默认加载所有singleton的bean，设置该属性后会进行懒加载，在使用时候在进行加载bean。
* parent：有时候会存在两个bean间使用相同的属性，而且注入的属性值也相同的情况。为了减少代码书写量，可以使用parent来指定从一个bean中继承相同的属性的注入。
* abstract：作为一个bean的模板，通常和parent属性共同属性，使用的abstract属性后可以不指定class属性。因为使用了abstract属性的bean不会被实例化。
* scope：限定bean的作用域。
* 工厂方法：如果一个bean是通过 工厂方法获得的，可以在bean定义时指定工厂方法的属性，工厂方法又分为静态工厂方法和非静态工厂方法之分。配置时要分开配置。
  + 配置静态工厂方法时，指定bean的class属性为静态工厂方法所在的类名，factory-method为工厂方法的名称。如：`<bean id="stu" class="....StudentFactory" factory-method="getStudent"/>`
  + 配置非静态工厂方法时，需要使用工厂方法所在的bean的引用，然后再声明工厂方法。`<bean id="stu" factory-bean="stuFactory" factory-method="getStu">`
  + 当工厂方法存在需要参数的时候，可以使用<constructor-arg>来指定方法调用的参数。

**`<bean>的子元素`**

* <constructor-arg>：为构造方法的参数进行注入。该元素可以使用如下属性：type，要注入的属性的类型；index，所传入的依赖在构造方法顺序的第几位；
* <property>: 为bean中的变量进行注入。该元素可以使用如下属性：name，用来指定将要注入的变量名称。
* <constructor-arg>和<property>的可配置项：

  * <value>: 可以注入一个简单数据类型
* <ref>：用来引用容器中其他对象实例。该元素有三个属性可供选择：local，parent，bean。分别代表被引用的实例来自：与当前配置对象在同一配置文件，只能位于当前容器的父容器，既可以在同一文件也可以在父容器。
  * <idref>：如果注入一个所依赖的对象名称，可以使用该元素。它与<vlaue>元素的区别在于，它可以检查所指定的所依赖的对象名称对应的对象是否存在。
* 内部<bean>：有时我们所依赖的对象只有当前一个对象引用，或者不想别的对象通过<ref>引用他，这是就可以使用内部<bean>，内部<bean>可以不指定id属性
  * <list>：如果注入的对象是java.util.List类型或者是数组类型，使用该元素
* <set>：如果注入的对象是java.util.Set类型则使用该元素
  * `<map>` : 如果注入的对象是java.util.Map类型则使用该元素
* <props>:如果注入的对象是java.util.Properties类型则使用该元素，其key和value只能是String类型
  * <null/>:注入一个空对象
* <lookup-method>：方法注入，可以为一个方法注入一个bean。配置时name属性表示所要进行注入的方法，bean属性表示要被注入的bean的名称。
* <replace-method>：方法替换，可以将bean中的一个方法替换为另一个方法。使用时需要一个类实现MethodReplace接口，然后在将要被替换的方法所在的bean定义中添加replace-method元素，name属性表示将要被替换的方法，replacer属性表示实现了MethodReplace接口的类的bean的名称。

### 容器背后的秘密

直接看源码有点hold不住了，很多接口虽然看着熟悉，但是一直没有用过，乍一看想不起来使用场景。所以系统的再回顾一下，借助《spring揭秘》这本书从总览的视角分析一下spring的各个过程。

































