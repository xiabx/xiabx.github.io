---
title: SpringBoot自动装配
author: XIA
categories:
  - null
tags:
  - null
date: 2020-02-04 15:48:32
---

# 注解驱动发展史

+ Spring Framework 1.x：由于java5的发布使spring也开始支持注解驱动，这时支持@Transaction等注解，但XML仍是配置方式的唯一选择。
+ Spring Framework 2.x：在此版本中，spring开始支持@Autowired  @Qualifier等注解。但是并没有完全替代xml配置驱动，因为这些注解的入口还得需要在xml中配置`<context:component-scan>`等元素。
+ Spring Framework 3.x：在此版本中spring使用@ImportResource和@Import注解结合AnnotationApplicationContext引导类来摆脱xml配置驱动。引入@ComponentScan。引入`@Enable 模块驱动`。
+ Spring Framework 4.x：继续完善注解，引入@Conditional，@EventListener等注解。@AliasFor注解。
+ Spring Framework 5.x：相对4.x改变较小，引入@Indexed。

# Spring注解编程模型

Spring注解分类：

+ 元注解（Meta-Annotaions）
+ Spring模式注解（Stereotype Annotations）
+ Spring组合注解（Composed Annotaions）
+ Spring注解属性别名和覆盖（Attribute Aliases And Overrides）

# 元注解（Meta-Annotaions）

元注解是声明在别的注解上的，像@Document注解。

如果一个注解标注在其他注解上，那么他就是元注解。

在Spring中@Component就是标注元注解，但是在spring中@Component被归类于模式注解。

# Spring模式注解（Stereotype Annotations）

模式注解是指用来声明在应用中作为组件的注解，包括@Component以及以@Component为元注解的@Service、@Repository等。

由于java注解并不允许继承，所以没有派生子类的能力。所以spring使用元注解的方式实现注解之间的”派生”。

模式注解用来标注一个类是否需要装配为bean。

# Spring组合注解（Composed Annotaions）

组合注解是指某个注解元注解中有一个或多个其他注解，而该注解的作用则是将这些元注解组合成单个自定义注解。比如：@TransactionService是由@Transaction和@Service组成，其作用也是组合这两个元注解的作用。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Transactional
@Service(value = "transactionalService")
public @interface TransactionalService 
```

组合注解的原理是通过AnnotationMetadata那一类API实现的。

# Spring注解属性别名和覆盖（Attribute Aliases And Overrides）

待补充























