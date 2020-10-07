---
title: SQL
author: XIA
categories:
  - null
tags:
  - null
date: 2020-10-05 16:32:52
---

# 概述

DBMS：数据库关系管理系统，如mysql、oracle等

RDBMS：关系型数据库管理系统。

----



**SQL分类**

* DDL(Data Definition Language)：用来操作数据库或数据库中表的语句。包含：
  * CREATE：创建数据库和表等对象 
  * DROP： 删除数据库和表等对象 
  * ALTER： 修改数据库和表等对象的结构
* DML(Data Manipulation Language): 用来查询或者变更表中记录的语言。包括：
  * SELECT：查询表中的数据 
  * INSERT：向表中插入新数据 
  * UPDATE：更新表中的数据 
  * DELETE：删除表中的数据
* DCL（Data Control Language）：用来确认或者取消数据库中对数据进行的变更，还可以对RDBMS中的用户权限进行设定。包括：
  * COMMIT： 确认对数据库中的数据进行的变更 
  * ROLLBACK：取消对数据库中的数据进行的变更 
  * GRANT： 赋予用户操作权限 
  * REVOKE： 取消用户的操作权限

---

**SQL书写规范**

* SQL语句要以分号`;`结尾
* SQL语句不区分大小写，SELECT=select=Select
* 常数书写方式是固定的，常用的常数包括字符串、数字、日期。字符串与日期的书写需要使用单引号包裹，数字可以直接书写

---

**表的创建**

数据库的创建：`CREATE DATABASE <数据库名称>;`

表的创建：

```SQL
CREATE TABLE <表名>
（<列名1> <数据类型> <该列所需约束>，
 <列名2> <数据类型> <该列所需约束>，
 <列名3> <数据类型> <该列所需约束>，
 <列名4> <数据类型> <该列所需约束>，
 .
 .
 .
 <该表的约束1>， <该表的约束2>，……）；
```

例如：

```sql
CREATE TABLE Product
(product_id CHAR(4) NOT NULL,
 product_name VARCHAR(100) NOT NULL,
 product_type VARCHAR(32) NOT NULL,
 sale_price INTEGER ,
 purchase_price INTEGER ,
 regist_date DATE ,
 PRIMARY KEY (product_id));
```

---

**表的删除与更新**

删除表：`DROP TABLE <表名>；`

表定义的更新:

* 添加列：`ALTER TABLE <表名> ADD COLUMN <列的定义>；`
* 删除列：`ALTER TABLE <表名> DROP COLUMN <列名>；`

# SELECT语句

select语句包含两个子句：SELECT和FROM子句，SELECT子句列出希望查询到的列、FROM指定了表的名称

在SELECT子句中可以使用AS设定列的别名，如果设置的别名为汉字则需要使用双引号包裹

----

**DISTINCT**

DISTINCT关键字可以去除查询结果中重复的数据：

```sql
SELECT DISTINCT product_type, regist_date FROM Product;
```

则删除查询记录中product_type, regist_date都相同的数据。

DISTINCT只允许放在SELECT关键字之后。

# 聚合与排序

聚合函数：

* COUNT：计算表中的记录数（行数） 
* SUM： 计算表中数值列中数据的合计值 
* AVG： 计算表中数值列中数据的平均值 
* MAX： 求出表中任意列中数据的最大值 
* MIN： 求出表中任意列中数据的最小值

在使用COUNT函数时，如果COUNT(*)则会把NULL也统计进来，如果使用COUNT(列名)统计时则会排除掉NULL值得记录。

COUNT函数中可以使用DISTINCT关键字，如`SELECT DISTINCT COUNT(product_type) FROM Product;`可以统计Product表中包含的不同产品类型的数量。

---

**GROUP BY语句**

GROUP BY语句可以对数据表中数据根据指定的列名进行分组，将指定列中相同的数据归为一组。如：

```sql
SELECT product_type, COUNT(*)
 FROM Product
 GROUP BY product_type;
 
结果如下：
 product_type | count
--------------+------
衣服 | 2
办公用品 | 2
厨房用具 | 4
```

所实现的结果是将Product表中的数据按照product_type进行分组，如图：

![image-20201005171236801](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20201005171236801.png)

当使用where后使用group by后sql执行顺序为：**FROM → WHERE → GROUP BY → SELECT**

**使用group by注意事项**

* 在select子句中使用非group by中的列，会造成合并行问题，想象一下上面那个图~，分成了三部分，每部分只会显示一条，如果不是select中使用分组条件的列，就会只返回改组第一条数据的值
* GROUP BY子句中不能使用SELECT子句中定义的别名。原因参照sql执行顺序
* where子句中不可以使用聚合函数。只有select与having子句中可以使用聚合函数

---

**having为聚合结果指定条件**

having过滤分组后的结果，where过滤行的结果。

> WHERE 子句 = 指定行所对应的条件 
>
> HAVING 子句 = 指定组所对应的条件

having子句中只可以包含聚合函数或group by指定的列。

---

**order by排序**

指定order by的语句指定顺序：

FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

由于select子句在order by子句执行前执行，所以order by可以使用select子句中的别名

# 数据更新

**INSERT**

语法格式：

`INSERT INTO <表名> (列1, 列2, 列3, ……) VALUES (值1, 值2, 值3, ……);`

如果对表的全列进行插入，则可以省略列名。

如果需要插入NULL值，只需要在指定值处直接使用NULL。如果列有默认值，则可以使用DEFAULT来插入默认值。

