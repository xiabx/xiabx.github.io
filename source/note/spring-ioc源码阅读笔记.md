---
title: spring-ioc源码阅读笔记
author: XIA
date: 2019-09-25 19:33:02
categories:
- 后端
tags:
- Spring
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

- `<description>`：配置一些描述信息
- `<import>`: 合并一些spring的配置文件
- `<alias>`：为bean进行起别名
- `<bean>`: 就是定义spring的bean啊

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

## Spring IOC 执行流程

Spring IOC启动流程总的可以分为两大步骤，容器启动阶段和容器实例化阶段。

![1569061930193](https://xbxblog.bj.bcebos.com/beanPactory%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.png)

容器的加载阶段主要是加载配置的xml文件到内存中，对xml文件的结构与定义信息进行分析，将xml中定义的信息注册到一个BeanDefinition对象中。至于上图中的其他后处理主要指的是BeanFactoryPostProcessor接口的功能，它在bean实例化之前执行，可以用来进行修改已经定义BeanDefinition信息等操作。BeanFactory需要手动注册并执行BeanFactoryPostProcessor接口，ApplicationContext可以自动注册BeanFactoryPostProcessor注册完成后自动执行相关方法。

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

![1569079638040](https://xbxblog.bj.bcebos.com/bean%E5%AE%9E%E4%BE%8B%E5%8C%96%E8%BF%87%E7%A8%8B.png)

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

#### BeanPostProcessor接口

从bean的实例化流程可以知道再bean进行了各种Aware接口的解析之后，下一步就是执行BeanPostProcessor接口的PostProcessorBeforeInitialization方法了。

BeanPostProcessor容易与BeanFactoryPostProcessor混淆。但只要记住BeanPostProcessor是存在于对象实例化阶段，而BeanFactoryPostProcessor则是存在于容器启动阶段，这两个接口就比较容易区分了。

与BeanFactoryPostProcessor通常会处理容器内所有符合条件的BeanDefinition类似，BeanPostProcessor会处理容器内所有符合条件的实例化后的对象实例。

该接口声明了两个方法，他们都会接受一个所要创建的对象的引用，这样可以极大提高扩展性。这两个方法分别在两个不同的时机执行，也就是bean的创建流程图中前置处理与后置处理。见如下代码定义：

```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

该接口的一个使用场景就是再ApplicationContext中各种Aware接口的注入。

当需要自定义该接口实现时只需实现该类并实现相应方法，如果使用BeanFactory容器还需要将其注入到容器中，方法是使用BeanFactory对象的addBeanPostProcessor方法，该方法实际是调用的ConfigurableBeanFactory中的。

> 实际上，有一种特殊类型的BeanPostProcessor我们没有提到，它的执行时机与通常的BeanPostProcessor不同。org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor
>接口可以在对象的实例化过程中导致某种类似于电路“短路”的效果。
>
>实际上，并非所有注册到Spring容器内的bean定义都是按照图4-10的流程实例化的。在所有的步骤之前，也就是实例化bean对象步骤之前，容器会首先检查容器中是否注册有InstantiationAwareBeanPostProcessor类型的BeanPostProcessor。如果有，首先使用相应的InstantiationAwareBeanPostProcessor来构造对象实例。构造成功后直接返回构造完成的对象实例，而不会按照“正规的流程”继续执行。这就是它可能造成“短路”的原因。
>
>不过，通常情况下都是Spring容器内部使用这种特殊类型的BeanPostProcessor做一些动态对象代理等工作，我们使用普通的BeanPostProcessor实现就可以。这里简单提及一下，目的是让大家有所了解。

spring aop自动代理实现类就是InstantiationAwareBeanPostProcessor类型

#### InitializingBean和init-method

在BeanPostProcessor的前置方法调用结束后，接下来如果被实例化的对象实现了InitializingBean接口，那么就会调用afterPropertiesSet方法用为进一步调整实例的状态。

但是使用InitializingBean会显得spring容器具有侵入性，所以一般使用<bean>的init-method属性（或<beans>的default-init-method属性为所有bean指定默认init方法）为bean指定初始化方法。用来调整实例的状态。由bean的实例化流程图可以看出来该方法的执行位于InitializingBean接口的afterPropertiesSet方法之后。

#### DisposableBean与destroy-method方法

与InitializingBean和init-method功能类似，DisposableBean与destroy-method同样是一个是接口一个是`<bean>`的配置属性，他们表示bean的销毁时调用的方法。

但是这个方法并不会自动调用，当在BeanFactory容器中需要调用ConfigurableBeanFactory提供的destroySingletons方法。

在ApplicationContext容器中需要AbstractApplicationContext为我们提供了registerShutdownHook()方法，该方法底层使用标准的Runtime类的addShutdownHook()方式来调用相应bean对象的销毁逻辑，从而保证在Java虚拟机退出之前，这些singtleton类型的bean对象实例的自定义销毁逻辑会被执行。当然AbstractApplicationContext注册的shutdownHook不只是调用对象实例的自定义销毁逻辑，也包括ApplicationContext相关的事件发布等。

同样的道理，在自定义scope之后，使用自定义scope的相关对象实例的销毁逻辑，也应该在合适的时机被调用执行。不过，所有这些规则不包含prototype类型的bean实例，因为prototype对象实例在容器实例化并返回给请求方之后，容器就不再管理这种类型对象实例的生命周期了。

# BeanFactory与ApplicationContext

spring提供了两种容器类型：BeanFactory与ApplicationContext

* BeanFactory:基础类型的ioc容器，提供完整的ioc服务。如果没有特殊指定默认使用懒加载模式，只有当客户端访问某个受管的对象时才会对该对象进行初始化和依赖注入的操作。所以，相对来说BeanFactory的启动速度较快，所需要的资源有限。

* ApplicationContext：ApplicationContext是在BeanFactory的基础上构建的，是相对高级的容器实现。除了Beanfactory提供的特性之外还提供了事件发布、国际化支持等。ApplicationContext在启动时默认就会对所管理的对象进行初始化和依赖注入。所以ApplicationContext比BeanFactory消耗更多的资源。

* 如图表示这两者间的关系：

  ![beanfactory和applicationContext](https://xbxblog.bj.bcebos.com/beanfactory%E5%92%8CapplicationContext.png)

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

![XmlBeanFactory](https://xbxblog.bj.bcebos.com/XmlBeanFactory.png)

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

### 小结

至此已经完成了xml文件的加载和解析过程。我们从XmlBeanFactory类开始解析过程主要涉及XmlBeanDefinitionReader、DefaultDocumentLoader、DefaultBeanDefinitionDocumentReader、BeanDefinitionParserDelegate这几个类。

再XmlBeanFactory中包含一个XmlBeanDefinitionReader对象，负责xml文件的加载。XmlBeanDefinitionReader又将把xml文件包装成Document对象的任务交给了DefaultDocumentLoader类，将document的解析交给DefaultBeanDefinitionDocumentReader类，该类又对spring默认命名空间的元素和自定义命名空间的元素进行区分，交给BeanDefinitionParserDelegate进行解析。最后解析结束将解析得到的BeanDefintion注册到Register中，注册时分为name和alias类型进行注册。而那个Register就是XmlBeabFactory。。。

所以。。。这三天就搞了怎么把xml中定义的bean封装为BeanDefinition注册到Register中这点事。。。

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

### getBean方法

BeanFactory对bean的加载方式为懒加载，只有当BeanFactory对象调用getBean方法时才会对bean进行加载。加载的过程大致分为两种：如果缓存中存在将要获取的bean就从缓存中返回；若缓存中没有要返回的bean则需要根据元数据配置新创建一个bean，并将bean加入缓存中。

当使用getBean方法获取一个bean时，将会在内部调用doGetBean方法。当使用包含beanName的getBean方法时都会进入到doGetBean方法中。至于那些没有传入beanName的getBean方法。。。回头再看吧。

```java
	public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
		return doGetBean(name, requiredType, null, false);
	}
```

### doGetBean方法

这个方法比较长。

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		//1.解析传入的name，将其转换为beanName。他可能是alias或者是带有前缀的FactoryBean
		final String beanName = transformedBeanName(name);
		Object bean;

		//2.检查缓存中或者实例工厂中是否有对应的实例
		//直接尝试从缓存中或者singletonFactories中获取
		Object sharedInstance = getSingleton(beanName);
		//这里取出的sharedInstance并不一定是最终想要的bean，比如说他可能是一个FactoryBean类型
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			//返回对应实例，有时候是FactoryBean类型，则需要返回他的getObject方法的返回值
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
    	//说明缓存中没有beanName对应的bean
		else {
			//只有单例模式才会尝试解决循环依赖,如果beanName对应的bean是原型，则直接抛异常
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			//如果当前beanDefinitionMap中不包含beanName，则尝试从parentBeanFactory中加载
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				//在父BeanFactory中创建bean，针对各种传参进行不同的方法调用
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}
			//仅仅做类型检查，则在这里进行记录
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				//将GernericBeanDefinition转换RootBeanDefinition，因为之后都是用的RootBeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				//singleton模式的bean创建
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				//prototype模式的bean创建
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				//在指定的scope上创建bean
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		//检查需要的类型是否符合bean的实际类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

由此我们可以看到，doGetBean方法一共做了这几件事：

1. 转换beanName。比如将FactoryBean前面的&符号去掉，根据alias获取真实beanName等。
2. 尝试从缓存中加载bean。这里首先从singletonObject中获取，若获取不到则再从singletonFactories中获取，这里是为了解决循环依赖而建立的缓存。
3. bean的实例化。现在获取的bean是原始状态的bean，比如一个FactoryBean，而我们要获取的对象是在该bean的factory-method中返回的bean。这里就是做的这个工作。
4. 原型模式的依赖检查。
5. 检测parentBeanFactory。如果当前beanFactory中不包含将要创建的BeanDefinition就会到其parentBeanFactory中去检测。
6. 将解析XML得到的GernericBeanDefinition转换为RootBeanDefinition。因为在bean的后续处理中都是用的RootBeanDefinition。
7. 验证DependsOn也就是在xml文件中bean定义的depends-on属性。
8. 针对不同的scope进行创建属性。如创建singleton、prototype、request等作用域类型。
9. 类型转换。将bean的类型转换为需要的类型。

### 2 从缓存中获取singletonBean

这里涉及到多个缓存的map：

- singletonObjects：
- earlySingletonObjects：
- singletonFactories：

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   //singletonObjects是一个concurrentHashMap
   Object singletonObject = this.singletonObjects.get(beanName);
   //在singletonObjects中没有 &&
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         //
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```

### 3 从原始bean中获取最终bean

当beanName是一个FactoryBean的类型时，在缓存中获取的bean其实是原始的FactoryBean，而我们想要的是factory-method方法的返回值。这里就是做的这件事。

```java
protected Object getObjectForBeanInstance(
      Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
   
   //如果name前面有一个&符号，表示他是一个FactoryBean类型
   if (BeanFactoryUtils.isFactoryDereference(name)) {
      //空对象，直接返回
      if (beanInstance instanceof NullBean) {
         return beanInstance;
      }
      //加了&符号但不是FactoryBean类型，则直接抛异常
      if (!(beanInstance instanceof FactoryBean)) {
         throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
      }
   }
   
   //现在的beanInstance可能是一个正常的Bean,也可能是一个FactoryBean
   //如果不是FB或者这个name是以&开头。说明他希望的bean就是注册的类的bean
   if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
      return beanInstance;
   }
   //到这里，beanInstance已经确定是一个factoryBean类型了
   Object object = null;
   //mbd = RootBeanDefinition
   //此时为空
   if (mbd == null) {
      //尝试从缓存中加载，根据beanName，这个缓存的结构为beanName：真是的bean（getObject方法的返回的那个）
      object = getCachedObjectForFactoryBean(beanName);
   }
   //缓存中没有
   if (object == null) {
      //转换为FB类型
      FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
      //containsBeanDefinition检测beanDefinnitionMap中是否包含beanName
      if (mbd == null && containsBeanDefinition(beanName)) {
         //将GernericBeanDefinition转换为RootBeanDefinition
         //如果指定beanName是子bean的话，会合并父类相关属性
         mbd = getMergedLocalBeanDefinition(beanName);
      }
      boolean synthetic = (mbd != null && mbd.isSynthetic());
       //委托给该方法去获取bean
      object = getObjectFromFactoryBean(factory, beanName, !synthetic);
   }
   return object;
}
```

此方法首先验证了FactoryBean的正确性，当使用&符号但不是FactoryBean类型时则会报错，当不是BeanFactory或者使用&前缀的FactoryBean则直接返回。然后尝试从缓存中获取，若缓存中不存在则委托getObjectFromFactoryBean方法新创建一个bean。

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
   //factoryBean为单例 并且 singletonObjects存在这个beanName
   if (factory.isSingleton() && containsSingleton(beanName)) {
      synchronized (getSingletonMutex()) {
         //这里就是缓存factory-method返回值的缓存
         Object object = this.factoryBeanObjectCache.get(beanName);
         if (object == null) {
            //这里的object就是返回的getObject方法的返回值了
            object = doGetObjectFromFactoryBean(factory, beanName);

            Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
            if (alreadyThere != null) {
               object = alreadyThere;
            }
            else {
               if (shouldPostProcess) {
                  if (isSingletonCurrentlyInCreation(beanName)) {
                     // Temporarily return non-post-processed object, not storing it yet..
                     return object;
                  }
                  beforeSingletonCreation(beanName);
                  try {
                     object = postProcessObjectFromFactoryBean(object, beanName);
                  }
                  catch (Throwable ex) {
                     throw new BeanCreationException(beanName,
                           "Post-processing of FactoryBean's singleton object failed", ex);
                  }
                  finally {
                     afterSingletonCreation(beanName);
                  }
               }
               if (containsSingleton(beanName)) {
                	//进行缓存
                   this.factoryBeanObjectCache.put(beanName, object);
               }
            }
         }
         return object;
      }
   }
   else {
      Object object = doGetObjectFromFactoryBean(factory, beanName);
      if (shouldPostProcess) {
         try {
            object = postProcessObjectFromFactoryBean(object, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
         }
      }
      return object;
   }
}
```

此方法主要维护了一个缓存，当我们需要一个单例模式的bean时，将会把创建好的bean保存到缓存中。但是这个方法也并没有我们想要看到的factory.getObject()代码，这句代码在被委托给了doGetObjectFromFactoryBean去执行。

### 8 获取单例

回到doGetBean方法，之前的分析都是能在缓存中获取到单例的情况。如果缓存中获取不到单例，那么就需要进行创建了。在doGetBean方法中通过getSingleton进行单例bean的创建。

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(beanName, "Bean name must not be null");
   synchronized (this.singletonObjects) {
      //检查bean是否已经被加载过
      Object singletonObject = this.singletonObjects.get(beanName);
      //没有被加载过，开始bean的初始化
      if (singletonObject == null) {
         if (this.singletonsCurrentlyInDestruction) {
            throw new BeanCreationNotAllowedException(beanName,
                  "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                  "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
         }
         //记录加载状态
         beforeSingletonCreation(beanName);
         boolean newSingleton = false;
         boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
         if (recordSuppressedExceptions) {
            this.suppressedExceptions = new LinkedHashSet<>();
         }
         try {
            //初始化bean
            singletonObject = singletonFactory.getObject();
            newSingleton = true;
         }
         catch (IllegalStateException ex) {
            // Has the singleton object implicitly appeared in the meantime ->
            // if yes, proceed with it since the exception indicates that state.
            singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
               throw ex;
            }
         }
         catch (BeanCreationException ex) {
            if (recordSuppressedExceptions) {
               for (Exception suppressedException : this.suppressedExceptions) {
                  ex.addRelatedCause(suppressedException);
               }
            }
            throw ex;
         }
         finally {
            if (recordSuppressedExceptions) {
               this.suppressedExceptions = null;
            }
            afterSingletonCreation(beanName);
         }
         if (newSingleton) {
            //将结果记录至缓存，并删除bean加载过程中各种辅助状态
            addSingleton(beanName, singletonObject);
         }
      }
      return singletonObject;
   }
}
```

该方法首先检查缓存中是否真的没有将要加载的bean。在初始化单例前通过beforeSingletonCreation记录单例状态，通过调用singletonFactory获取单例对象，执行加载单例后的处理方法afterSingletonCreation，最后将获取到的单例结果进行缓存并删除创建单例过程中记录的各种中间状态。

### 准备创建bean

上一个方法中，通过回调singletonFactory.getObject()来创建一个新的bean。getObject方法如下：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   //获取将要创建的bean的class对象
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   //验证及准备覆盖的方法
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      //短路拦截
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
      //真正执行创建的方法
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

方法先获取将要创建bean的calss对象。然后对override属性进行标记，该属性是在解析XML时保存的lookup-method和replace-method的配置，这里解析主要时对该bean将要被覆盖的方法进行预处理，包括校验所要覆盖的方法是否存在，如果存在且只存在一个则增加一个未被重载的标记。然后执行一个短路拦截操作，通过执行resolveBeforeInstantiation方法，如果容器有注册InstantiationAwareBeanPostProcessor类型的处理器，该处理器时BeanPostProcessor的一个子类，如果处理器返回一个bean，则方法返回，后面的创建bean的操作将会被短路。如果此方法未被短路，则执行doCreateBean方法来创建新的bean。

### 创建bean

如上一节代码所示，当通过了短路方法后，就开始真正的创建一个bean了。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      /** Cache of unfinished FactoryBean instances: FactoryBean name to BeanWrapper. */
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      //根据指定bean使用对应的策略创建新的实例，如工厂方法、构造函数自动注入、简单初始化
      //返回BeanWrapper对象
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            //postProcessMergedBeanDefinition方法执行
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   //单例 允许循环依赖 当前bean正在创建中  检测循环依赖
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      //未避免后期循环依赖，可以在bean初始化完成前将创建实例的ObjectFactory加入工厂
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
      //属性值填充，可能依赖于其他的bean，则会递归初始化依赖bean
      populateBean(beanName, mbd, instanceWrapper);
      //调用初始化方法，如init-method
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }

   if (earlySingletonExposure) {
      Object earlySingletonReference = getSingleton(beanName, false);
      //
      if (earlySingletonReference != null) {
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,
                     "Bean with name '" + beanName + "' has been injected into other beans [" +
                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                     "] in its raw version as part of a circular reference, but has eventually been " +
                     "wrapped. This means that said other beans do not use the final version of the " +
                     "bean. This is often the result of over-eager type matching - consider using " +
                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // Register bean as disposable.
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}
```

1. 如果是单例的bean则会首先清除缓存。
2. 实例化bean，将bean包装为beanWrapper
3. postProcessMergedBeanDefinition方法执行
4. 处理依赖，为了解决循环依赖，因为此时已经获取了原始的bean（只是新建出来，并没有进行属性填充），通过`addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));`,将ObjectFactory进行缓存，getEarlyBeanReference返回bean的引用。
5. 属性填充
6. 循环依赖检测
7. 注册DisposableBean，以及配置的destroy-method方法
8. 返回创建的bean

#### 2.实例化bean

createBeanInstance方法的主要作用就是根据不同的情况使用不同的策略，最后货地一个原始的bean，将其封装到BeanWrapper中。

如果bean中定义了factory-method则直接通过instantiateUsingFactoryMethod类获取beanWrapper。如果没有配置工厂方法，下一步对mbd中的resolvedConstructorOrFactoryMethod属性进行检测，查看是否存在过解析的情况以获得构造函数。如果该baen对应的calss存在自定义构造，则调用autowireConstructor进行解析并创建BeanWrapper，否则调用instantiateBean方法，使用空参构造来创建BeanWrapper。

autowireConstructor有点复杂，概括起来主要做了这几件事，确定构造参数，确定构造方法，调用实例化策略获得原始bean，将原始bean封装为BeanWrapper。

实例化策略是根据前面确定的信息，如类、构造方法、构造参数等。实例化策略有两种实现，SimpleInstantiationStrategy和CglibSubClassingInstantiationStrategy。默认调用的是前一个策略，当检测到mbd中methodOverrid非空时，说明存在方法替换情况，此时会调用第二个策略生成cglib的代理对象。

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   // Make sure bean class is actually resolved at this point.
   //解析class
   Class<?> beanClass = resolveBeanClass(mbd, beanName);

   if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   }

   Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
   }

   //工厂方法不为空则使用工厂方法策略进行初始化
   if (mbd.getFactoryMethodName() != null) {
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }

   // Shortcut when re-creating the same bean...
   boolean resolved = false;
   boolean autowireNecessary = false;
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
         //因为根据构造参数找到对应的构造函数是一个很复杂，很浪费性能的过程。
         //当这个beanDefinition被解析过后则会在resolvedConstructorOrFactoryMethod进行缓存
         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            resolved = true;
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
   //存在缓存的构造方法
   if (resolved) {
      if (autowireNecessary) {
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
         return instantiateBean(beanName, mbd);
      }
   }

   //
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      return autowireConstructor(beanName, mbd, ctors, args);
   }

   // Preferred constructors for default construction?
   ctors = mbd.getPreferredConstructors();
   if (ctors != null) {
      return autowireConstructor(beanName, mbd, ctors, null);
   }

   // No special handling: simply use no-arg constructor.
   return instantiateBean(beanName, mbd);
}
```

