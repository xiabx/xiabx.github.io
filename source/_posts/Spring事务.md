---
title: Spring事务
author: XIA
categories:
  - null
tags:
  - null
date: 2020-01-01 18:12:18
---

# Spring事务管理概述

JDBC的事务控制是由同一个Connection完成的，所以在两个DAO方法中要保证使用同一个Connection对象，spring将这个要使用的connection对象保存到ThreadLocal中。

比如在JdbcTemplate中获取Connection是通过DataSourceUtils这个工具类获取的。该工具类会在ThreadLocal中获取与当前线程绑定的Connection对象，这样就保证了在使用JdbcTemplate时执行的每个方法使用的connection对象都是同一个。

![image-20200101203737781](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200101203737781.png)

PlateformTranscationManager为事务控制的中心，负责界定事务边界。TransactionDefinition负责定义事务相关属性，包括隔离级别、传播行为等。事务开启后TranscationStatus负责事务的状态，我们可以通过TranscationStatus对事务进行有限的控制。

# TransactionDefinition

该类主要定义事务的属性，包括隔离级别、传播行为、超时时间、是否只读。

## 属性介绍

#### 隔离级别

脏读（读到了未提交的数据），不重复读（一个事务前后两次读的数据不一样，即在两次读的中间有另一个事务进行的update操作），幻读（例如，一个事务读取整个表有多少条数据，读完第一次数据后另一个事务在这个表中insert插入了一条数据，第一个事务再次读取有多少条数据时两次的结过不一样）

Read uncommitted（读未提交）：会出现脏读，不可重复读，幻读。
Read committed（读已提交）：解决脏读（出现不可重复读，幻读）
Repeatable read（重复读）：解决不可重复读（会出现幻读）
Serializable （序列化）：解决幻读`
大多数数据库默认的事务隔离级别是Read committed，比如Sql Server , Oracle。Mysql的默认隔离级别是Repeatable read

#### 事务的传播

事务的传播主要用在一个service层的方法调用另一个service层的方法时，两个方法的事务之间的关系。例如当methodA存在事务而methodB没有事务时methodB该以哪种方式运行的这类问题。

```java
ServiceA {   
	void methodA() {   
         ServiceB.methodB();   
    }   
}  

