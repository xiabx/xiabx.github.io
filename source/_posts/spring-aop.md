---
title: spring-aop
author: XIA
categories:
  - null
tags:
  - null
date: 2019-12-13 20:40:43
---

# AOP基础回顾

## 为什么需要aop

软件开发一直在寻找更加高效、更易于维护的方式。

面向对象编程（OOP）可以对业务需求等普通关注点进行很好的抽象和封装，并使之模块化。但是oop也存在一些缺点，比如我们需要对一个系统中加入日志功能或者对业务方法添加权限限制等。如果系统较小或业务方法少时我们可以手动添加，但是当系统规模较大业务方法增多，那这些方法的书写也会造成很大的负担。

oop可以对业务中的关注点进行很好的分解并使之模块化，但是无法更好的处理系统中散落的问题。AOP就是为解决和弥补oop模式造成的不足而出现的。在系统中，日志记录、安全检查、事务管理等系统需求就像一把刀横切在我们组织的各个业务功能模块上。aop将这些系统需求点称为横切关注点。

aop引入了aspect概念，用来以模块化的方式对系统中的横切关注点进行封装。aspect之于aop就像class之于oop。

## aop的实现

aop就像oop一样，是一种概念。他同oop一样存在很多实现，如AspectJ等。

因为aop是寄生在oop语言上的，所以要想aop发挥作用需要将aop组件集成到oop组件中。这个过程就是织入（Weave）。

aop的织入方式主要分为静态和动态两种：

* 静态AOP：当横切关注点使用aspect实现后，会通过特定的编译器将失陷后的aspect编译并植入到系统的静态类中。由于aspect会直接以java字节码的形式编译到java类中，所以静态AOP具有相对较高的性能，但是缺乏灵活性。
* 动态AOP：动态aop中aspect的植入放在了系统运行中，而不是预先编译到系统类中。这样虽然会带来一些性能问题，但是增大了灵活性。在动态aop中aspect和class都是作为系统的一等公民存在的。

## aop相关概念

### Joinpoit

在系统运行前需要将aop的功能模块织入到oop的功能模块中。而在系统中将要进行织入的执行点就被称为Joinpoint。