### 循环依赖

#### 循环依赖是什么

循环依赖就是循环引用。当两个或多个bean之间相互持有对象，就构成了循环依赖。例如：beanA引用beanB，beanB引用beanC，beanC又引用beanA。

#### 三种不同的循环依赖形式

**构造器依赖:**通过构造器注入形成的循环依赖，此依赖是无法解决的。只能抛出BeanCurrentlyInCreationException异常。在spring中将每一个正在创建的额bean标识符放在一个“当前创建池”中，如果bean的创建过程中发现自己已经在“当前创建池”中，将会抛出BeanCurrentlyInCreationException异常。

**setter循环依赖：**对于setter构成的依赖是通过spring容器提前暴露刚完成构造器注入但未完成其他步骤的bean来完成的，而且只能解决单例作用域的bean循环依赖。通过提前暴露一个单例工厂方法，从而使其他bean能引用到该bean。

**prototype范围内的依赖处理：**对于prototype作用域的bean，spring无法解决循环依赖问题，因为spring容器不进行缓存prototype类型的bean，因此无法提前暴露一个创建中的bean。

#### spring解决循环依赖具体方式

spring通过将已实例化但未属性的注入的bean提前暴露来解决循环依赖。主要通过singletonFactories 缓存来实现。其中涉及这几个的方法：`getSingleton(String beanName, boolean allowEarlyReference)` `getSingleton(String beanName, ObjectFactory<?> singletonFactory)` `beforeSingletonCreation(beanName);`  `addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory)`。

