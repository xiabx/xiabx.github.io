# 总览

为什么webflux会被创建

1. 第一点，我么需要非阻塞的web栈，它通过少量的硬件资源就可以实现高的性能。现在的servlet3.1提供了非阻塞IO，但是使用servelt还是会受限于Filter这些同步操作。所以要定义一个公共api作为任何非阻塞服务的基础。因为netty已经把异步非阻塞做的非常好了所以定义公共api去兼容他很重要。
2. 第二点是函数式编程，它对异步和连续风格的api很友好。

## 定义Reactive

reactive是围绕变化而做出反应的编程模式。nio就是响应式，随着数据可用我们根据通知做出反应。

在spring中reactive（根据变化做出相应）还有一个重要的机制就是背压。在同步，命令式编程中，阻塞调用是一个自然的背压它强制调用者去等待。在非阻塞io，他变成了控制事件速率使生产者不会压倒他的目的地。

Reactive Streams定义了异步组件之间的压力与反压力。比如数据库（publisher）作为生产者生产数据，http服务器（Subscriber）将数据写到response中，Reactive Streams可以让Subscriber控制publisher生产数据的快慢速率。

## Reactive API

Reactive Streams在互操作性上扮演重要角色。但是它只关心库和组件，对应用的api有很少的帮助，因为它太低级了。应用pai需要高等级的丰富的函数式api。这就需要reacitve库了。

Reactor是一个webflux可选择的响应式库。它提供mono（0...1）和flux(0...N)类型api和丰富的reactive操作符来处理mono和flux上的数据队列。Reactor是一个Reactive Streams 库，所以所有的操作都支持非阻塞后压。

Reactor是webflux的核心依赖

# 编程模式

spring-web模块包含webflux响应性基础，包括http抽象、Reactive Streams 适配器、codec、核心的webhandler.

webflux提供两种编程模式。

* 注解：和springmvc一样支持基于spring-web模块同样的注解，springmv和webflux支持响应式的返回值。不同的是webflux的@RequestBody注解的参数也支持响应式
* 函数式端点：lambda基础的、轻量级的函数式编程模型。可以看作一个小型库去路由与处理请求。

## 适用性

什么情况下用webflux

## 服务器

webflux支持tomcat、jetty这些servlet3.1+的容器，同时也支持像netty这种不基于servlet的服务器。所有服务器都可以适配低等级公共api，这样可以跨服务器支持高级的编程模型。

webflux没有内建的服务器。但是可以通过简单的配置去启动它。

webflux默认使用netty，但是可以通过调整maven依赖进行改变。

tomcat可以应用于springmvc和webflux。但是他们之间有很大不同，springmvc可以直接使用servlet api。webflux需要一个适配器才可以使用servlet api。

## 性能

webflux并不会让应用运行更快。它增加了应用的更好的伸缩性。

## 并发模型

speingmvc和webflux都支持注解contoller，他们的关键区别在于并发模型和阻塞与线程的假设。

在springmvc中，它假设应用是阻塞当前线程的，因为这个原因，所以servlet容器需要比较大的线程池去进行请求处理。

在webflux它假设应用是不阻塞的，因此可以使用更小的，线程固定的线程池去处理请求。

**实现阻塞api**

当在webflux要使用阻塞库，可以使用publishon操作，将阻塞放在另一个线程中运行。，

**可变状态**











