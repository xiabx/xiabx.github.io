---
title: Spring-JDBC
author: XIA
date: 2019-12-25 19:30:59
tags:
---

# 统一的数据异常处理体系

首先我们从DAO模式开始讲起，DAO=Data Access Object。使用DAO模式可以将完全分离数据的访问和存储，屏蔽数据访问方式的差异性。

但是DAO模式在对数据的访问和存储进行分离的时候存在一些问题，由于DAO层可能会使用各种不同的技术，每种技术可能会产生不同的异常信息，这些异常往往是checked exception类型。那么这些异常是直接抛出去还是在内部自己处理掉呢？

如果直接抛出去会有两种麻烦，第一由于DAO层使用的技术选择性很多，可能面临更换技术的情况，比如由JDBC改为了使用内存型数据库的情况，那么原先抛出去的JDBC类型的异常的处理已经耦合到Service层中，不好处理。第二个问题就是由于之前Service层只是处理了JDBC类型的异常，若引入了别的异常则需要再改动Service层代码，让其可以处理新的异常类型。

若直接在DAO层处理了异常信息，那么对于Service来说就不知道数据访问期间发生了什么问题。

处理这个问题的方式就是将特定的数据访问异常进行封装，封装后的异常类型选择unchecked exception。这是因为数据库的异常通常无法在Service层进行处理，如链接失败、无法取得相应资源等。而采用unchecked exception后DAO层接口也不会声明式抛出异常信息。

以Runtime Exception形式的特定数据访问异常转换后抛出，虽然解决了统一数据访问接口的问题，但是还是存在些问题。各个数据库对错误信息的表达方式不同，有的数据库通过ErrorCode，有的数据库通过SqlState。所以在抛出Runtime Exception时需要对异常信息进行相应处理后再抛出。

既然到这里我们已经对异常信息进行了分析，那么我们可以将特定的异常信息进行归类，通过定义相应的Runtime Exception子类型。然后根据错误类型的不同抛出不同的运行时异常的子类型。

在org.springframework.dao包下存在spring为我们提前定义好的一些类。他们均以DataAccessException为父类。

# JdbcTemplate

## JDBC的尴尬

JDBC在使用中会存在一些问题，就是jdbc的设计主要是面向较为底层的数据库操作，所以在实际使用过程中会存在一个小小的查询或更新就要写一大段雷同的代码，获取Connection、获取Statement、关闭资源等等。而且在书写这些重复的代码时也会增加出错的风险。

## JDBCTemplate的诞生

JdbcTemplate主要做两件事情，第一就是封装所有基于JDBC的数据访问代码，以统一的格式和规范来使用JDBC API。第二就是对SQLException所提供的异常信息在框架内进行统一转译，将JDBC的异常体系转换为Spring提供的异常体系。

## JDBCTemplate的实现

JDBCTemplate主要就是使用了模板方法的设计模式，但是标准的模板方法的设计模式需要每次使用时都创建JdbcTemplate的子类，未免过于繁琐，所以现在的JdbcTemplate的方式改为了使用回调函数的方式。

实际的jdbcTemplate直接继承自JdbcAccessor抽象类，实现JdbcOperations接口。