![image-20191213215255529](https://xbxblog2.bj.bcebos.com/springaop/image-20191213215255529.png)

如图中这个程序执行流程，圆圈处就代表Joinpoint。就表示这个点可以执行织入。这种Joinpoint类型称为方法调用。除此之外还有方法调用执行（感觉没啥用），构造方法调用，字段设置，字段获取等。

Jointpoint只是表示可以进行织入的点，如果需要指定进行织入还需要使用Pointcut来表示。

### Pointcut

Pointcut用来指定在系统中哪些Joinpoint织入横切逻辑。如上图中`helloBean.helloMethod（）`方法，如果需要在此Jointpoint上进行织入，就需要使用Pointcut指定这个Joinpoint。

声明一个PointCut可以通过直接指定Jointpoint所在方法名称，正则表达式或使用特定的Pointcut表述语言。

### Advice

Advice表示在Pointcut指定的Jointpoint织入的逻辑。如果将aspect比作class，那么advice就相当于method。

Advice分为以下几种形式：

* Before Advice

  在Jointpoint指定位置之前执行的advice类型，如果Before Advice被织入到方法执行类型Jointpoint时，那么这个Before Advice就会优于方法执行而执行。

* After Advice

  表示在连接点之后执行的Advice类型，还可以分为以下三种：

  + After returning Advice：只有连接点处执行流程正常完成后，该类型Advice才会执行。
  + After throwing Advice: 只有连接点执行过程中抛出异常才会执行。
  + After Advice：无论连接点处执行流程正常结束还是抛出异常都会执行。

![image-20191213222331191](https://xbxblog2.bj.bcebos.com/springaop/image-20191213222331191.png)

* Around Advice

  对附加在其上的Jointpoint进行包裹，可以在连接点之前和之后都指定相应的逻辑。

### Aspect

由上面的三个概念可以知道，在业务系统中有很多的Joinpoint，这些Joinpoint都可以被指定织入，但仅仅是可以，究竟要在哪个Joinpoint上进行织入需要使用Pointcut进行指定。有了需要进行织入的连接点就需要被织入的逻辑，这里的被织入的逻辑就是Advice。所以Pointcut和Advice构成了aop主要要素。

通过Aspect可以将Pointcut和Advice进行封装，封装后的aspect就是aop中的一个“类”。

### 织入和织入器

织入就是将Aspect模块化的横切关注点集成到OOP系统中，这个过程就是织入。而完成这个过程的对象就是织入器。

### 目标对象

符合Pointcut所指定的条件，将在织入过程中被织入横切逻辑的对象，称之为目标对象。

# spring aop一世

## Joinpoint

Spring Aop仅提供方法级别的Joinpoint，虽然仅支持方法拦截，但这也可以满足开发中80%的开发需求。Keep It Simple，Stupid。

## Pointcut

spring aop中以Pointcut接口作为所有Pointcut的最顶层抽象。

```java
public interface Pointcut {

	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();

	Pointcut TRUE = TruePointcut.INSTANCE;
}

```

ClassFilter与MethodMatcher分别用于匹配被织入操作对象以及相应的方法。

ClassFilter接口相对简单：matches方法可根据传入的clazz确定是否匹配。如果类型与我们所捕捉的Joinpoint无关时，可使用默认的`ClassFilter TRUE = TrueClassFilter.INSTANCE;`,表示会匹配系统中所有类型。

```java
public interface ClassFilter {

	boolean matches(Class<?> clazz);

	ClassFilter TRUE = TrueClassFilter.INSTANCE;
}
```

MethodMatcher接口，根据isRuntime()方法的返回值，当为false时为StaticMethodMatcher，为true时为DynamicMethodMatcher。二者区别在于，StaticMethodMatcher只会根据`boolean matches(Method method, Class<?> targetClass);`方法进行判断传入方法时候匹配，而DynamicMethodMatcher则会使用`boolean matches(Method method, Class<?> targetClass, Object... args);`来判断传入的方法是否匹配。因为`matches(Method method, Class<?> targetClass)`方法并不会对传入参数进行判断，所以此方法的结果可以缓存起来，故StaticMethodMatcher的性能要比DynamicMethodMatcher要好。

```java
public interface MethodMatcher {

	boolean matches(Method method, Class<?> targetClass);

	boolean isRuntime();

	boolean matches(Method method, Class<?> targetClass, Object... args);

	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}
```

在MethodMatcher类型的基础上，Pointcut可以分为两类StaticMethodMatcherPointcut和DynamicMethodMatcherPointcut。

![image-20191221155014567](https://xbxblog2.bj.bcebos.com/springaop/image-20191221155014567.png)

一些常见的Pointcut：

- NamedMatchMethodPointcut：可以根据指定的方法名对Joinpoint处的方法名称进行匹配。
- AbstractRegexpMethodPointcut：根据正则表达式对Joinpoint处的方法名进行匹配。
- AnnotationMatchingPointcut：根据类或方法上有没有指定的注解，对方法进行匹配。
- ControlFlowPointcut：匹配程序的调用流程，不是对某个方法执行所在的Joinpoint处的单一特征进行匹配。可以指定特定的流程才会被匹配。

当框架提供的Pointcut无法满足需求时，可以自定义Pointcut。只需要根据情况继承StaticMethodMatcherPointcut或DynamicMethodMatcherPointcut重写相应方法即可。

## Advice

Spring AOP加入了AOP Alliance组织。所以spring aop中各种advice类型全部遵守AOP Alliance规定的接口。

![image-20191221162202771](https://xbxblog2.bj.bcebos.com/springaop/image-20191221162202771.png)

spring aop按照advice其自身实例能否在目标对象类的所有实例中共享这一标准，分为per-class类型和per-instance类型。

### per-class类型的Advice

这种类型的Advice是指可以在目标对象类之间共享，这种Advice只提供方法拦截功能，不会为目标对象类保存任何状态或者添加新的特性。

**Before Advice**

Before Advice所实现的横切逻辑将在相应的Joinpoint之前执行。在Before Advice执行完成后程序将在Joinpoint处继续执行，所以Before Advice不会打断程序的执行，但是有必要也可以通过抛出异常的方式来中断程序流程。

```java
public interface MethodBeforeAdvice extends BeforeAdvice {

	void before(Method method, Object[] args, @Nullable Object target) throws Throwable;

}
```

**ThrowsAdvice**

该类型的advice在joinpoint处的方法抛出异常后才会执行。

对应spring aop中org.springframework.aop.ThrowsAdvice接口，该接口没有实现任何方法，但是我们在定义方法时需要遵循如下规则：

`void afterThrowing([Method,args,target], ThrowableSubclass)`

其中[]中的参数可以省略，在同一个ThrowsAdvice可以实现多个afterThrowing方法，框架会通过反射机制调用对应的方法。

**AfterReturningAdvice**

该类型的advice在joinpoint处的方法正常执行后才会执行。

对应于框架中的org.springframework.aop.AfterReturningAdvice接口，afterReturning可以获取方法的返回值，但是无法更改它。

```java
public interface AfterReturningAdvice extends AfterAdvice {

	void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;

}
```

**Around Advice**

spring aop中没有定义Around Advice相应的接口，而是直接使用了AOP Alliance中的接口，org.aopalliance.intercept.MethodInterceptor。该接口定义如下：

```java
public interface MethodInterceptor extends Interceptor {

	Object invoke(MethodInvocation invocation) throws Throwable;

}
```

invoke方法的proceed()方法可以让程序继续沿着调用链传播。

### per-instance类型的Advice

在不修改类代码的前提下, Introduction可以在运行期为类动态地添加一些方法或Field.

## Aspect

在spring aop中使用Advisor代表Aspect，Advisor通常只持有一个Pointcut和一个Advice。而理论上Aspect可以持有多个Pointcut和多个Advice。所以Advisor是一个特殊的Aspect。

Advisor可以分成两个体系：PointcutAdvisor和IntroductionAdvisor

### PointcutAdvisor

大部分的Advisor都是实现自PointcutAdvisor。

![image-20191221195355309](https://xbxblog2.bj.bcebos.com/springaop/image-20191221195355309.png)



* DefaultPointcutAdvisor：最通用的Advisor，可以通过构造方法或setter方法设置相应的Pointcut和Advisor。
* NameMatchMethodPointcutAdvisor：内部持有一个NameMatchMethodPointcut类型实例。
* RegexpMethodPointcutAdvisor：内部持有一个AbstractRegexpMethodPointcut类型实例。
* DefaultBeanFactoryPointcutAdvisor：使用较少。

### IntroductionAdvisor

为封装Introduction而生。

## 织入

### ProxyFactory

ProxyFactory是spring aop中最基本的一个织入器。使用ProxyFactory进行横切逻辑的织入非常简单：

![image-20191221201518439](https://xbxblog2.bj.bcebos.com/springaop/image-20191221201518439.png)

当目标对象有实现接口时会使用jdk动态代理，当以下三种情况之一发生时会使用CGLIB代理：

* 目标类没有实现任何接口
* ProxyFactory的proxyTargetClass为true时
* ProxyFactory的optimize属性为true时

### ProxyFactory的本质

ProxuFactory层次关系：

![image-20191221212907894](https://xbxblog2.bj.bcebos.com/springaop/image-20191221212907894.png)

* ProxyConfig：它定义了5个boolean类型的属性，分别控制在生成代理对象的时候，应该采取哪些行为措施。
  + proxyTargetClass:如果这个属性为true，则ProxyFactory将会使用CGLIB对目标对象进行代理。
  + optimize：该属性主要告知代理对象是否需要采取进一步的优化措施，如代理对象生成后，即使为其添加或者移除相应的Advice，代理对象也可以忽略这种变动。
  + opaque：用于控制生成的代理对象是否可以强制类型转换为Advised
  + exposeProxy：设置该属性可以让框架生成代理对象时，将当前代理对象绑定到ThreadLocal。
  + frozen：如果将该属性设置为true，那么一旦针对代理对象生成的各项信息配置完成，则不允许更改。
* Advised：设置或查询生成代理对象所需要的一些具体信息，如：advisor，targetClass等。
* AdvisedSupport：保存生成代理对象的必要信息与设置。
* ProxyCreatorSupport：内部提供一个快捷访问AopProxyFactory的方式，AopProxyFactory（只有一个实现类DefaultAopProxyFactory）用来根据AdvisedSupport内容生成AopProxy。

ok，到这里剩下的内容就到了另一个类的体系中了。

![image-20191221215344868](https://xbxblog2.bj.bcebos.com/springaop/image-20191221215344868.png)

AopProxy对使用的不同代理机制进行了适度抽象，针对不同的代理实现机制提供相应的AopProxy子类实现。



在ProxyFactory的getProxy()方法中，createAopProxy()会根据AdvisedSupport提供的信息返回相应的AopProxy实现类。

```java
	//ProxyFactory
    public Object getProxy() {
		return createAopProxy().getProxy();
	}

	//ProxyCreatorSupport
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
        //this--》AdvisedSupport
		return getAopProxyFactory().createAopProxy(this);
	}

	//DefaultAopProxyFactory
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}


```

![image-20191221220314771](https://xbxblog2.bj.bcebos.com/springaop/image-20191221220314771.png)



### ProxyFactoryBean

#### 原理

ProxyCreatorSupport有三个子类，之前介绍了最基本的ProxyFactory，现在开始ProxyFactoryBean。

![image-20191221233656947](https://xbxblog2.bj.bcebos.com/springaop/image-20191221233656947.png)

ProxyFactoryBean是一个将spring ioc与spring aop结合起来的织入器。这个织入器的段词应该是：Proxy+FactoryBean。在spring ioc中，当有个类是FactoryBean的子类时，在获取这个类的bean时不是类本身的bean，而是getObject方法的返回值。

ProxyFactoryBean的getObject()方法获取代理对象其实很简单，因为ProxyFactoryBean与ProxyFactory继承自同一父类ProxyCreatorSupport。该父类已经做完了大部分工作。

```java
	public Object getObject() throws BeansException {
		initializeAdvisorChain();
		if (isSingleton()) {
			return getSingletonInstance();
		}
		else {
			if (this.targetName == null) {
				logger.info("Using non-singleton proxies with singleton targets is often undesirable. " +
						"Enable prototype proxies by setting the 'targetName' property.");
			}
			return newPrototypeInstance();
		}
	}
```

当指定生成的代理对象为单例时，返回缓存的代理对象。当不是单例时则创建新的代理对象。

创建新代理对象的过程如同ProxyFactory中的getProxy() 方法。

#### 用法

ProxyFactoryBean在继承了ProxyCreatorSupport属性之外，还添加了几个独有的属性：

+ proxyInterfaces：如果采用接口代理的方式，配置相应的接口类型，这是一个Collection类型实例。
+ interceptorNames：通过该属性可以指定织入到目标对象的Advice、Advisor。是一个Collection类型实例。
+ singleton：设置代理对象是否为单例。

使用时就将ProxyFactoryBean当作一个bean，配置在xml中，设置必要属性即可。

![image-20191221235757568](https://xbxblog2.bj.bcebos.com/springaop/image-20191221235757568.png)

### 自动织入

为了解决spring aop配置代理时的繁琐问题，spring aop给出了自动织入的解决方法。

#### 原理

Spring AOP自动代理的实现是建立在IOC容器的BeanPostProcessor概念之上的，更确切的说是InstantiationAwareBeanPostProcessor之上。

在IOC中我们知道在实例化对象之前容器会首先检查容器中是否注册有InstantiationAwareBeanPostProcessor类型的BeanPostProcessor。如果有，首先使用相应的InstantiationAwareBeanPostProcessor来构造对象实例。构造成功后直接返回构造完成的对象实例，而不会按照“正规的流程”继续执行。这就是“短路”的用法之一。

![image-20191222171918141](https://xbxblog2.bj.bcebos.com/springaop/image-20191222171918141.png)

#### 用法

这里主要介绍BeanNameAutoProxyCreator与DefaultAdvisorAutoProxyCreator的用法。

**BeanNameAutoProxyCreator**

在容器内声明BeanNameAutoProxyCreator类型的bean，然后为其设置需要代理的beanName与Advice。

![image-20191222172331613](https://xbxblog2.bj.bcebos.com/springaop/image-20191222172331613.png)

**DefaultAdvisorAutoProxyCreator**

DefaultAdvisorAutoProxyCreator比BeanNameAutoProxyCreator的自动化更近了一步。它只要在容器中声明该类型的bean，便可以扫描容器中的Advisor类型的bean，然后自动进行代理。

![image-20191222172530324](https://xbxblog2.bj.bcebos.com/springaop/image-20191222172530324.png)

## TargetSource

在使用ProxyFactory时，我们通常通过setTarget()方法指定具体的目标对象，其实还有另一种方式就是setTarget Source()。

TargetSource的作用就像是目标对象的容器。当针对目标对象的方法调用经历层层拦截到达调用链终点时，通常来说就是要调用目标对象的方法了，但是spring aop会先通过TargetSource取得目标对象，然后再调用目标对象相应的方法。

![image-20191222175827394](https://xbxblog2.bj.bcebos.com/springaop/image-20191222175827394.png)

**可用的TargetSource**

* SingletonTargetSource：使用的最多的。该实现内部只存有一个目标对象。每次调用getTarget都返回同一个对象。用法：![image-20191222180131587](https://xbxblog2.bj.bcebos.com/springaop/image-20191222180131587.png)
* PrototypeTargetSource：每次返回不同的目标对象。![image-20191222180255290](https://xbxblog2.bj.bcebos.com/springaop/image-20191222180255290.png)
* HotSwappableTargetSource    CommonsPoolTargetSource   ThreadLocalTargetSource
* 当框架提供的TargetSource无法满足需求时，可以考虑自定义TargetSource。通过实现TargetSource接口，实现相应方法即可。

# Spring AOP二世

spring框架2.0以后增加了一些新的特性：

+ 直接使用POJO来定义Aspect以及相关的Advice。使用一套标准的注解来标注这些POJO。
+ 简化了XML配置。

虽然2.0以后集成了AspectJ，但是底层的各种概念和实现还是用的1.x中的实现体系。spring aop 只是使用了AspectJ的类库进行Pointcut的解析和匹配。

## @Pointcut

2.0以后spring aop使用AspectJ的pointcut描述语言。

@Pointcut注解使用在@Aspect注解的Aspect定义类中，然后将指定了表达式的注解标注到Aspect类里的某个方法上。

Pointcut声明包含以下几个部分：

+ Pointcut Expression：载体为@Pointcut，该注解是方法级别的注解。@Pointcut所指定的表达式由以下两部分组成：
  + Pointcut标识符：表示Pointcut将以什么样的行为来匹配表达式
  + 表达式匹配模式：Pointcut标识符之内使用的具体匹配模式
+ Pointcut Signature：具体为一个方法的定义，是Pointcut Expression的载体。除了返回值为void没有别的限制。public类型的Pointcut Signature可以在其他Aspect中使用，private类型只能在当前定义的Aspect中使用。可以将Pointcut Signature作为Pointcut Expression的标志符，用来取代重复的表达式定义。![image-20191222233231442](https://xbxblog2.bj.bcebos.com/springaop/image-20191222233231442.png)

### 标识符

**execution**

该标志符可以匹配拥有指定方法签名的Joinpoint。格式如下：![image-20191222233446559](https://xbxblog2.bj.bcebos.com/springaop/image-20191222233446559.png)

其中返回类型、方法签名以及参数部分的匹配模式是必须指定的，其他部分可以省略。

在execution的表达式中可以使用两种通配符：

+ *可以匹配相邻的多个字符，即一个Word。
+ ..只可以在两个位置使用，declaring-type-pattern与方法参数。在declaring-type-pattern表示多个层次类型的声明。在方法参数中表示0到多个参数，参数类型不限。

![image-20191222234221269](https://xbxblog2.bj.bcebos.com/springaop/image-20191222234221269.png)

![image-20191222234316247](https://xbxblog2.bj.bcebos.com/springaop/image-20191222234316247.png)

**within**

只接受类型声明，他将会匹配指定类型下的所有Joinpoint。

`within("cn.spring21.aop.target.MockTest")`将匹配MockTest类里的所有方法。

**this和target**

在spring aop的标志符中，this表示目标对象的代理对象，target表示目标对象。

+ this(ObjectType) ：目标对象代理对象的类型是ObjectType，匹配ObjectType类型中所有Joinpoint
+ target(ObjectType) ：目标对象类型是ObjectType，匹配ObjectType类型中所有Joinpoint

这两个标识符通常和其他标志符一起使用，用来限定前面的标志符范围。如：

`execution(void cn.spring21.*.doSometing(*)) && this(TargetFoo)`

表示cn.spring21这一层下的所有类型的doSomething方法，并且目标对象的代理对象类型为TargetFoo。

**args**

该标志符的作用是捕捉拥有指定参数类型、指定参数数量的方法级Joinpoint，而不管方法在什么类型中被声明。

`args(cn.spring21.domain.User)`

将匹配方法参数类型为User类型的方法。

args标志符是动态Pointcut，如果某方法声明类型不是User类型，但是实际调用时候传入的参数类型是User时也会被捕捉。

**@within**

指定某种类型的注解，只要对象标注了该类型注解，使用@within标志符的Pointcut表达式将匹配该对象内部所有Joinpoint。

`@within(AnyJoinpointAnnotation)`

如果一个类中使用了@AnyJoinpointAnnotation注解，那么该类中所有方法都将被匹配。

**@target**

效果与@within相同，将匹配标注指定注解的类中所有方法。

区别在于@within是静态匹配，@target是动态匹配。

**@args**

该标志符将会检查当前方法级别的Joinpoint的方法参数类型。如果该次传入的参数类型拥有@args所指定的注解，当前Joinpoint将被匹配。

![image-20191223101412524](https://xbxblog2.bj.bcebos.com/springaop/image-20191223101412524.png)

**@annotation**

使用 @annotation标志符的Pointcut表达式，将会尝试检查系统中所有对象的所有方法级别Joinpoint。如果检查的方法标注有@annotation标志符所指定的注解类型，那么当前方法所在的Joinpoint将被Pointcut表达式所匹配。

@annotation使用场景之一就是事务的管理，对需要进行事务管理的方法使用统一注解进行标注，如@Transactional。然后通过`@Annotaion(org.xxx.com.Transactional)`表达式匹配这些使用了@Transactional的方法，然后为其织入事务控制逻辑。

### @Pointcut原理

@Pointcut形式声明的Pointcut表达式，在spring aop内部会通过解析，转化为具体的Pointcut对象。转化后的最终结果还是原来spring aop中pointcut的结构。

![image-20191223105654220](https://xbxblog2.bj.bcebos.com/springaop/image-20191223105654220.png)

ExpressionPointcut与AbstractExpressionPointcut主要是为了以后的扩展性。如果还需要支持除了AspectJ之外的语言，可以在这两个类的基础上进行集成。

![image-20191223110224361](https://xbxblog2.bj.bcebos.com/springaop/image-20191223110224361.png)

## Advice

在使用@Aspect标注的类中，可以使用如下注解来标注一个方法作为Advice。

+ @Before：标注方法为Before Advice
+ @AfterRetuing：标注方法为After Returning Advice
+ @AfterThrowing: 标注方法为ThrowsAdvice类型
+ @After：标注方法为After Advice类型
+ @Around：标注方法为Around Advice

此外还有一个@DeclareParents注解，用来标注Introduction类型的Advice。

### @Before

@Before注解中既可以直接写Pointcut表达式，也可以引用Pointcut Signature。

![image-20191223111756162](https://xbxblog2.bj.bcebos.com/springaop/image-20191223111756162.png)

在此种Advice中，访问Joinpoint处的方法参数，有两种方式。

+ 在Before Advice方法的第一个参数传入JoinPoint类型，通过JoinPoint可以使用getArgs()可以访问相应的参数值。
+ 通过args标志符绑定。![image-20191223112243514](https://xbxblog2.bj.bcebos.com/springaop/image-20191223112243514.png)

### @AfterThrowing

@AfterThrowing有一个独特的属性，throwing。通过它我们可以限定Advice定义方法的参数名，并在方法调用的时候，将相应的异常绑定到具体方法参数上。

如果还需要获取其他信息，我们也可以使用Joinpoint的参数。

![image-20191223113110971](https://xbxblog2.bj.bcebos.com/springaop/image-20191223113110971.png)

### @AfterReturning

该注解也有一个独特的属性returning，通过它可以绑定Pointcut处方法的返回值。

![image-20191223113325443](https://xbxblog2.bj.bcebos.com/springaop/image-20191223113325443.png)

### @After

使用该注解标注的Advice，无论匹配的Joinpoint处的方法是否正确返回都会执行。

![image-20191223113525676](https://xbxblog2.bj.bcebos.com/springaop/image-20191223113525676.png)

### @Around

使用该注解标注的方法第一个参数类型必须为ProceedingJoinPoint。通常情况下，通过ProceedingJoinPoint的Proceed()方法继续调用链的执行。

![image-20191223115739288](https://xbxblog2.bj.bcebos.com/springaop/image-20191223115739288.png)

传入参数的方式同Before Advice。

## Aspect

POJO+@Aspect=Aspect

Aspect中Advice执行顺序问题，分为同一个Aspect中的和不是同一个Aspect中的。

同一个Aspect中的根据声明顺序决定，不是同一个Aspect中可是将标注@Aspect的类实现Order接口，然后重写相应方法即可。

## 织入

织入器ProxyFactory有一个"兄弟"类AspectJProxyFactory，通过该类我们可以实现将Aspect织入到目标对象中。

![image-20191223134246913](https://xbxblog2.bj.bcebos.com/springaop/image-20191223134246913.png)

除了手动进行织入，还可以使用AnnotationAwareAspectJAutoProxyCreator进行自动织入，使用时只需要在ioc容器中注册即可，或使用spring2.0支持的xsd方式进行配置。

直接声明bean的方式：

![image-20191223134512739](https://xbxblog2.bj.bcebos.com/springaop/image-20191223134512739.png)

xsd方式配置：

![image-20191223134527892](https://xbxblog2.bj.bcebos.com/springaop/image-20191223134527892.png)