setter形式的循环依赖参考<https://www.cnblogs.com/zzq6032010/p/11406405.html>。站在巨人的肩膀上。。。

构造方法循环依赖的检测位于`beforeSingletonCreation(beanName)`，当对singletonsCurrentlyInCreation 属性进行add时，若返回false则表示重复创建了一个单例bean，且存在构造函数循环依赖。抛出错误BeanCurrentlyInCreationException。

#### 参考

- Spring中的循环依赖解决详解：<https://www.cnblogs.com/zzq6032010/p/11406405.html>

### 属性注入

当实例化bean以后，下一步就是要进行属性注入，调用populateBean方法。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   //beanWrapper为空但是存在要注入的属性，报错。不存在要注入的属性，返回。
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // Skip property population phase for null instance.
         return;
      }
   }


   boolean continueWithPropertyPopulation = true;
   //执行InstantiationAwareBeanPostProcessor的接口，返回false时则终止属性注入。
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }

   if (!continueWithPropertyPopulation) {
      return;
   }


   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
   //当配置了自动注入，则进行自动解析。默认不走这里
   int resolvedAutowireMode = mbd.getResolvedAutowireMode();
   if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // 通过name进行自动注入
      if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }
      // 通过类型进行自动注入
      if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
   }


   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

   PropertyDescriptor[] filteredPds = null;
   if (hasInstAwareBpps) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      //执行后处理
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
               if (filteredPds == null) {
                  filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
               }
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
   if (needsDepCheck) {
      if (filteredPds == null) {
         filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
      checkDependencies(beanName, mbd, filteredPds, pvs);
   }

   if (pvs != null) {
      //将依赖的属性进行注入
      //当没有自动注入的情况下，还需要进行转换
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

1. 检查是否要进行属性注入
2. 根据注入类型（byName/byType），提取依赖的bean，并存入PropertyValues中。
3. 调用InstantiationAwareBeanPostProcessor处理器的postProcessAfterInstantiation方法。此方法控制程序是否继续进行属性填充
4. 将所有PropertyValues属性填充到BeanWrapper中

#### autowireByName

```java
protected void autowireByName(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
   //获取beanWrapper中需要依赖注入的属性
   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
      if (containsBean(propertyName)) {
         Object bean = getBean(propertyName);
         //直接调用getBean方法
         pvs.add(propertyName, bean);
         registerDependentBean(propertyName, beanName);
         if (logger.isTraceEnabled()) {
            logger.trace("Added autowiring by name from bean name '" + beanName +
                  "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
         }
      }
      else {
         if (logger.isTraceEnabled()) {
            logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                  "' by name: no matching bean found");
         }
      }
   }
}
```

#### autowireByType

```java
protected void autowireByType(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

   TypeConverter converter = getCustomTypeConverter();
   if (converter == null) {
      converter = bw;
   }

   Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
      try {
         //属性描述器，用于获取属性的相关信息
         PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
         if (Object.class != pd.getPropertyType()) {
            //获取相关属性的写方法参数
            MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
            // Do not allow eager init for type matching in case of a prioritized post-processor.
            //是否允许懒加载
            boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
            //setter方法的属性依赖包装器
            DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
            //主要解析方法
            Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
            if (autowiredArgument != null) {
               pvs.add(propertyName, autowiredArgument);
            }
            for (String autowiredBeanName : autowiredBeanNames) {
               //注册依赖
               registerDependentBean(autowiredBeanName, beanName);
               if (logger.isTraceEnabled()) {
                  logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
                        propertyName + "' to bean named '" + autowiredBeanName + "'");
               }
            }
            autowiredBeanNames.clear();
         }
      }
      catch (BeansException ex) {
         throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
      }
   }
}
```

**resolveDependency**

```java
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

   descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
   if (Optional.class == descriptor.getDependencyType()) {
      return createOptionalDependency(descriptor, requestingBeanName);
   }
   else if (ObjectFactory.class == descriptor.getDependencyType() ||
         ObjectProvider.class == descriptor.getDependencyType()) {
      return new DependencyObjectProvider(descriptor, requestingBeanName);
   }
   else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
      return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
   }
   else {
      Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
            descriptor, requestingBeanName);
      if (result == null) {
          //通用的的解析方法
         result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
      }
      return result;
   }
}
```

该方法首先对一些特殊情况进行过滤，包括Optional类，ObjectFactory类等，当过滤完成后通过doResolveDependency进一步解析。

**doResolveDependency**

```java
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

   InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
   try {
      Object shortcut = descriptor.resolveShortcut(this);
      if (shortcut != null) {
         return shortcut;
      }
      //注入的类型
      Class<?> type = descriptor.getDependencyType();
      //对@value处理
      Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
      if (value != null) {
         if (value instanceof String) {
            String strVal = resolveEmbeddedValue((String) value);
            BeanDefinition bd = (beanName != null && containsBean(beanName) ?
                  getMergedBeanDefinition(beanName) : null);
            value = evaluateBeanDefinitionString(strVal, bd);
         }
         TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
         try {
            return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
         }
         catch (UnsupportedOperationException ex) {
            // A custom TypeConverter which does not support TypeDescriptor resolution...
            return (descriptor.getField() != null ?
                  converter.convertIfNecessary(value, type, descriptor.getField()) :
                  converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
         }
      }

      //对于数组、容器的处理
      Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
      if (multipleBeans != null) {
         return multipleBeans;
      }
      //对非数组、容器进行处理
      Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
      if (matchingBeans.isEmpty()) {
         if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
         }
         return null;
      }

      String autowiredBeanName;
      Object instanceCandidate;

      //如果类型匹配的bean不止一个，Spring需要进行筛选，筛选失败的话抛出异常
      if (matchingBeans.size() > 1) {
         autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
         if (autowiredBeanName == null) {
            if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
               return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
            }
            else {
               return null;
            }
         }
         instanceCandidate = matchingBeans.get(autowiredBeanName);
      }
      else {
         //只有一个bean与类型匹配，那么直接使用该bean
         Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
         autowiredBeanName = entry.getKey();
         instanceCandidate = entry.getValue();
      }

      if (autowiredBeanNames != null) {
         autowiredBeanNames.add(autowiredBeanName);
      }
      //如果获取到instanceCandidate是Class类型
      //那么还需要beanFactory.getBean(autowiredBeanName, instanceCandidate)来获取bean的实例
      if (instanceCandidate instanceof Class) {
         instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
      }
      Object result = instanceCandidate;
      if (result instanceof NullBean) {
         if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
         }
         result = null;
      }
      if (!ClassUtils.isAssignableValue(type, result)) {
         throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
      }
      return result;
   }
   finally {
      ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
   }
}
```

**findAutowireCandidates**

```java
protected Map<String, Object> findAutowireCandidates(
      @Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {
   //查找符合条件的beanName，原理就是对注册的beanDefinition进行遍历，类型匹配的就返回
   String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
         this, requiredType, true, descriptor.isEager());
   //返回结果
   Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
   //特殊类型？
   for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
      Class<?> autowiringType = classObjectEntry.getKey();
      if (autowiringType.isAssignableFrom(requiredType)) {
         Object autowiringValue = classObjectEntry.getValue();
         autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
         if (requiredType.isInstance(autowiringValue)) {
            result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
            break;
         }
      }
   }
   //实际就是调用了getBean方法，将beanName转换为bean，加入result
   for (String candidate : candidateNames) {
      if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
         addCandidateEntry(result, candidate, descriptor, requiredType);
      }
   }
   if (result.isEmpty()) {
      boolean multiple = indicatesMultipleBeans(requiredType);
      // Consider fallback matches if the first pass failed to find anything...
      DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
      for (String candidate : candidateNames) {
         if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor) &&
               (!multiple || getAutowireCandidateResolver().hasQualifier(descriptor))) {
            addCandidateEntry(result, candidate, descriptor, requiredType);
         }
      }
      if (result.isEmpty() && !multiple) {
         // Consider self references as a final pass...
         // but in the case of a dependency collection, not the very same bean itself.
         for (String candidate : candidateNames) {
            if (isSelfReference(beanName, candidate) &&
                  (!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
                  isAutowireCandidate(candidate, fallbackDescriptor)) {
               addCandidateEntry(result, candidate, descriptor, requiredType);
            }
         }
      }
   }
   return result;
}
```

#### applyPropertyValues

将propertyValues注入到beanWrapper中，如果之前没有设置自动装配，那么这里还需要根据依赖的beanName通过getBean方法获取所依赖的对象。最后通过beanWrapper的setPropertyValues方法将属性设置到bean中。

#### 自动注入:@Autowired

```java
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
               if (filteredPds == null) {
                  filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
               }
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
```

在进行属性注入时，通过后置处理器对@Autowired进行了自动注入。通过`AutowiredAnnotationBeanPostProcessor`类实现。

#### 参考

* populateBean实现：<https://blog.csdn.net/qwe6112071/article/details/85225507>
* resolveDependency方法详解：<https://blog.csdn.net/finalcola/article/details/81537380>
* Spring 工具类 DependencyDescriptor：<https://blog.csdn.net/andy_zhang2007/article/details/88135669>
* @Autowired自动注入解析：<https://www.cnblogs.com/liferecord/p/7281655.html>
* Spring beans架构（看着很不错，没见过的船新版本，但是看起来没头没尾的，mark一下）：<https://blog.csdn.net/szwandcj/article/details/50688616>

### 初始化Bean

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
      //执行Aware方法
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      //执行BeanPostProcessor接口的前置方法
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      //执行init方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
      //执行BeanPostProcessor接口的后置方法
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

#### 激活Aware

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
   if (bean instanceof Aware) {
      if (bean instanceof BeanNameAware) {
         ((BeanNameAware) bean).setBeanName(beanName);
      }
      if (bean instanceof BeanClassLoaderAware) {
         ClassLoader bcl = getBeanClassLoader();
         if (bcl != null) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
         }
      }
      if (bean instanceof BeanFactoryAware) {
         ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
      }
   }
}
```

#### BeanPostProcessor接口的方法

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessBeforeInitialization(result, beanName);
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

#### init方法

```java
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
      throws Throwable {
   //是否实现了InitializingBean接口
   boolean isInitializingBean = (bean instanceof InitializingBean);
   if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
      if (logger.isTraceEnabled()) {
         logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
      }
      if (System.getSecurityManager() != null) {
         try {
            AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
               ((InitializingBean) bean).afterPropertiesSet();
               return null;
            }, getAccessControlContext());
         }
         catch (PrivilegedActionException pae) {
            throw pae.getException();
         }
      }
      else {
         //执行afterPropertiesSet方法
         ((InitializingBean) bean).afterPropertiesSet();
      }
   }
   
   if (mbd != null && bean.getClass() != NullBean.class) {
      //获取在bean中定义的init方法
      String initMethodName = mbd.getInitMethodName();
      if (StringUtils.hasLength(initMethodName) &&
            !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
            !mbd.isExternallyManagedInitMethod(initMethodName)) {
         //调用自定义初始化方法
         invokeCustomInitMethod(beanName, bean, mbd);
      }
   }
}
```

# 啊哈，ApplicationContext

## ClassPathXmlApplicationContext

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

## super(parent);

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

## setConfigLocations(configLocations)

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
   //获取Environment，进行解析
   return getEnvironment().resolveRequiredPlaceholders(path);
}
```