ServiceB {   
     void methodB() {   
     }   
} 
```

属性及含义：

1. PROPAGATION_REQUIRED:     ServiceB.methodB的事务级别定义为PROPAGATION_REQUIRED, 那么由于执行ServiceA.methodA的时候，ServiceA.methodA已经起了事务，这时调用ServiceB.methodB，ServiceB.methodB看到自己已经运行在ServiceA.methodA的事务内部，就不再起新的事务。而假如ServiceA.methodA运行的时候发现自己没有在事务中，他就会为自己分配一个事务。这样，在ServiceA.methodA或者在ServiceB.methodB内的任何地方出现异常，事务都会被回滚。即使ServiceB.methodB的事务已经被提交，但是ServiceA.methodA在接下来fail要回滚，ServiceB.methodB也要回滚

2. PROPAGATION_SUPPORTS ： 如果当前在事务中，即以事务的形式运行，如果当前不再一个事务中，那么就以非事务的形式运行这就跟平常用的普通非事务的代码只有一点点区别了。

3. PROPAGATION_MANDATORY ： 必须在一个事务中运行。也就是说，他只能被一个父事务调用。否则，他就要抛出异常。就是说methodB执行的时候methodA必须是以事务执行的，如果methodA没有事务那么methodB就会报错

4. PROPAGATION_REQUIRES_NEW ： 比如我们设计ServiceA.methodA的事务级别为PROPAGATION_REQUIRED，ServiceB.methodB的事务级别为PROPAGATION_REQUIRES_NEW，那么当执行到ServiceB.methodB的时候，ServiceA.methodA所在的事务就会挂起，ServiceB.methodB会起一个新的事务，等待ServiceB.methodB的事务完成以后，他才继续执行。他与PROPAGATION_REQUIRED 的事务区别在于事务的回滚程度了。因为ServiceB.methodB是新起一个事务，那么就是存在两个不同的事务。如果ServiceB.methodB已经提交，那么ServiceA.methodA失败回滚，ServiceB.methodB是不会回滚的。如果ServiceB.methodB失败回滚，如果他抛出的异常被ServiceA.methodA捕获，ServiceA.methodA事务仍然可能提交。

5. PROPAGATION_NOT_SUPPORTED ： 当前不支持事务。比如ServiceA.methodA的事务级别是PROPAGATION_REQUIRED ，而ServiceB.methodB的事务级别是PROPAGATION_NOT_SUPPORTED ，那么当执行到ServiceB.methodB时，ServiceA.methodA的事务挂起，而他以非事务的状态运行完，再继续ServiceA.methodA的事务。

6. PROPAGATION_NEVER ： 不能在事务中运行。假设ServiceA.methodA的事务级别是PROPAGATION_REQUIRED， 而ServiceB.methodB的事务级别是PROPAGATION_NEVER ，那么ServiceB.methodB就要抛出异常了。

7. PROPAGATION_NESTED ： 他与PROPAGATION_REQUIRES_NEW的区别是，PROPAGATION_REQUIRES_NEW另起一个事务，将会与他的父事务相互独立，而Nested的事务和他的父事务是相依的，他的提交是要等和他的父事务一块提交的。也就是说，如果父事务最后回滚，他也要回滚的。而Nested事务的好处是他有一个savepoint。

## 相关实现

TransactionDefinition是一个接口，要为PlatformTransactionManager创建事务提供信息，需要有相应的实现类提供支持。TransactionDefinition实现类主要分为两类，一类是按照编程式事务场景，一类是声明式事务场景。

![、](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200101212754237.png)

**编程式事务**

DefaultTransactionDefinition是TransactionDefinition接口的默认实现类，他提供了各事务属性的默认值，并且通过setter方法可以更改这些默认值。

TransactionTemplate是spring提供的编程式事务管理的模板类，他直接继承了DefaultTransactionDefinition，所以我们使用TransactionTemplate时可以通过TransactionTemplate本身控制事务属性。

**声明式事务**

TransactionAttribute接口继承自TransactionDefinition接口，主要面向使用spring aop进行声明式事务管理的场合。他在TransactionDefinition定义的基础上添加了一个`boolean rollbackOn(Throwable ex);`方法，用来指定业务方法抛出哪些异常时可以回滚事务。

DefaultTransactionAttribute是TransactionAttribute接口的默认实现，他同时继承了DefaultTransactionDefinition，增加了roolbackOn实现，当异常为RuntimeException或Error时，将回滚事务。

RuleBasedTransactionAttribute根据传入的回滚规则来决定是否进行事务回滚。

# TransactionStatus

TransactionStatus接口表示整个事务处理过程中的事务状态，在编程式事务中较多地使用使用该接口。

在事务处理过程中TransactionStatus进行如下工作。

+ 使用相应的方法查询事务状态
+ 通过设置RollbackOnly()方法标记当前事务以使其回滚
+ 当PlatformTransactionManager支持savepoint，可以通过TransactionsStatus在当前事务中创建内部嵌套事务

TransactionStatus结构层次：

![image-20200101221758484](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200101221758484.png)

SavepointManager是在JDBC3.0基础上，对savepoint的支持提供的抽象。通过继承Savepoint，TransactionStatus获得管理savepoint的能力，从而支持创建内部嵌套事务。

TransactionExecution提供查询事务状态的方法和设置RollbackOnly的方法。

AbstractTransactionStatus是TransactionStatus的抽象实现，主要为其子类提供公共的方法实现。它下面有两个实现类，DefaultTransactionStatus和SimpleTransactionStatus。其中DefaultTransactionStatus是spring事务框架内使用的主要实现，SimpleTransactionStatus在框架内部没有用到过，主要用于测试使用。

# PlatformTransactionManager

PlatformTransactionManager是spring事务框架的核心组件，它的整个抽象体系是基于策略模式的，同时事务框架的实现更多的也依赖于模板方法模式。由PlatformTransactionManager对事务界定进行统一抽象，而具体的界定策略的实现交由具体的实现类。

PlatformTransactionManager的实现类可以划分为面向局部事务和面向全局事务两个分支。

**面向局部事务的实现类**

spring为各种数据访问技术提供了现成的PlatformTransactionManager实现类支持。

![image-20200102204458354](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200102204458354.png)

**面向全局的事务实现类**

略。。。

## 分析DataSourceTransactionManager

PlatformTransactionManager的各个实现类基本上遵循统一的结构和理念，所以从分析DataSourceTransactionManager入手，通过分析DataSourceTransactionManager就可以类推到其他PlatformTransactionManager实现类逻辑。

DataSourceTransactionManager层次结构图：

![image-20200102211412947](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200102211412947.png)

AbstractPlatformTransactionManager以模板方法的形式封装了固定的事务处理逻辑，而只将与事务资源有关的操作以protect或者abstract方法的形式留给DataSourceTransactionManager来实现。

作为模板方法父类，AbstractPlatformTransactionManager替各子类实现了以下固定的事务内部处理逻辑：

+ 判断是否存在当前事务，然后根据判断结果执行不同的处理逻辑
+ 结合是否存在当前事务的情况，根据TransactionDefinition中指定的传播行为的不同语义执行后继逻辑
+ 根据情况挂起或者恢复事务
+ 提交之前检查readOnly字段是否被设置，如果是的话，以事务的回滚代替事务的提交
+ 在事务回滚的情况下，清理并恢复事务状态
+ 如果事务的Synchonization处于active状态，在事务处理的规定时点触发注册的Synchonization回调接口

这些固定的事务内部处理逻辑位于以下几个主要的模板方法中：

![image-20200102212838583](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200102212838583.png)

**getTransaction(TransactionDefinition)**

该方法的主要目的是开启一个事务，但需要在此之前判断以下是否存在一个事务，如果存在则根据TransactionDefnition定义的传播行为的具体语义来决定如何处理。

基本的执行流程如下：

1. 获取transaction object 。该对象会根据实现类的不同而不同，AbstractPlatformTransactionManager也不需要知道该对象的具体类型。
2. 检查TransactionDefinition参数的合法性，如果为空，则创建一个DefaultTransactionDefinition实例以提供默认事务定义数据
3. 根据transaction object 判断是否存在当前事务，根据判定结果采取不同的处理方式。如果判断为true，即存在当前事务，此时就开始处理事务的传播规则。若为false，即不存在当前事务，则开启一个新的事务。

suspend和resume这两个方法会把保存在ThreadLocal中的信息释放和重新保存。

# 编程式事务管理

通过spring进行编程式事务管理有两种方式：直接使用PlatformTransactionManager和使用更方便的TransactionTemplate。

## 直接使用PlatformTransactionManager

直接使用PlatformTransactionManager进行事务管理的代码清单如下：

![image-20200103093742032](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200103093742032.png)

使用合适的PlatformTransactionManager实现类，然后结合TransactionDefinition开启事务，并结合TransactionStatus来回滚或者提交事务，就完成了整个事务管理。

不过这么做存在会有很多重复的代码，这时可以使用TransactionTemplate将PlatformTransactionManager直接管理事务的代码进行封装，以便更方便的管理事务。

## 使用TransactionTemplate

TransactionTemplate对PlatformTransactionManager相关的事务界定操作与相关的异常处理进行了模板化封装。开发人员可以通过callback接口提供具体的事务界定内容。

spring提供了两个callback接口，TransactionCallback和TransactionCallbackWithoutResult，二者的区别在于是否需要返回值。

![image-20200103094637663](https://xbxblog2.bj.bcebos.com/springTransaction%2Fimage-20200103094637663.png)

# 声明式事务管理

声明式事务的原理是将事务控制与aop结合起来。通过定义一个环绕通知，在invoke中进行事务代码编程，根据异常信息决定事务提交还是回滚。

这样需要确定两件事：

1. 怎么确定目标对象中的方法需要事务支持。我们通过将这些信息配置在xml文件或在源代码中注解。
2. 调用方法过程中抛出异常，如何对这些异常进行处理，哪些异常的情况下需要回滚事务。解决这个问题是使用TransactionAttribute实现类代替默认的TransactionDefinition，TransactionAttribute可以设置执行回滚的异常。

spring通过TransactionInterceptor来对声明式事务进行管理。