如果需要用一个表复制数据到另一个表可以使用**INSERT...SELECT**语句。

`INSERT INTO <表名1> (列1, 列2, 列3, ……) SELECT (列1, 列2, 列3, ……) FROM <表名2>;`

---

**DELETE**

删除表：  DROP TABLE

DELETE删除表中数据

```sql
DELETE FROM <表名> WHERE <条件>;
```

----

**UPDATE**

向表中插入数据:`UPDATE <表名> SET <列名1> = <表达式>,<列名2> = <表达式> WHERE <条件>;`

---

**事务**

事务就是需要在同一个处理单元中执行的一系列更新处理的集合。

```sql
事务开始语句;
 DML语句①;
 DML语句②;
 DML语句③;
 . . .
事务结束语句（COMMIT或者ROLLBACK）;
```

不同数据库的事务开始语句不同：

* SQL Server、PostgreSQL 
  * BEGIN TRANSACTION
* MySQL 
  * START TRANSACTION 
* Oracle、DB2 
  * 无

事务结束语句：

* COMMIT：提交事务所变化的数据到数据库
* ROLLBACK：回滚事务

事务的性质：

* 原子性：要么全部执行要么都不执行
* 一致性：事务更新的数据要遵守数据库的约束条件，如非空等
* 隔离性：事务之间的操作互相不影响
* 持久性：事务结束后，事务的数据可以持久的保存，例如将事务的执行记录保存到硬盘日志中

# 视图

视图保存的是 SELECT 语句。 我们从视图中读取数据时，视图会在内部执行该 SELECT 语句并创建出 一张临时表。

创建视图使用CREATE VIEW语句：

```sql
CREATE VIEW 视图名称(<视图列名1>, <视图列名2>, ……)
AS
<SELECT语句>
```

使用视图注意事项：

* 创建视图时不可以使用order by语句
* 对视图数据进行更新需要满足以下条件
  * SELECT 子句中未使用 DISTINCT 
  * FROM 子句中只有一张表 
  * 未使用 GROUP BY 子句 
  * 未使用 HAVING 子句

删除视图使用DROP VIEW：

```sql
DROP VIEW 视图名称(<视图列名1>, <视图列名2>, ……)
```

# 子查询

子查询就是将用来定义视图的SELECT语句直接用于FROM子句当中。如：

```sql
SELECT product_type, cnt_product
   FROM (
   SELECT Product_type, COUNT(*) AS cnt_product
   FROM Product
   GROUP BY product_type
 ) AS ProductSum;
```

子查询必须设定别名。

## 标量子查询

子查询的结果只返回一个值，不像放在from语句中的会返回多行数据可以当做表用。如：

```sql
 SELECT product_id, product_name, sale_price
 FROM Product
 WHERE sale_price > (SELECT AVG(sale_price)
 FROM Product);
```

标量子查询可以应用于：SELECT 子句、GROUP BY 子句、HAVING 子句，ORDER BY 中。

## 关联子查询

在细分的组内进行比较时，需要使用关联子查询。通俗讲就是子查询中需要使用外查询的条件进行限制，子查询关联外查询。例如：当需要查出产品表中大于该产品对应类型平均价格的产品，此时就需要使用关联子查询。

```sql
 SELECT product_type, product_name, sale_price
 FROM Product AS P1 
 WHERE sale_price > (SELECT AVG(sale_price)
                       FROM Product AS P2 
                       WHERE P1.product_type = P2.product_type
                       GROUP BY product_type);
```

小结：

* 关联子查询可以想象为对集合进行切分，类似于group by的作用
* 结合条件必须写在子查询中，原因在于子查询中定义的表的表名作用域只在子查询中

# CASE表达式

可以用于select子句中，类似于java中switch的作用。可以根据选择到的数据的值进行不同分支的计算。

```sql
CASE WHEN <求值表达式> THEN <表达式>
 WHEN <求值表达式> THEN <表达式>
 WHEN <求值表达式> THEN <表达式>
 . . .
 ELSE <表达式>
END
```

如：

```sql
SELECT product_name,
 CASE WHEN product_type = '衣服'
 THEN 'A ：' | | product_type
 WHEN product_type = '办公用品'
 THEN 'B ：' | | product_type
 WHEN product_type = '厨房用具'
 THEN 'C ：' | | product_type
 ELSE NULL
 END AS abc_product_type
 FROM Product;
```

# 集合运算

## 表的加法——UNION

将两个表的数据行进行加法合并，例如两个Product表，进行union合并，结果如下图。

union操作会合并重复的行。

```sql
SELECT product_id, product_name
 FROM Product
UNION
SELECT product_id, product_name
 FROM Product2;
```



![image-20201006234925281](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20201006234925281.png)

## 不合并重复行的加法--ALL

也就是合并后的重复行会显示两次。

```sql
SELECT product_id, product_name
 FROM Product
UNION ALL
SELECT product_id, product_name
 FROM Product2;
```

## 选取表中公共部分——INTERSECT

```sql
SELECT product_id, product_name
 FROM Product
INTERSECT
SELECT product_id, product_name
 FROM Product2
ORDER BY product_id;
```



![image-20201006235311231](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20201006235311231.png)

## 记录的减法——EXCEPT

```sql
SELECT product_id, product_name
 FROM Product
EXCEPT
SELECT product_id, product_name
 FROM Product2
ORDER BY product_id;
```

![image-20201006235420308](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20201006235420308.png)

