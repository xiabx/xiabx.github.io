---
title: Reactor相关
author: XIA
categories:
  - null
tags:
  - null
date: 2020-07-05 17:33:09
---

#  操作符

## 过滤：filter

传入一个Predicate，对flux中的内容进行过滤，然后根据传入的谓词，返回true则通过过滤。

```java
@Test
public void testFluxFilter() {
    Flux<Integer> ids = Flux.just(1, 2, 3, 4, 5);
    ids.filter(x -> {
        return x > 2;
    }).subscribe(System.out::println);
}

out：3 4 5 
```

flux是一个不可变对象，经过过滤操作的flux对象本身并不会改变，则是新建一个flux。将flux设计为不可变对象可以保证多个执行流程可以安全的使用同一个源。

```java
public void testFluxFilter() {
    Flux<Integer> ids = Flux.just(1, 2, 3, 4, 5);  //初始flux
    //经过fliter后会新建一个flux，并不是在初始flux上进行更改，初始flux并不会改变
    Flux<Integer> filterIds = ids.filter(x -> {
        return x > 2;
    });
    
    ids.subscribe(System.out::println); //out: 1 2 3 4 5
    filterIds.subscribe(System.out::println); //out：3 4 5
}
```

![image-20200705195002940](F:\SmallBear1996.github.io\source\_posts\Reactor相关.assets\image-20200705195002940.png)

## 映射：map

map操作符能够将flux发射出来的元素进行映射(转换)，如将json转换为实体对象这种操作。

![image-20200705195241076](F:\SmallBear1996.github.io\source\_posts\Reactor相关.assets\image-20200705195241076.png)

```java
//将字符串映射为长度
@Test
public void testFluxMap() {
    Flux.just("hello","world","!")
            .map(s->s.length())
            .subscribe(System.out::println);
}
```

## 扁平化映射：flatMap

flatMap是为了解决`Flux<Flux<R>>`这种对象而产生的。试想有这么一个场景，通过用户id获得用户信息的一个场景，实现思路如下面伪代码：这里userMapper.selectUserById方法的返回值是`Mono<User>`，那这个方法就是有错误的，因为通过map操作，最后的返回值为`Mono<Mono<User>>`。

```java
public Mono<User> selectUserById(String id) {
    return Mono.just(id).map(id->{
        return userMapper.selectUserById(id);
    })
}
```

这种情况就需要使用flatMap操作，flatMap会自动订阅内部的Flux流，返回内部流中包装的对象。

所以flatMap的其中一个使用场景就是，在一个映射环境中，内部会返回另一个Flux流。通过flatMap对内部流进行自动订阅，返回内部流中的包装内容。

![image-20200705203010691](F:\SmallBear1996.github.io\source\_posts\Reactor相关.assets\image-20200705203010691.png)

> 从概念上讲，flatMap()接收一个`Flux<T>`以及一个从T到`Flux<R>`类型的函数。flatMap()首先会构造一个`Flux<Flux<R>>`，将上游T类型的值替换为
>
> `Flux<R>`（与map()类似）。但是，并未结束，它会自动订阅这些内部的`Flux<R>`流，并生成一个R类型的流，这个流包含了所有内部流的值。

**两种猜测：1. flatMap自动订阅的内部Flux，然后将内部流包装的值返回，类似于map。2. flatMap内部的流代替了原来的流，继续与flatMap后面的流进行操作。以后有机会看源码再研究吧。**

flatMap内部是并发执行的，根据传入的元素获取子流的过程是并发的，所以他产生子流的耗时也是不一样的。这就引出了flatMap另一个问题就是他的无序性，每个进入faltMap中的元素进行转换，转换的结果为一个子流，然后产生的顺序并不一样，例如一个背后调用数据库的flatMap，根据传入的每个值去数据库中查找，但是并不一定先传入的值会先查到，并且产生子流。所以这就发生了子流顺序的问题。解决这种问题的方式是使用concatMap操作。

## 延迟发射：delayElements&delaySequence

这两个操作可以延迟元素的发射。他们会将发射行为放到新的调度器中。

![image-20200705205533489](F:\SmallBear1996.github.io\source\_posts\Reactor相关.assets\image-20200705205533489.png)

```java
@Test
public void testDelay() throws InterruptedException {
    Flux<Integer> integerFlux = Flux.just(1, 10, 20);
    //延迟每个元素的发射
    //每隔五秒发射一个元素
    integerFlux.delayElements(Duration.ofSeconds(5)).subscribe(System.out::println);
    //延迟整个队列元素的发射
    //延迟五秒发射一个元素
    integerFlux.delaySequence(Duration.ofSeconds(5)).subscribe(System.out::println);
    Thread.currentThread().join();
}
```

## 保证顺序的flatMap：concatMap

由于flatMap内部产生子流的过程是并发的。所以子流被订阅后的执行顺序就与最初Flux发射的元素顺序可能不同。如下代码，最终会输出2 3 5。

```java
@Test
public void testConcatMap() throws InterruptedException {
    Flux<Integer> integerFlux = Flux.just(5, 2, 3);
    integerFlux.flatMap(x->{
        return Mono.just(x).delayElement(Duration.ofSeconds(x));
    }).subscribe(System.out::println);

    Thread.currentThread().join();
}
```

解决这个问题的方式有两种，一个是指定flatMap的并发数，将并发数设置为1。

```java
public void testConcatMap() throws InterruptedException {
    Flux<Integer> integerFlux = Flux.just(5, 2, 3);
    integerFlux.flatMap(x->{
        return Mono.just(x).delayElement(Duration.ofSeconds(x));
    },1).subscribe(System.out::println);

    Thread.currentThread().join();
}
```

另一个就是使用concatMap操作。

```
public void testConcatMap() throws InterruptedException {
    Flux<Integer> integerFlux = Flux.just(5, 2, 3);
    integerFlux.concatMap(x->{
        return Mono.just(x).delayElement(Duration.ofSeconds(x));
    }).subscribe(System.out::println);

    Thread.currentThread().join();
}
```

> flatMap()内部使用了merge()操作符，同时订阅所有的子Observable，对它们不做任何的区分。这也是下游事件互相交叉的原因。但是，concatMap()可以在
>
> 技术上使用concat()操作符。concat()只会先订阅第一个底层的Observable，只有第一个完成之后，才会订阅第二个。

## 合并多个源Flux：merge

当我的源数据来源于多个Flux，而这些数据源使用同一套执行流程，这种情况可以使用merge操作将多个源Flux合并为一个。

merge会保留一个对所有被合并Flux的引用，当被合并的Flux发布数据时，则会被转发到合并的Flux上。这是没有顺序的，并不会一个被合并的Flux发布完数据其他的Flux再发送。

```java
public void testMerge() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("aa", "bb", "ccc");
    Flux<String> stringFlux2 = Flux.just("dd", "eee", "ffff");

    Flux.merge(stringFlux,stringFlux2).map(s->{
        return s+": "+s.length();
    }).subscribe(System.out::println);

}
```



























