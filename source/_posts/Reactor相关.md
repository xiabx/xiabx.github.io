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

## 合并多个源Flux：merge、concat

当我的源数据来源于多个Flux，而这些数据源使用同一套执行流程，这种情况可以使用merge操作将多个源Flux合并为一个。

merge会保留一个对所有被合并Flux的引用，当被合并的Flux发布数据时，则会被转发到合并的Flux上。这是没有顺序的，并不会一个被合并的Flux发布完数据其他的Flux再发送。

```java
public void testMerge() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("aa", "bb", "ccc");
    Flux<String> stringFlux2 = Flux.just("dd", "eee", "ffff");

    Flux.merge(stringFlux,stringFlux2).map(s->{
        return s+": "+s.length();
    }).subscribe(System.out::println);

    //输出并不是按照源的顺序进行的
}
```

concat会以顺序的方式发布被合并的源，当前一个源的数据完全发布后，之后的源的数据才会被发布。

```java
public void testConcatMerge() throws InterruptedException {
    Flux<Integer> integerFlux = Flux.range(1, 5).delayElements(Duration.ofSeconds(1));
    Flux<String> stringFlux = Flux.just("a", "b", "c", "d", "e").delayElements(Duration.ofSeconds(2));

    Flux.concat(integerFlux,stringFlux).subscribe(System.out::println);

    Thread.currentThread().join();
    
    //out:
    //1 2 3 4 5 a b c d e
}
```

## 组合：zip&zipWith

zip操作会组合两个或多个流产生的数据，将他们封装为一个元组(Tuple)数据结构。*元组是一个类似于数组的数据结构，封装多个数据，可以使用get方法获取包装的数据。*当要合并两个发射源时可以使用Flux的成员方法zipWith。

如下代码，定义两个发射源，使用zip操作将他们组合在一起，然后订阅组合后的发射源，订阅者接收的数据类型为元组。

```java
public void testZip(){
    Flux<String> stringFlux = Flux.just("a", "b", "c", "d");
    Flux<Integer> integerFlux = Flux.just(1, 2, 3, 4);
    //组合后，新的源发射元素类型为Tuple
    Flux.zip(stringFlux,integerFlux).subscribe(x->{
        System.out.println(x.getT1()+x.getT2());
    }); //输出 a1 b2 c3 d4
}

public void testZip2() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("a", "b", "c", "d");
    Flux<Integer> integerFlux = Flux.just(1, 2, 3, 4);
    //可以使用传入BiFunction的方法，这样就使Tuple变得透明
    Flux.zip(stringFlux,integerFlux,(x,y)->{
        return x+y;
    }).subscribe(System.out::println);

}
```

zip操作的特点是，它会等到所有的底层操作源都发射数据时才会发射出的数据进行组合，当被组合的源中有一个没有发射新的数据，则其他的源发射的数据将被缓存，等待缓慢的发射源。

## 流之间不同步的情况：combineLatest()、withLatestFrom()

当被zip的多个数据源发射数据的速度相似时，使用zip可以获得较好的效果，当被zip的多个源的发射速度存在明显差异，这时候就会出现发射速度慢的源产生的数据可以被立即消费，而发射数据快的源产生的数据会被换缓存起来进行等待，随着时间的推移，两个发射源之间的是时间差异越来越大，产生过期的数据，甚至会导致内存泄漏。

```java
public void testZip1() throws InterruptedException {
    //两个发射源产生数据的速度不同
    Flux<String> stringFlux = Flux.just("a", "b", "c", "d").delayElements(Duration.ofSeconds(3));
    Flux<Integer> integerFlux = Flux.just(1, 2, 3, 4).delayElements(Duration.ofSeconds(6));
	//timestamp()操作会返回一个Tuple结构的数据，包含一个数据的发射时的时间戳，一个数据值。
    Flux.zip(stringFlux.timestamp(),integerFlux.timestamp(),(x,y)->{
        return y.getT1()-x.getT1();
    }).subscribe(System.out::println);

    Thread.currentThread().join();
    
    //out：可见两个发射源之间的时间差越来越大
    //3000
	//5999
	//8999
	//12001
}
```

**combineLatest操作符**实现的效果是，假如组合两个流，产生数据速度快的源不再等待产生数据慢的源，而是产生数据快的源产生数据后会直接使用产生数据慢的源所产生的相对新值。该操作符的发射数据的主动权在速度快的源。如下，当产生速度快的源产生了前三个数据，但是产生速度慢的源才产生一个数据`1`,所以会直接组合为 a1 b1 c1。

```java
public void testZip1() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("a", "b", "c", "d").delayElements(Duration.ofSeconds(1));
    Flux<Integer> integerFlux = Flux.just(1, 2, 3, 4).delayElements(Duration.ofSeconds(2));

    Flux.combineLatest(stringFlux,integerFlux,(x,y)->{
        return x+y;
    }).subscribe(System.out::println);

    Thread.currentThread().join();
    
    // out: a1 b1 c1 c2 d2 d3 d4
}
```

**withLatestFrom**可以更灵活的指定组合后发射的时机主动权。当使用combineLatest操作符时，只有只有当发射数据快的源产生数据时才会去速度慢的源获取最新值，组成新的数据。使用withLatestFrom则可以使发射速度慢的源获得主动权，当速度慢的源产生数据时会去速度快的源查看最新数据，然后组成新的要发射的数据，而速度快的源发射的没有被用到数据则会被丢弃。

