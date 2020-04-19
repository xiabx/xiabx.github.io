---
title: Spring事物控制
author: XIA
date: 2019-05-31 09:50:19
categories:
- 后端
tags:
- Spring
---
# 事务的基本知识

## 隔离级别
脏读（读到了未提交的数据），不重复读（一个事务前后两次读的数据不一样，即在两次读的中间有另一个事务进行的update操作），幻读（例如，一个事务读取整个表有多少条数据，读完第一次数据后另一个事务在这个表中insert插入了一条数据，第一个事务再次读取有多少条数据时两次的结过不一样）

Read uncommitted（读未提交）：会出现脏读，不可重复读，幻读。
Read committed（都已提交）：解决脏读（出现不可重复读，幻读）
Repeatable read（重复读）：解决不可重复读（会出现幻读）
Serializable （序列化）：解决幻读`
大多数数据库默认的事务隔离级别是Read committed，比如Sql Server , Oracle。Mysql的默认隔离级别是Repeatable read

## Spring中的事务的传播
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
1. PROPAGATIONREQUIRED:     ServiceB.methodB的事务级别定义为PROPAGATION_REQUIRED, 那么由于执行ServiceA.methodA的时候，ServiceA.methodA已经起了事务，这时调用ServiceB.methodB，ServiceB.methodB看到自己已经运行在ServiceA.methodA的事务内部，就不再起新的事务。而假如ServiceA.methodA运行的时候发现自己没有在事务中，他就会为自己分配一个事务。这样，在ServiceA.methodA或者在ServiceB.methodB内的任何地方出现异常，事务都会被回滚。即使ServiceB.methodB的事务已经被提交，但是ServiceA.methodA在接下来fail要回滚，ServiceB.methodB也要回滚

2. PROPAGATION_SUPPORTS ： 如果当前在事务中，即以事务的形式运行，如果当前不再一个事务中，那么就以非事务的形式运行这就跟平常用的普通非事务的代码只有一点点区别了。

3.  PROPAGATION_MANDATORY ： 必须在一个事务中运行。也就是说，他只能被一个父事务调用。否则，他就要抛出异常。就是说methodB执行的时候methodA必须是以事务执行的，如果methodA没有事务那么methodB就会报错

4. PROPAGATION_REQUIRES_NEW ： 比如我们设计ServiceA.methodA的事务级别为PROPAGATION_REQUIRED，ServiceB.methodB的事务级别为PROPAGATION_REQUIRES_NEW，那么当执行到ServiceB.methodB的时候，ServiceA.methodA所在的事务就会挂起，ServiceB.methodB会起一个新的事务，等待ServiceB.methodB的事务完成以后，他才继续执行。他与PROPAGATION_REQUIRED 的事务区别在于事务的回滚程度了。因为ServiceB.methodB是新起一个事务，那么就是存在两个不同的事务。如果ServiceB.methodB已经提交，那么ServiceA.methodA失败回滚，ServiceB.methodB是不会回滚的。如果ServiceB.methodB失败回滚，如果他抛出的异常被ServiceA.methodA捕获，ServiceA.methodA事务仍然可能提交。

5. PROPAGATION_NOT_SUPPORTED ： 当前不支持事务。比如ServiceA.methodA的事务级别是PROPAGATION_REQUIRED ，而ServiceB.methodB的事务级别是PROPAGATION_NOT_SUPPORTED ，那么当执行到ServiceB.methodB时，ServiceA.methodA的事务挂起，而他以非事务的状态运行完，再继续ServiceA.methodA的事务。

6. PROPAGATION_NEVER ： 不能在事务中运行。假设ServiceA.methodA的事务级别是PROPAGATION_REQUIRED， 而ServiceB.methodB的事务级别是PROPAGATION_NEVER ，那么ServiceB.methodB就要抛出异常了。

7. PROPAGATION_NESTED ： 理解Nested的关键是savepoint。他与PROPAGATION_REQUIRES_NEW的区别是，PROPAGATION_REQUIRES_NEW另起一个事务，将会与他的父事务相互独立，而Nested的事务和他的父事务是相依的，他的提交是要等和他的父事务一块提交的。也就是说，如果父事务最后回滚，他也要回滚的。而Nested事务的好处是他有一个savepoint。

# Spring管理事务的方式

## 三个接口
1.TranscationDefinition： 定义事务的信息，例如：隔离级别，事务的传递 ，超时，是否只读。
有多个实现类，对应的不同的事务管理，例如：DataSourceTransactionManager用来管理Spring JDBC和MyBatis的事务管理。HibernateTransactionManager用来管理Hibernate的事务。

2.PlatformTranscationManager：事务管理器，对事物的提交会滚进行控制的接口

3.TransactionStatus：  反应事务的运行状态，例如 是否提交，有没有保存点

## 四种事务管理方法
场景：存在两个账户进行转账。
```java
**************dao层**************
@Component
public class AccountDao {
    @Autowired
    private JdbcTemplate template;