getEnvironment()中如果this.environment为空则调用createEnvironment()方法创建新的StandardEnvironment对象。

```java
protected ConfigurableEnvironment createEnvironment() {
    return new StandardEnvironment();
}
```

### Environment

![Environment](https://xbxblog.bj.bcebos.com/Environment.jpg)

Environmen接口**代表了当前应用所处的环境。**从此接口的方法可以看出，其主要和profile、Property相关。

**构造方法：**

StandardEnvironment没有显示的构造方法，其父类AbstractEnvironment中构造方法如下：

```java
private final MutablePropertySources propertySources = new MutablePropertySources();
/** System environment property source name: {@value} */
public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";
/** JVM system properties property source name: {@value} */
public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";

//父类构造
public AbstractEnvironment() {
   customizePropertySources(this.propertySources);
}

protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(
				new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
		propertySources.addLast(
				new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
}
```

MutablePropertySources实现了**PropertySources**接口，其作用是作为PropertySource的容器。内部包含一个CopyOnWriteArrayList作为容器。

customizePropertySources()是向MutablePropertySources中注册PropertySource。getSystemProperties(）底层调用的`System.getProperties()`，getSystemEnvironment()底层调用的 `System.getenv()`。

**PropertySource**接口代表了键值对的Property来源。内部包含name和source属性，代表property的来源和值。继承体系：

![PropertySource](https://xbxblog.bj.bcebos.com/PropertySource.jpg)

到这里StandardEnvironment对象已经创建完成。接下来就是`resolveRequiredPlaceholders(path)`方法了。

### resolveRequiredPlaceholders(path)

StandardEnvironment并没有重写父类AbstractEnvironment中的这个方法

AbstractEnvironment.resolveRequiredPlaceholders:

```java
private final ConfigurablePropertyResolver propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);

public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
   return this.propertyResolver.resolveRequiredPlaceholders(text);
}
```

**PropertyResolver**继承关系图：

![PropertyResolver](https://xbxblog.bj.bcebos.com/PropertyResolver.jpg)

resolveRequiredPlaceholders()最终调用是在AbstractPropertyResolver中。

AbstractPropertyResolver.resolveRequiredPlaceholders:

```java
public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
   if (this.strictHelper == null) {
      this.strictHelper = createPlaceholderHelper(false);
   }
   return doResolvePlaceholders(text, this.strictHelper);
}
```

```java
private PropertyPlaceholderHelper createPlaceholderHelper(boolean ignoreUnresolvablePlaceholders) {
    //三个参数分别是${, }, :
   return new PropertyPlaceholderHelper(this.placeholderPrefix, this.placeholderSuffix,
         this.valueSeparator, ignoreUnresolvablePlaceholders);
}
```

```java
private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
   return helper.replacePlaceholders(text, this::getPropertyAsRawString);
}
protected String getPropertyAsRawString(String key) {
	return getProperty(key, String.class, false);
}
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				Object value = propertySource.getProperty(key);
				if (value != null) {
					if (resolveNestedPlaceholders && value instanceof String) {
						value = resolveNestedPlaceholders((String) value);
					}
					logKeyFound(key, propertySource, value);
					return convertValueIfNecessary(value, targetValueType);
				}
			}
		}
	return null;
}
```

## refresh()

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

				// I留给子类来初始化其他的Bean
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

​      @Qualifer与@Autowired应该是大家非常熟悉的注解,那么这两个注解正是在这- - 步骤中增加的支持。

4. 子类覆盖方法做额外的处理。

   Spring之所以强大，为世人所推崇，除了它功能上为大家提供了便例外,还有一方面是它的完美架构，开放式的架构让使用它的程序员很容易根据业务需要扩展已经存在的功能。这种开放式的设计在Spring 中随处可见，例如在本例中就提供了- .个空的函数实现postProcess-BeanFactory来方便程序员在业务上做进一步扩展。

5. 激活各种BeanFactory处理器。

6. 注册拦截bean创建的bean处理器，这里只是注册，真正的调用是在getBean时候。
7. 为上下文初始化Message源，即对不同语言的消息体进行国际化处理。
8. 初始化应用消息广播器，并放入“applicationEventMulticaster” bean 中。
9. 留给子类来初始化其他的bean。

10. 在所有注册的bean中查找listener bean,注册到消息广播器中。
11. 初始化剩下的单实例(非惰性的)。

12. 完成刷新过程，通知生命周期处理器lifecycleProcessor 刷新过程，同时发出Context-RefreshEvent通知别人。

### 环境准备：prepareRefresh()

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

### 初始化BeanFactory：obtainFreshBeanFactory()

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
      //以及是否允许循环依赖，设置@AutoWired和@Qualifier注解的解析器
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

#### 加载BeanDefinition：loadBeanDefinitions(beanFactory)

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

### 功能扩展：prepareBeanFactory(beanFactory)

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

   // 增加默认的系统环境bean
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

#### 属性编辑器的注册

`beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));`

进行默认属性编辑器注册的方法是ResourceEditorRegistrar中的registerCustomEditors方法。

该放在AbstractBeanFactory类的initBeanWrapper中被调用。

#### 增加一些内置类

`beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));`

ApplicationContextAwareProcessor是一个BeanPostProcessor

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

### BeanFactoryPostProcessor的调用

`this.invokeBeanFactoryPostProcessors(beanFactory);`对spring中注册的BeanFactoryPostProcessors进行invoke。

该方法通过委托PostProcessorRegistrationDelegate类的invokeBeanFactoryPostProcessors方法进行执行。主要思路为先遍历硬编码的BeanFactoryPostProcessors，即通过AbstractApplicationContext的addBeanFactoryPostProcessors方法添加的。

然后就是自定义的BeanFactoryPostProcessors执行了，主要原理是通过对beanFactory中注册beanDefinition进行遍历，只要是实现了对应接口就会执行。。。

执行时如果实现了BeanDefinitionRegistryPostProcessor接口，那么会先执行postProcessBeanDefinitionRegistry方法，然后才是BeanFactoryPostProcessors接口中的方法。

### 注册BeanPostProcessors

`this.registerBeanPostProcessors(beanFactory);`

原理与上面的差不多，但是这个只是注册不涉及到调用所以没有对硬编码的处理。

注册原理也是对beanDefinitions进行遍历，将对应类型的的beanName进行注册。

### 容器内事件原理

在refresh方法中`initApplicationEventMulticaster();`，对事件广播器进行初始化。

当调用applicationContext.publishEvent(event)方法时，内部会调用ApplicationEventMulticaster的事件广播方法，遍历所有监听器执行监听方法。

监听器的注册是在refresh方法的`registerListeners();`中，原理也是对BeanDefinitions进行遍历，根据类型匹配进行注册。

### 初始化非延迟单例

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

# `<context:component-scan base-package="com.*">`

spring在解析注解时使用了很多元数据类，可以看这篇博客：<https://blog.csdn.net/f641385712/article/details/88765470>

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
                //根据resource获取MatedataReader
               MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
               if (isCandidateComponent(metadataReader)) {
                   //创建BeanDefiniton对象，ScannedGenericBeanDefinition是AnnotatedBeanDefinition子类
                  ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                  sbd.setResource(resource);
                  sbd.setSource(resource);
                  if (isCandidateComponent(sbd)) {
                     candidates.add(sbd);
                  }
               }
      }
   }
   return candidates;
}
```

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

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
	  //获取刚实例化的bean，即未被属性注入的
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }


   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         // 第一个入口！！！！
         applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         mbd.postProcessed = true;
      }
   }
   ************省略代码********
   return exposedObject;
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



# 参考：

> * spring源码深度解析
>
> * spring揭秘
> * BeanFactoryPostProcessor和BeanPostProcessor：<https://blog.csdn.net/caihaijiang/article/details/35552859>
> * Spring元数据Metadata的使用 ：<https://blog.csdn.net/f641385712/article/details/88765470>
> * ConfigurationClassPostProcessor解析：<https://blog.csdn.net/qq_26000415/article/details/78917682#commentBox>
> * spring-analysis：<https://github.com/seaswalker/spring-analysis>
