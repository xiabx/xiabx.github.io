---
title: Spring-IOC之Bean的实例化
author: XIA
categories:
  - null
tags:
  - null
date: 2020-02-01 15:53:49
---

# Spring IOC 执行流程

Spring IOC启动流程总的可以分为两大步骤，容器启动阶段和容器实例化阶段。

![1569061930193](https://xbxblog.bj.bcebos.com/beanPactory%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.png)

在传统的xml配置中，容器的加载阶段主要是加载配置的xml文件到内存中，对xml文件的结构与定义信息进行分析，将xml中定义的信息注册到一个BeanDefinition对象中。

上图中的其他后处理主要指的是BeanFactoryPostProcessor接口的功能，它在bean实例化之前执行，可以用来进行修改已经定义BeanDefinition信息等操作。BeanFactory需要手动注册并执行BeanFactoryPostProcessor接口，ApplicationContext可以自动注册BeanFactoryPostProcessor注册完成后自动执行相关方法。

而Bean实例化阶段主要是指IOC容器对象调用了getBean方法后执行的对Bean的实例化过程。

## BeanFactoryPostProcessor接口

BeanFactoryPostProcessor是spring的一种容器扩展机制。它允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做相应的修改。这就相当于在容器实现的第一阶段最后加入一道工序，让我们对最终的BeanDefinition做一些额外的操作，比如修改其中bean定义的某些属性，为bean定义增加其他信息等。

如果要自定义实现BeanFactoryPostProcessor，通常我们需要实现org.springframework.
beans.factory.config.BeanFactoryPostProcessor接口。同时，因为一个容器可能拥有多个BeanFactoryPostProcessor，这个时候可能需要实现类同时实现Spring的org.springframework.core.
Ordered接口，以保证各个BeanFactoryPostProcessor可以按照预先设定的顺序执行。

Spring已经提供了几个现成的BeanFactoryPostProcessor实现类，所以，大多时候，我们很少自己去实现某个BeanFactoryPostProcessor。其中org.springframework.beans.factory.config.PropertyPlaceholderConfigurer和org.springframework.beans.factory.config.Property OverrideConfigurer是两个比较常用的BeanFactoryPostProcessor。另外，为处理配置文件中的数据类型与真正的业务对象所定义的数据类型转换，Spring还允许我们通过org.springframework.beans.factory.config.CustomEditorConfigurer来注册自定义的PropertyEditor以补助容器中默认的PropertyEditor。

BeanFactory使用BeanFactoryPostProcessor需要手动注册，而ApplicationContext会自动识别配置文件中的BeanFactoryPostProcessor并应用他。

### PropertyPlaceholderConfigurer

通常我们不想将系统相关的信息同业务相关的配置信息混在到XML配置文件中，这样可能会出现更改xml文件时出现问题。我们可以把系统相关的信息如：数据库连接信息等。配置到一个单独的properties文件中，这样系统信息发生变动时只需要更改properties文件就行了。

使用方法很简单，只需要再配置xml文件时将需要从properties文件中读取的数据项使用占位符`${placeHolder}`占位即可。然后将PropertyPlaceholderConfigurer注册到IOC容器中，就可以实现对XML配置文件中的占位符进行替换的功能。

### PropertyOverrideConfigurer

该类的作用是对xml中定义的bean的属性值进行覆盖。比如在xml中定义的一个bean的属性值a为100，通过该类可以将该属性值更改为200。原理也是通过创建一个properties文件，在文件中以beanName.propertyName形式指定将要进行替换的bean的属性。

使用时将PropertyOverrideConfigurer指定properties文件路径，然后注册到IOC容器中。

### CustomEditorConfigurer

当我们在xml配置了一个属性的值的时候，他在xml中只是一个String类型。就像在pojo类中指定一个变量的类型为FIle类，而在xml文件为其注入时只需要value属性填写文件路径在pojo类中就可以自动转化为File类型。那这是怎么实现的呢？PropertyEditor就提供了这种转化功能，通过实现PropertyEditor接口（通常都是继承PropertyEditorSupport）来实现转化的逻辑，然后将PropertyEditor接口的实现类注册到CustomEditorConfigurer这个后处理器中。然后再将这个后处理器注册到IOC容器中去。

# Bean的生命周期

在BeanFactory中默认是使用延迟加载的，只有ioc容器对象显式调用getBean方法或者对象间依赖进行的隐式调用时bean才开始进行创建。如下图就是bean的生命周期：

![1569079638040](https://xbxblog.bj.bcebos.com/bean%E5%AE%9E%E4%BE%8B%E5%8C%96%E8%BF%87%E7%A8%8B.png)

## Bean的实例化与BeanWrapper

>  容器在内部实现的时候，采用“策略模式”来决定采用何种方式初始化bean实例。通常，可以通过反射或者CGLIB动态字节码生成来初始化相应的bean实例或者动态生成其子类。
>
>  org.springframework.beans.factory.support.InstantiationStrategy定义是实例化策略的抽象接口，其直接子类SimpleInstantiationStrategy实现了简单的对象实例化功能，可以通过反射来实例化对象实例，但不支持方法注入方式的对象实例化。CglibSubclassingInstantiationStrategy继承了SimpleInstantiationStrategy的以反射方式实例化对象的功能，并且通过CGLIB的动态字节码生成功能，该策略实现类可以动态生成某个类的子类，进而满足了方法注入所需的对象实例化需求。默认情况下，容器内部采用的是CglibSubclassingInstantiationStrategy。
>
>  容器只要根据相应bean定义的BeanDefintion取得实例化信息，结合CglibSubclassingInstantiationStrategy以及不同的bean定义类型，就可以返回实例化完成的对象实例。但是，返回方式上有些“点缀”。不是直接返回构造完成的对象实例进行包裹，返回相应的BeanWrapper实例。

BeanWrapper接口通常在Spring框架内部使用，它有一个BeanWrapperImpl的实现类，其作用就是对某个bean进行包裹，然后对这个包裹的bean进行操作，比如设置或者获取bean的相关属性值。在Bean的实例化过程中返回的不是原先的对象实例而是一个BeanWrapper对象，就是为了方便对其进行对象属性的设置。

> BeanWrapper定义继承了org.springframework.beans.PropertyAccessor接口，可以以统一的方式对对象属性进行访问；
>
> BeanWrapper定义同时又直接或者间接继承了PropertyEditorRegistry和TypeConverter接口。
>
> 不知你是否还记得CustomEditorConfigurer？当把各种PropertyEditor注册给容器时，知道后面谁用到这些PropertyEditor吗？对，就是BeanWrapper！
>
> 在第一步构造完成对象之后，Spring会根据对象实例构造一个BeanWrapperImpl实例，然后将之前CustomEditorConfigurer注册的PropertyEditor复制一份给BeanWrapperImpl实例（这就是BeanWrapper同时又是PropertyEditorRegistry的原因）。这样，当BeanWrapper转换类型、设置对象属性值时，就不
> 会无从下手了。

## 各种Aware接口

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

## BeanPostProcessor接口

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
> 接口可以在对象的实例化过程中导致某种类似于电路“短路”的效果。
>
> 实际上，并非所有注册到Spring容器内的bean定义都是按照图4-10的流程实例化的。在所有的步骤之前，也就是实例化bean对象步骤之前，容器会首先检查容器中是否注册有InstantiationAwareBeanPostProcessor类型的BeanPostProcessor。如果有，首先使用相应的InstantiationAwareBeanPostProcessor来构造对象实例。构造成功后直接返回构造完成的对象实例，而不会按照“正规的流程”继续执行。这就是它可能造成“短路”的原因。
>
> 不过，通常情况下都是Spring容器内部使用这种特殊类型的BeanPostProcessor做一些动态对象代理等工作，我们使用普通的BeanPostProcessor实现就可以。这里简单提及一下，目的是让大家有所了解。

spring aop自动代理实现类就是InstantiationAwareBeanPostProcessor类型

## InitializingBean和init-method

在BeanPostProcessor的前置方法调用结束后，接下来如果被实例化的对象实现了InitializingBean接口，那么就会调用afterPropertiesSet方法用为进一步调整实例的状态。

但是使用InitializingBean会显得spring容器具有侵入性，所以一般使用`<bean>`的init-method属性（或`<beans>`的default-init-method属性为所有bean指定默认init方法）为bean指定初始化方法。用来调整实例的状态。由bean的实例化流程图可以看出来该方法的执行位于InitializingBean接口的afterPropertiesSet方法之后。

## DisposableBean与destroy-method方法

与InitializingBean和init-method功能类似，DisposableBean与destroy-method同样是一个是接口一个是`<bean>`的配置属性，他们表示bean的销毁时调用的方法。

但是这个方法并不会自动调用，当在BeanFactory容器中需要调用ConfigurableBeanFactory提供的destroySingletons方法。

在ApplicationContext容器中需要AbstractApplicationContext为我们提供了registerShutdownHook()方法，该方法底层使用标准的Runtime类的addShutdownHook()方式来调用相应bean对象的销毁逻辑，从而保证在Java虚拟机退出之前，这些singtleton类型的bean对象实例的自定义销毁逻辑会被执行。当然AbstractApplicationContext注册的shutdownHook不只是调用对象实例的自定义销毁逻辑，也包括ApplicationContext相关的事件发布等。

同样的道理，在自定义scope之后，使用自定义scope的相关对象实例的销毁逻辑，也应该在合适的时机被调用执行。不过，所有这些规则不包含prototype类型的bean实例，因为prototype对象实例在容器实例化并返回给请求方之后，容器就不再管理这种类型对象实例的生命周期了。

# BeanFactory和FactoryBean

一般情况下，Spring通过反射机制利用`<bean>`的class属性指定实现类实例化Bean，在某些情况下，实例化Bean过程比较复杂，如果按照传统的方式，则需要在`<bean>`中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring为此提供了一个org.springframework.bean.factory.FactoryBean的工厂类接口，用户可以通过实现该接口定制实例化Bean的逻辑。

FactoryBean接口对于Spring框架来说占用重要的地位，Spring自身就提供了70多个FactoryBean的实现。它们隐藏了实例化一些复杂Bean的细节，给上层应用带来了便利。

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

# bean的实例化

BeanFactory对bean的实例化的入口方法就是getBean方法。

## getBean方法

BeanFactory对bean的加载方式为懒加载，只有当BeanFactory对象调用getBean方法时才会对bean进行加载。加载的过程大致分为两种：

1. 如果缓存中存在将要获取的bean就从缓存中返回；
2. 若缓存中没有要返回的bean则需要根据元数据配置新创建一个bean，并将bean加入缓存中。

当使用getBean方法获取一个bean时，将会在内部调用doGetBean方法。当使用包含beanName的getBean方法时都会进入到doGetBean方法中。

```java
	public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
		return doGetBean(name, requiredType, null, false);
	}
```

## doGetBean方法

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
				
                //验证DependsOn也就是在xml文件中bean定义的depends-on属性。
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

### 从缓存中获取singletonBean

对应doGetBean方法的第二步。

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

### 从原始bean中获取最终bean

对应doGetBean方法的第三步。

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

### 获取单例(开始实例化)

对应doGetBean方法的第8步。

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

方法先获取将要创建bean的calss对象。

然后对override属性进行标记，该属性是在解析XML时保存的lookup-method和replace-method的配置，这里解析主要时对该bean将要被覆盖的方法进行预处理，包括校验所要覆盖的方法是否存在，如果存在且只存在一个则增加一个未被重载的标记。

然后执行一个短路拦截操作，通过执行resolveBeforeInstantiation方法，如果容器有注册InstantiationAwareBeanPostProcessor类型的处理器，该处理器时BeanPostProcessor的一个子类，如果处理器返回一个bean，则方法返回，后面的创建bean的操作将会被短路。如果此方法未被短路，则执行doCreateBean方法来创建新的bean。

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

#### 实例化bean

对应上述第二步。

createBeanInstance方法的主要作用就是根据不同的情况使用不同的策略，最后获得一个原始的bean，将其封装到BeanWrapper中。

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