    public void outMoney(String name,Integer money){
        String sql="update account set money = money - ? where name = ?";
        template.update(sql,money,name);
    }
    public void inMoney(String name,Integer money){
        String sql="update account set money = money + ? where name = ?";
        template.update(sql,money,name);
    }
}
```

1：编程式事务管理
通过spring的事务的模版进行事务的控制
代码侵入性强，不常用

```java
******************javaconfig配置springContext****************
@Configuration
@ComponentScan("com")
@EnableTransactionManagement
public class SpringConfig {
    //配置数据源
    @Bean
    public DataSource dataSource() throws PropertyVetoException {
        ComboPooledDataSource pooledDataSource = new ComboPooledDataSource();
        pooledDataSource.setDriverClass("com.mysql.jdbc.Driver");
        pooledDataSource.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/user");
        pooledDataSource.setUser("root");
        pooledDataSource.setPassword("xbx");
        return pooledDataSource;
    }
    //jdbc模版
    @Bean
    public JdbcTemplate template(DataSource dataSource){
        return new JdbcTemplate(dataSource);
    }
    //配置DataSourceTransactionManager，不管是哪种事务管理方法都是必须的
    @Bean
    public DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource){
        return new DataSourceTransactionManager(dataSource);
    }
    //编程式进行事务的控制，声明事务的模版
    @Bean
    public TransactionTemplate transactionTemplate(DataSourceTransactionManager manager){
        return new TransactionTemplate(manager);
    }
}
************在service中使用编程式事务控制************************
@Component
public class AccountService {
    @Autowired
    private AccountDao dao;
    @Autowired
    private TransactionTemplate template;
    public void transfer(final String outName, final String inName, final Integer money){
        template.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
                dao.outMoney(outName, money);
                dao.inMoney(inName, money);
            }
        });
    }
}
```


2：使用TransactionProxyFactoryBean进行事务管理
```java
***********************javaconfig设置*********************
@Configuration
@ComponentScan("com")
public class SpringConfig {
    //配置数据源
    @Bean
    public DataSource dataSource() throws PropertyVetoException {
        ComboPooledDataSource pooledDataSource = new ComboPooledDataSource();
        pooledDataSource.setDriverClass("com.mysql.jdbc.Driver");
        pooledDataSource.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/user");
        pooledDataSource.setUser("root");
        pooledDataSource.setPassword("xbx");
        return pooledDataSource;
    }
    //jdbc模版
    @Bean
    public JdbcTemplate template(DataSource dataSource){
        return new JdbcTemplate(dataSource);
    }
    //配置DataSourceTransactionManager，不管是哪种事务管理方法都是必须的
    @Bean
    public DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource){
        return new DataSourceTransactionManager(dataSource);
    }
    //定义代理TransactionProxyFactoryBean 进行代理，测试时注入service2要注入这个bean
    @Bean
    @Primary
    public TransactionProxyFactoryBean transactionProxyFactoryBean(DataSourceTransactionManager transactionManager, AccountService2 service2){
        TransactionProxyFactoryBean bean = new TransactionProxyFactoryBean();
        //定义要代理的的对象
        bean.setTarget(service2);
        bean.setTransactionManager(transactionManager);
        //定义事务的规则
        Properties properties = new Properties();
        properties.put("*", "PROPAGATION_REQUIRED");
        bean.setTransactionAttributes(properties);
        return bean;
    }
}
************AccountService2 不需要增加任何代码*********************
//通过TransactionProxyFactoryBean进行声明事务
@Component
public class AccountService2 {
    @Autowired
    private AccountDao dao;
    public void transfer( String outName,  String inName,  Integer money){
                dao.outMoney(outName, money);
                int i=1/0;
                dao.inMoney(inName, money);
    }
}
```
3：通过注解的方式进行控制事务
```java
************************javaConfig*****************************
@Configuration
@ComponentScan("com")
//在javaconfig增加注解@EnableTransactionManagement，并在其中配置DataSourceTransactionManager即可开始注解事务
@EnableTransactionManagement
public class SpringConfig {
    //配置数据源
    @Bean
    public DataSource dataSource() throws PropertyVetoException {
        ComboPooledDataSource pooledDataSource = new ComboPooledDataSource();
        pooledDataSource.setDriverClass("com.mysql.jdbc.Driver");
        pooledDataSource.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/user");
        pooledDataSource.setUser("root");
        pooledDataSource.setPassword("xbx");
        return pooledDataSource;
    }
    //jdbc模版
    @Bean
    public JdbcTemplate template(DataSource dataSource){
        return new JdbcTemplate(dataSource);
    }
    //配置DataSourceTransactionManager，不管是哪种事务管理方法都是必须的
    @Bean
    public DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource){
        return new DataSourceTransactionManager(dataSource);
    }
}
**********************************AccountService3 *******************************
@Component
public class AccountService3 {
    @Autowired
    private AccountDao dao;
    //注解中可以指定事务的类型
    @Transactional(propagation= Propagation.REQUIRED,isolation = Isolation.DEFAULT)
    public void transfer( String outName,  String inName,  Integer money){
                dao.outMoney(outName, money);
                int i=1/0;
                dao.inMoney(inName, money);
    }
}
```
4：通过aspectj进行事务控制
由于在javaconfig无法表述tx的名字空间，所以用xml配置
```xml
********************************xml中配置事务aop**************************************
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context-2.5.xsd
                        http://www.springframework.org/schema/tx
                        http://www.springframework.org/schema/tx/spring-tx-2.5.xsd
                        http://www.springframework.org/schema/aop
                        http://www.springframework.org/schema/aop/spring-aop-2.5.xsd ">
    <context:component-scan base-package="com"></context:component-scan>
    <!--引入javaConfig-->
    <bean class="com.config.SpringConfig"></bean>
    <!--事务通知-->
    <tx:advice transaction-manager="dataSourceTransactionManager" id="transactionInterceptor">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED"/>
        </tx:attributes>
    </tx:advice>
    <!--织入-->
    <aop:config >
        <!--定义切点-->
        <aop:pointcut id="pointcut" expression="execution(* com.service.AccountService4.transfer(..))"></aop:pointcut>
        <!--定义aop通知器-->
        <aop:advisor advice-ref="transactionInterceptor" pointcut-ref="pointcut"></aop:advisor>
    </aop:config>
</beans>
```
```java
***********************javaConfig**************************
@Configuration
@ComponentScan("com")
public class SpringConfig {
    //配置数据源
    @Bean
    public DataSource dataSource() throws PropertyVetoException {
        ComboPooledDataSource pooledDataSource = new ComboPooledDataSource();
        pooledDataSource.setDriverClass("com.mysql.jdbc.Driver");
        pooledDataSource.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/user");
        pooledDataSource.setUser("root");
        pooledDataSource.setPassword("xbx");
        return pooledDataSource;
    }
    //jdbc模版
    @Bean
    public JdbcTemplate template(DataSource dataSource){
        return new JdbcTemplate(dataSource);
    }
    //配置DataSourceTransactionManager，不管是哪种事务管理方法都是必须的
    @Bean
    public DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource){
        return new DataSourceTransactionManager(dataSource);
    }
}
******************************AccountService4不需要别的代码********************************************
//通过aspectj进行事务的配置
@Component
public class AccountService4{
    @Autowired
    private AccountDao dao;
    public void transfer( String outName,  String inName,  Integer money){
                dao.outMoney(outName, money);
                //int i=1/0;
                dao.inMoney(inName, money);
    }
}
```