![image-20191226202849699](https://xbxblog2.bj.bcebos.com/springJdbc/image-20191226202849699.png)

JdbcOperations定义了JdbcTemplate可以使用的JDBC操作集合，从增删改查到存储过程调用无所不包。JdbcAccessor则提供了一些公用的属性：表示数据源的DataSource和用于对SQLException转译的SQLExceptionTranslator。

JdbcTemplate中各种模板方法按照其通过相应Callback接口所公开的API自由程度的大小，可以分为如下4组：

+ 面向Connection的模板方法：通过ConnectionCallback回调接口所公开的Connection进行数据访问。
+ 面向Statement的模板方法：通过StatementCallback回调接口对外公开的Statement进行操作。
+ 面向PreparedStatement的模板方法：这种模板方法会先通过PreparedStatementCreator创建PreparedStatement，然后通过PreparedStatementCallback接口对PreparedStatement进行操作。
+ 面向CallableStatement的模板方法：用于对存储过程的访问，先通过CallableStatementCreator创建用于调用存储过程的CallableStatement，然后再通过CallableStatementCallback来对创建的CallableStatement进行操作。

**几个要注意的地方**

+ 在JdbcTemplate中获取Connection时使用了`Connection con = DataSourceUtils.getConnection(obtainDataSource())`,通过DataSourceUtils获取的Connection会将取得的Connecton绑定到当前线程，以便在使用Spring提供的统一事务抽象层进行事务管理的时候使用。

+ 控制JdbcTemplate的行为，在通过Statement或PreparedStatement进行数据库操作之前，会调用`applyStatementSettings(stmt);`这个代码，通过查看applyStatementSettings方法的内容我们可以知道，可以通过JdbcTemplate对象设置fetchSize、maxRows、queryTimeout等参数。

**异常转译**

在JdbcTemplate中通过父类JdbcAccessor中定义的SQLExceptionTranslator进行对异常的转译，该接口的定义如下：

```java
public interface SQLExceptionTranslator {
    
	DataAccessException translate(String task, @Nullable String sql, SQLException ex);
}
```

类的继承关系与实现类如下：

![image-20191226223903966](https://xbxblog2.bj.bcebos.com/springJdbc%2Fimage-20191226223903966.png)

+ SQLExceptionSubclassTranslator主要是用来将JDBC4中定义的异常体系转化为spring的数据访问异常体系。
+ SQLErrorCodeExceptionTranslator会基于SQLException所返回的ErrorCode进行异常转译。
+ SQLStateSQLExceptionTranslator会根据SQLException.getSQLState()所返回的信息进行异常转译。

SQLErrorCodeExceptionTranslator的异常转译流程如下：

1. 首先会检查customTranslate方法是否可以对当前传入的SQLException进行转译。如果可以则返回DataAccessException类型异常，如果不可以则返回null。
2. 使用SQLExceptionSubclassTranslator进行异常转译。
3. 使用SQLErrorCodesFactory所加载的SQLErrorCodes进行异常转译。其中SQLErrorCodesFactory加载SQLErrorCodes的流程为：先加载spring发布的jar包下的sql-error-codes.xml文件，如果发现classpath的根路径下也存着sql-error-codes.xml文件则加载该文件内容并覆盖默认ErrorCodes定义。
4. 如果SQLErrorCode搞定不了的话则委托给SQLStateSQLExceptionTranslator进行处理。

如上如果我们需要插入异常转译逻辑，有两个点：

1. 在流程第一步会先检查customTranslate方法，这里可以通过继承SQLErrorCodeExceptionTranslator类，重写此方法。并将子类设置到JdbcTemplate中。
2. 在流程的第三步，会通过classpath下的根路径的sql-error-codes.xml进行覆盖默认定义，所以这里可以通过配置这个xml文件来插入逻辑。

# JdbcTemplate的使用

## 初始化JdbcTemplate

初始化JdbcTemplate其实比较容易，只需要设置DataSource即可。

## 查询数据

JdbcTemplate针对数据查询提供了多个重载方法，我们可以根据自己的需求选择不同的模板方法。

![image-20191231211133395](https://xbxblog2.bj.bcebos.com/springJdbc%2Fimage-20191231211133395.png)

如果以上接口无法满足需求时可以使用相应的Callback接口对查询结果进行定制。

+ ResultSetExtractor：该接口定义如下，我们可以对结果进行任意形式的包装后返回

  ```java
  public interface ResultSetExtractor<T> {
      
  	@Nullable
  	T extractData(ResultSet rs) throws SQLException, DataAccessException;
  }
  ```

+ RowCallbackHandler：相对于ResultSetExtractor来说只关注于单行处理结果，处理后的结果可以根据需求放在当前RowCallbackHandler对象中或JdbcTemplate的程序上下文中。

  这个回调接口的背后是使用RowCallbackHandlerResultSetExtractor对其封装的，RowCallbackHandlerResultSetExtractor是ResultSetExtractor的一个实现类。

  该接口定义如下：

  ```java
  public interface RowCallbackHandler {
  
  	void processRow(ResultSet rs) throws SQLException;
  }
  ```

+ RowMapper：ResultSetExtractor的精简版，功能类似于RowCallbackHandler，只关注单行的结果处理。

  其实使用这个回调接口的背后也是ResultSetExtractor的支持，当我们使用这个回调接口是，spring内部会使用RowMapperResultSetExtractor将其封装。

  接口定义如下：

  ```java
  public interface RowMapper<T> {
  
  	@Nullable
  	T mapRow(ResultSet rs, int rowNum) throws SQLException;
  }
  ```

## 更新数据

无论是对数据库的增删改都可以使用JdbcTemplate重载的一组update()方法进行。

![image-20191231213922281](https://xbxblog2.bj.bcebos.com/springJdbc%2Fimage-20191231213922281.png)

如果我们相对更新方法有更多的控制权可以使用PreparedStatementCreator或PreparedStatementSetter回调接口。

批量更新的话可以使用`int[] batchUpdate(final String... sql) `或`int[] batchUpdate(String sql, final BatchPreparedStatementSetter pss)`方法。

## 其他操作

**调用存储过程**

因为存储过程的调用过程也是可以进行模板化处理的，所以这里也可以像jdbc的增删改查操作似的进行封装。

**递增主键策略的抽象**

数据库的逐渐自增可以放在数据库服务器上，也可以放在应用程序中。通常使用第二种，因为有更好的性能。根据数据库对递增主键的生成支持，spring提供了两种方式，一种是像mysql这类的不支持sequence的，一种是支持squence的数据库，例如oracle。

**NamedParameterJdbcTemplate**

可以使用`:name`这种预占符号对原来sql中的？进行替换。内部封装了一个JdbcTemplate对象来实现基本操作。

# Spring中的DataSource

自JDBC2.0以后，jdbc获取connection的操作都是用dataSource进行。

## 简单的DataSource

这类datasource只提供获取connection的基本功能。可以作为开发或测试使用，不可以作为生产使用。

+ DriverManagerDataSource：基于最基本的DriverManager获取connection，当每次请求一个数据库时都会返回新的连接。
+ SingleConnectionDataSource：每次请求数据库都会返回同一个connection。

## 拥有连接池的DataSource

这类DataSource除了可以获取connection的基本功能之外。还提供缓冲池对连接进行管理。这类DataSource实现的代表：DBCP，C3P0等。

## 支持分布式事务的DataSource

这一类DataSource实现类，应该是XADataSource的实现类。

# JdbcDaoSupport

当我们使用JdbcTemplate实现DAO层时，通常需要JdbcTemplate和DataSource对象。DAO层可以直接继承JdbcDaoSupport抽象类来避免每次都声明这两个对象。