```java
public void testWithLatestFrom() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("a", "b", "c", "d").delayElements(Duration.ofSeconds(1));
    Flux<Integer> integerFlux = Flux.just(1, 2, 3, 4).delayElements(Duration.ofSeconds(3)).startWith(0);

    integerFlux.withLatestFrom(stringFlux,(i,s)->{
        return i+s;
    }).subscribe(System.out::println);
    Thread.currentThread().join();

    //out: 1b 2d 3d 4d
}
```

## 扫描整个序列：scan、reduce

假设有这么一个需求，需要统计一个Flux源中所有字符串的总长度。可以使用在外部定义一个表示总长度的变量，然后使用map操作对length增加每个字符串长度。

```java
    @Test
    public void testScan() throws InterruptedException {
        Flux<String> stringFlux = Flux.just("hello", "world", "hello", "java");
		//字符串总长度
        AtomicInteger length = new AtomicInteger(0);
        stringFlux.map(x->{
            length.addAndGet(x.length());
            return x;
        }).subscribe();
        System.out.println(length);

        //out: 19
    }
```

但是这种思路会带来其他的一些问题。这种情况下可以使用scan或reduce操作符，他们内部会维护一个变量，用来保存中间状态。他们的区别在于scan操作会返回每次操作，而reduce操作则只会在最后一数据计算结束后才会返回。

```java
@Test
public void testScan() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("hello", "world", "hello", "java");
	// 指定这个中间状态的初始值
    stringFlux.scan(0,(x,y)->{
        return x+y.length();
    }).subscribe(System.out::println);

    //scan操作会将每次的操作都返回
    //out: 0 5 10 15 19
}
```

```java
@Test
public void testReduce() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("hello", "world", "hello", "java");
	
    stringFlux.reduce(0,(x,y)->{
        return x+y.length();
    }).subscribe(System.out::println);

    //reduce只返回最后的结果，不返回中间的值，相当于在reduce处阻塞了
    //out: 19
}
```

scan和reduce操作都可以不传入指定的初始值，只传入BiFunction。这样初始值就是第一个得到的第一个元素。

```java
public void testScan() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("hello", "world", "hello", "java");
	//不传入初始值的scan操作
    stringFlux.scan((x,y)->{
        System.out.print(x+":"+y);
        System.out.println();
        return x+y;
    }).subscribe();

    //out: hello:world  helloworld:hello  helloworldhello:java
}
```

## 收集元素：collect

将Flux流中的每个元素收集起来，组成一个Mono。比如把一个发射string的Flux进行收集为`Mono<List<String>>`,这种功能也可以使用reduce实现，但是使用collect可以减少代码冗余。

```java
public void testCollect1() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("hello", "world", "hello", "java");
	//使用collectList收集为List
    stringFlux.collectList().subscribe(System.out::println);
	//指定收集对象的初始值，然后处理收集逻辑
    stringFlux.collect(ArrayList::new,(list,s)->{
        list.add(s);
        return;
    }).subscribe(System.out::println);
	//使用reduce操作符实现收集功能，但是这样会有冗余，比如list的add方法并不返回list，还需要手动返回
    stringFlux.reduce(new ArrayList<String>(),(list,s)->{
        list.add(s);
        return list;
    }).subscribe(System.out::println);

    //out: [hello, world, hello, java]
}
```

## 去除重复：distinct、distinctUntilChanged

distinct操作可以去除Flux流中与之前重复(根据hashcode和equals)的数据，distinctUntilChanged操作可以去除掉与上一次一样的数据。

```java
public void testDistinct() throws InterruptedException {
    Flux<String> stringFlux = Flux.just("hello", "world","hello", "java");
	//去除与之前数据中相同的数据
    stringFlux.distinct().subscribe(System.out::println);
    //去除与之前数据长度中相同长度的数据
    stringFlux.distinct(String::length).subscribe(System.out::println);

    //当数据变化时，才会通过该操作，否则被丢弃
    stringFlux.distinctUntilChanged().subscribe(System.out::println);
    //当数据长度变化时，才会通过该操作符
    stringFlux.distinctUntilChanged(String::length).subscribe(System.out::println);
    
}
```

## 对数据进行切块：take、skip等

可以对flux指定只取出指定数据

```java
public void test() throws InterruptedException {
    Flux<Integer> integerFlux = Flux.range(1, 5);

    integerFlux.take(3).subscribe(System.out::println); //取前三个数据 1 2 3
    integerFlux.skip(3).subscribe(System.out::println); //跳过前三个数据 4 5


    integerFlux.takeLast(3).subscribe(System.out::println); //取最后三个数据 3 4 5
    integerFlux.skipLast(3).subscribe(System.out::println); //跳过最后三个数据 1 2

    integerFlux.takeUntil(i->i>2).subscribe(System.out::println); //一直输出直到符合takeUntil中条件 1 2 3

    integerFlux.elementAt(2).subscribe(System.out::println); //输出下标为2的数据 3
    integerFlux.count().subscribe(System.out::println); //统计总共发送了多少数据 5
    
}
```

# 将反应式编程应用到现有应用程序

介绍如何一步步将反应式编程应用到现有程序，介绍反应式在应用程序中的使用场景。

## 从集合到Flux

这样一个场景，在dao层，我们会查询数据库，查询结果将返回`List<T>`，这里的List就可以用Flux代替了。

```java
//原来的返回类型为List
public List<User> listUser(){
    List<User> users = userMapper.selectAllUser();
    return users;
}

//使用Flux进行替换
public Flux<User> listUser(){
    Flux<User> userFlux = Flux.fromIterable(userMapper.selectAllUser());
    return userFlux;
}
```



















