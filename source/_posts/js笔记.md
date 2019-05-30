---
title: js笔记
date: 2019-05-29 16:09:46
categories:
- 前端
tags:
- js&JQuery
---
# js实现
js实现至少应该包括三部分：
- 核心（ECMAScript）
- 文档对象模型（DOM）
- 浏览器对象模型（BOM）
ECMAScript是一种规范，规定了实现该规范的语言的语法，类型，语句，关键字等的规范。js只是
ECMAScript规范的一种实现。
# js基本概念
## 数据类型
js中有5种数据类型：
- Undefined：变量未声明或未初始化
- Null
- Boolean
- Number
- String
- Object：object是所有对象的基础，object的每个实例都包含以下属性：
  - constructor:创建对象时的构造函数
  - hasOwnProperty(propertyName)：检查给定的属性名是否在该原型中，而不是在该实例的原型中。
  - isPrototypeOf(object):传入的对象是否是传入对象的原型
  - toString():返回对象的字符串表示
  - valueOf():返回对象的字符串，数值，布尔值得表示。通常和toString()的返回结果相同
## typeof 操作符
出现原因：由于js是松散类型的语言，所以就出现了一种可以检测变量类型的操作符，typeof。
typeof的返回结果：undefined，boolean，string,number,object,function
typeof null --->object 因为：null是一个空对象
函数在js中也是对象，但是他具有一些特殊属性，因此typeof区分object和function是有必要的
## 相等 不相等 全等 不全等
**相等 不相等**
相等操作符 ==
不相等操作符 !=
比较之前会对将要比较的双方进行强制类型转换。比如：数值和布尔，布尔会强转为数字，true--1，false--0
**全等 不全等**
全等 ===
不全等 !==
只有当比较的双方，数值和数据类型都相同时才会返回true。
## 语句
for-in语句：循环对象的属性
## 函数
### js函数的参数
js函数不介意传递参数的个数，比如：定义时两个，调用时可以传递三个，两个。。。js函数中有一个arguments属性，类似于数组包含方法调用时传递的所有参数。
**所以js中没有方法的重载，但是可以根据arguments的属性和类型模拟重载。**
# 变量及作用域 
## 基本类型和引用类型
- 基本类型：Undefined Null Boolean Number String
- 引用类型： 对象
引用类型可以动态的添加属性，而基本类型则不可以。
```js
//引用类型可以动态添加属性
var person = new Object();
person.name = "xia";
console.log(person.name);
//基本类型不可以动态添加属性
var s = "hello";
s.name="bao";
console.log(s.name);
```
## instanceof 操作符 
因为typeof操作符无法检测对象是哪个类型的对象，所以出现了instanceof操作符，例如：
```js
var p = new Object();
console.log(p instanceof Object);//true
console.log(p instanceof Array);//false
```
## 作用域链
每个函数执行都需要一个执行环境，这个执行环境是一个链式的结构。即作用域链。作用域链的最前端是一个arguments对象和该函数所包含的变量，下一个环境对象来自下一个包含环境，一直到全局执行环境。在浏览器中是window对象。
## 引用类型
### Object
**创建Object对象的两种方式：**
```js
//第一种类型
var p = new Object();
p.name = "xia";
console.log(p.name);
//第二种类型，属性名可以为字符串也可以直接写
var a = {
    name:"bao",
    "age":18
}
console.log(a.name);
```
**从object对象中取值的两种方式**
```js
var a = {
    name:"bao",
    "age":18
}
//直接通过点取值
console.log(a.age);
//通过属性名方括号取值
console.log(a["age"]);
//方括号取值的主要应用场景
var parm = "age";
console.log(a[parm]);
```
### Array
**简介**
js的数组是可以扩容的，而且一个数组对象可以在不同的索引存储多种对象。
```js
//定义一个数组
var arr = [0,1,2,3,4];
console.log(arr.length);
//数组是可以扩容的，直接往新位置添加元素就可以
arr[5]=5;
console.log(arr.length);
```
**Array对象的输出**
数组对象的join方法可替换分隔符
```js
var arr = [0,1,2,3,4];
console.log(arr.toString()); //0,1,2,3,4
console.log(arr); //[ 0, 1, 2, 3, 4 ]
//join方法替换分隔符
console.log(arr.join("、")); //0、1、2、3、4
```
**数组对象的栈方法**
push()和pop()方法
```js
var arr = [0,1,2,3,4];
arr.push("5");
console.log(arr.pop()); //5
console.log(arr.pop()); //4
console.log(arr.pop()); //3
```
**数组的队列方法**
**数组排序 与 重排序**
sort方法可以为数组排序，但是默认按字符串排序，所以需要重写排序方法。
```js
//从小到大排序
var arr = [1,8,50];
function compare (val1,val2) {
    if (val1 > val2) {
        return 1;
    }else if (val1 < val2) {
        return -1;
    }else{
        return 0;
    }
}
console.log(arr.sort(compare));
```
reverse方法对数组重排序
```js
var arr = [1,8,50];
console.log(arr.reverse()); //[ 50, 8, 1 ]
```
**数组的迭代方法**
数组的迭代方法有every,filter,forEach,map,some每个方法的功能不同，详见96页
每个方法传入两个参数，一个运行函数（包含三个参数，数组项的值、该项的索引、数组对象本身）一个作用域对象(可选)。
```js
var arr = [1,8,50];
arr.forEach(function (item,index,arr) {
    console.log(item+1);
})
```
**数组的归并方法**
### Date
p98
### RegExp
正则表达式 p103
### Function
函数是一个对象，是Function的对象
- 定义函数的两种方式

- 由于函数名实质就是指针，深刻理解js没有重载

- 函数提升

- 既然函数名是指针，所以可以将函数名作为另一个函数的参数

  **函数的内部属性**
  arguments，this和caller

- arguments:保存着传入函数的所有参数，同时还有一个callee的属性，指向拥有该arguments对象的函数

- this:指向该函数执行的环境对象

- caller:保存着调用该函数的函数引用
```js
function outer() {
    inner();
}
function inner() {
    console.log(arguments.callee.caller);
}
outer();//[Function: outer]
```
**函数的属性和方法**

- prototype属性
- apply()和call()方法：在特定作用域调用该函数,区别：apply()第二个参数为数组或arguments对象，call()第二个参数为列出的所传的参数。
- bind()方法：创建一个函数实例，其this值回绑定到传递的参数
### global对象
兜底对象，所有全局作用域中定义的对象和函数，都是它的属性。
- URI编解码方法：encodeURI()，encodeURIComponent(),decodeURI(),decodeURIComponent()。有效的URI不能包办某些字符如空格等。这几个方法就是对URI进行编解码。
  - encodeURI()，encodeURIComponent()：后一个方法比前一个编码更彻底。前一个只能编码空格，后一个除了空格之外冒号，斜杠等符号都可以编码。
  - decodeURI(),decodeURIComponent()：同上，后一个解码更彻底。
- eval()：执行参数的js语句
### window对象
在web浏览器中，window对象可以作为全局对象。

# 面向对象
## 理解对象与属性类型
- js创建对象两种方式：直接new Object（）与使用字面量创建
- ECMA有两种属性：数据属性和访问器属性
```js
//数据属性
var person = {
    name : "xia"
}
//对象 属性名 描述符对象
Object.defineProperty(person,"name",{
    value:"baoxin"
})

```
```js
//访问器属性，创建get和set
var book = {
    _year:2014,
    edition:1
}

Object.defineProperty(book,"year",{
    get:function () {
        return this._year;
    },
    set:function (newValue) {
      if (newValue > this._year) {
          this._year = newValue;
          this.edition+=1;
      }  
    }
})
book.year = 2002;
console.log(book.year);
```
## 创建对象的方式

- 工厂模式
- 构造函数模式
- 原型模式
### 工厂模式和构造函数模式
**使用工厂模式创建对象**
使用工厂模式解决的创建多个对象繁琐的问题，但是没有解决对象识别的问题。
```js
//使用工厂模式创建对象
function createPerson(name,age,job) {
    var o = new Object();
    o.name = name;
    o.age = age;
    o.job = job;
    o.sayName = function () {
        console.log(this.name);
    }
    return o;
}

var person = createPerson("xia",23,"coder");
person.sayName();
```
**构造函数模式**
构造函数模式解决了对象识别问题，可以通过instanceof操作符获取对象类型。
```js
//构造函数
function Person(name,age,job) {
    this.name = name;
    this.age = age;
    this.job = job;
    this.sayName = function sayName() {
        console.log(this.name);
    }
}
//使用new关键字创建对象
var person1 = new Person("xia",23,"coder")
//检测对象类型
console.log(person1 instanceof Person);
console.log(person1 instanceof Object);
//每个对象中包含constructor属性（存在于prototype中），指向生成该对象的构造函数
console.log(person1.constructor);
```
构造函数除了使用new关键字调用之外就是一个普通的函数。调用构造函数的基本过程：

1. 创建一个新对象
2. 将构造函数的作用域赋值给新对象，即this指向这个新对象
3. 执行构造函数中的代码（为这个新对象添加属性）
4. 返回新对象
```js
//构造函数执行的大致过程
function Animal(name) {
    this.name = name;
}

var ani = new Object();
Animal.call(ani,"cat");
console.log(ani.name);
```
### 原型模式创建对象
js在创建每个函数对象时都会创建一个该函数的原型对象，每个函数对象中提供一个指针，prototype指向该原型对象。创建自定义函数以后原型对象只会取得constructor属性，该属性指向拥有该原型对象的函数。

当调用构造函数之后创建的对象包含一个名叫[[Prototype]]的指针，指向构造函数的原型对象，在浏览器环境中提供了proto属性，可以访问该属性。通过Object.getPrototypeOf()可以获取到该对象构造函数的原型的值。

重写一个函数的原型对象时，如果需要则重写constructor属性，因为重写时会将该属性覆盖

实例中的指针仅指向原型，不指向构造函数。

**关于原型prototype常用的方法**
- isPrototypeOf() 确定对象与原型之间是否存在关系
- Object.getPrototypeOf() 获得某对象的[[Prototype]]指针指向的原型对象
- hasOwnProperty() 检测一个属性是否为实例属性
- in操作符，检测对象是否包含该属性，无论是在对象中还是在原型中
- for-in循环，获取所有可枚举属性(即对象属性设置[[Enumerable]]为true)
- Object.keys()（只能获取可枚举属性）与Object.getOwnPropretyNames()（可获取不可枚举属性）获取对象的属性，用以代替for-in循环
```js
        function Person(){
    
        }
        Person.prototype.age = 3;
        Person.prototype.sayName = function (){
            console.log(this.name);
        }

        var p1 = new Person();
        p1.name = "tom";

        //输出函数的原型对象
        console.log(Person.prototype);

        
        //直接调用原型中的方法
        p1.sayName()


        //constructor为函数生成时原型自动创建的属性,输出构造函数的源码
        console.log(p1.constructor);


        //获得某对象的[[Prototype]]指针指向的原型对象
        console.log(Object.getPrototypeOf(p1));

        
        // true 检测一个属性是否为实例属性
        console.log(p1.hasOwnProperty("name"));


        //true 检测对象是否包含该属性，无论是在对象中还是在原型中
        console.log("name" in p1);
        console.log("age" in p1);


        //获取所有可枚举属性(即对象属性设置[[Enumerable]]为true),包含实例属性和原型属性
        for(var arg in p1){
            console.log(arg);
        }


        //获取对象属性，只能获取可枚举属性，只能获取实例属性
        console.log(Object.keys(p1));
        
        //获取对象属性，可获取不可枚举属性，只可获取原型属性
        console.log(Object.getOwnPropertyNames(Person.prototype));

```
**使用原型的几种方式**
- 组合使用构造函数与原型的模式。将共享的变量或方法放在构造函数的原型对象中
- 动态原型模式。在构造函数中初始化原型。
- 寄生构造函数模式。当new一个构造函数包含一个返回值时，则获得的对象是那个返回值
- 稳妥构造函数模式。不使用this和new关键字

## 继承
### 原型链实现继承
将子类的构造函数prototype属性设置为父类的实例，实现继承。

但是原型链实现继承这种方式存在一些问题，如父类的引用类型值会被共享，以及无法向父类构造传递参数的问题。
```js
//父类
function SuperType() {
    this.property = true;
}
SuperType.prototype.getSuperValue = function () {
    return this.property;
}
//子类
function SubType() {
    this.subProperty = false;
}
//子类的构造的原型设置为父类实例
SubType.prototype = new SuperType();

SubType.prototype.getSubValue = function () {
    return this.subProperty;
}
//创建子类实例
var instance = new SubType();
//true
console.log(instance.getSuperValue());
//true
console.log(Object.getPrototypeOf(instance).property);

//子类的实例 使用intanceof父类时  为 true
console.log(instance instanceof Object);
console.log(instance instanceof SubType);
console.log(instance instanceof SuperType);
```
### 借用构造函数实现继承
在子类的构造函数中调用父类的构造函数，实现向父类构造传值的功能。

因为没有使用原型，所以无法实现函数的复用。

```js
function SuperType(name) {
    this.name = name;
}

function SubType() {
    //调用父类的构造
    SuperType.call(this,"xia");
    this.age = 18;
}

var instance = new SubType();
//18
console.log(instance.age);
//xia
console.log(instance.name);
```
### 组合继承
既然使用原型链和借用构造函数实现继承的方式各有利弊，组合继承便是将二者结合起来。
```js
function SuperType(name) {
    this.name = name;
    this.colors = ["red","blue"];
}
//设置公共方法
SuperType.prototype.sayName = function () {
    console.log(this.name);
}

function SubType(name,age) {
    SuperType.call(this,name);
    this.age = age;
}
//实现原型链的继承，获得公共方法
SubType.prototype = new SuperType();
SubType.prototype.constructor = SubType;

var person1 = new SubType("xia",18);
var person2 = new SubType("ma",19);

person1.sayName();//xia
person2.sayName();//ma

console.log(person1.age);//18
console.log(person2.age);//19
```
# 函数表达式
## 定义函数的两种方式
- 使用函数声明：存在函数声明提升的特征
- 使用函数表达式：先定义一个匿名函数，然后赋值给一个变量
```js
//使用函数声明来定义函数,可以在定义之前使用函数
fun1();
function fun1() {
    console.log("fun1");
}

//使用函数表达式的方式定义函数
var fun2 = function () {
    console.log("hello");
}
fun2();
```
## 作用域链
当某个函数被调用时，会创建一个执行环境及其相应的作用域链。然后使用arguments和其他命名参数的值初始化函数的活动对象。
>作用域链的非自己部分在函数对象被建立（函数声明、函数表达式）的时候建立，而不需要等到执行
>作用域链的前面部分是静态的，所有函数共享同一个链，当函数执行时，建立一个自己当次执行的作用域，然后把这个作用域与前面共享的链关联起来
```js
function compare(value1,value2) {
    if (value1 > value2) {
        return 1;
    }else if (value2 > value1) {
        return -1;
    }else{
        return 0;
    }
}
compare(1,2);
```
这个函数的活动对象为:arguments和value1，value2。

在作用域链中，头是活动对象，然后依次是外层函数的活动对象。。。直到window对象

## 闭包
### 闭包概念
闭包是指有权访问另一个函数作用域中的变量的函数。

函数在生成时就已经确定了他的作用域链。

闭包的形成原因就是在js中函数在创建时候会创建作用域链。

```js
function createFunction(name) {
    var name = "john";
    return function () {
        console.log(name);
    }
}

//john
createFunction()();
```
### 闭包中的this
内部函数在搜寻this时，只会在活动对象中寻找。
## 模仿块级作用域
块级作用域：例如for循环的i。

原理：函数声明后不可以直接跟括号，立即执行。但是函数表达式可以。用括号将函数声明包裹起来，就会使函数声明变成函数表达式，再紧跟括号立即执行。利用函数的作用域模仿块级作用域。
# BOM
## window对象
- window对象既是ECMAScript在浏览器环境中的全局对象，也是js提供的一个访问浏览器窗口的一个接口
- 当在html中使用frames时，每一个frame对应一个window对象。
- window对象的screenTop和screenLeft属性会显示浏览器窗口距离屏幕边缘的距离，通过window的moveTo()和moveBy()方法设置窗口位置和移动窗口
- window的innerWidth,innerHeight,outerWidth,outerHeiht可以获取浏览器窗口的内外宽高。通过window的resizeTo()和resizeBy()可以调整窗口大小
- window.open()打开一个新窗口，传三个参数。第一个为新窗口打开的连接，第二个为打开的窗口的位置，第三个为特征参数（比如新窗口的宽高，位置等）。方法调用后会返回一个打开的新窗口的window的引用，可以调用close()方法关闭新打开的窗口。
- 间歇调用和超时调用，setTimeout()和setInterval()。实际上js是单线程的语言，所有执行的代码都在一条线程上直线，之所以会产生定时器的效果是因为再js中有一个执行队列，当设置的时间到了，js会将要执行的代码放置到知执行队列中。设置的时间只是将代码放置到队列中，并不保证立即执行。执行要等到队列里他前面的函数都执行完了再执行。
- 提示框 alert(),confirm(),prompt()
```js
//设置超时调用，返回id
var timeOutId = setTimeout(() => {
    console.log("hello world");
}, 2000);
//清除超时调用
clearTimeout(timeOutId);

//设置间歇调用，返回id
var i = 0;
var intervalId = setInterval(() => {
    if (i<10) {
        console.log(i++);    
    }else{
        //清除间歇调用
        clearInterval(intervalId);
    }
}, 1000);
```
```js
//显示提示框
alert("hello");

//显示包含确认和取消的提示框，点击确认返回true，取消返回false
if(confirm("你爱我吗？")){
    console.log("我也爱你");
}else{
    console.log("死鬼！！！");
}

//显示一个可以输入文本的提示框，第一个参数为提示信息，第二个参数为默认值，返回输入的值
var value = prompt("tip","hello");
console.log(value);
```
## location对象
location对象提供了当前窗口的导航信息。是window对象的一个属性。

- location对象中保存的一些属性：P207

![location属性](/img/location.png "location属性")

- 位置操作:
  - 改变浏览器的位置：通过location的assign()方法传入一个url即可实现页面的跳转。该方法等同于window.location和location.href。直接修改location的属性也可以实现跳转。
  - 替换当前页面：使用location的replace()方法可以替换当前页面。替换后的页面不可以后退。
  - 重载当前页面：location的reload()方法可以重载当前页面。

## navigator对象
该对象为window对象的一个属性。包含浏览器的一些信息，如：浏览器的语言，是否联网，浏览器中安装插件的数组，所在操作系统等信息。
## screen对象
该对象是window对象的一个属性。包含了浏览器所在屏幕的一些信息，如：屏幕的像素，高度等信息
## history对象
- 该对象为window对象的一个属性，包含了用户在包含当前window对象的窗口或标签页中的浏览记录
- 可以使用history.go()方法进行跳转，类似于浏览器的前进后退按钮，方法参数为正数表示前进，负数表示后退，参数为字符串时跳转到包含此字符串的最近的页面
- history的back()和forward()方法一样可以实现前进和后退
- history的length属性表示当前窗口打开的历史记录数量
# DOM
## DOM的节点层次理解
Node是一个基类，DOM中的Element，Text和Comment都继承于它。
换句话说，Element，Text和Comment是三种特殊的Node，它们分别叫做ELEMENT_NODE,TEXT_NODE和COMMENT_NODE。

所以我们平时使用的html上的元素，即Element是类型为ELEMENT_NODE的Node。

Document类型是所有DOM树的根节点
## Node类型
- 所有节点都是node的子类，node类型主要定义了节点之间的关系。
- 每个Node类型都有nodeName和nodeValue属性
  - 在Element类型的元素中nodeName始终为标签名，nodeValue始终为null
  - 在Document类型元素中nodeName始终未#document，nodeValue为null
  - 在Text类型的元素中nodeName始终为#text，nodevalue为节点所包含的文本
- 每个节点都有一个childNodes属性，其中保存着一个NodeList对象。这是一个类似于数组的对象，有length属性，可以通过中括号加索引的方式取值。
- 节点的parentNode属性，保存着该节点的父节点对象。
- 节点的previousSibling和nextSibling属性可以获取同一层级下的兄弟节点
- 父节点的firstChild和lastChild属性保存着第一个和最后一个子节点
- 操作节点：（都是操作子节点）
  - 父节点调用appendChild()方法，可以向childNodes列表的末尾添加一个节点
  - 父节点调用insertBefore()该方法有两个参数，要插入的节点和参照节点。即在参照节点之前插入一个节点
  - 父节点调用replaceChild()方法该方法接受两个参数，要插入的节点和要替换的节点
  - 父节点调用removeChild()方法，该方法接受一个要删除的节点的参数，移除该子节点。
## Document类型
- document对象是window的一个属性
- Document节点只包含一个子节点就是`<html>`元素
- document.documentElement--> 获取`<html>`元素
- document.body-->获取`<body>`元素
- document.title 可以获取`<title>`标签内的文本，也可以设置网页的标题
- 查找元素：
  - document.getElementById() 根据标签的id值获取获取元素
  - document.getElementByTagName() 获取指定表签名的所有元素，如：div,img等。获得元素保存在HTMLCollection中，这是一个类似于NodeList的结构，有length属性，可以通过中括号加索引取到相应的值，也可以通过中括号加标签的name属性取值
  - document.getElementByName() 通过标签的name属性进行取值，返回的也是一个HTMLCollection集合
- document可以通过write(),writln(),open(),close(),方法向页面中写数据
## Element类型
- Element是html文档中的所有标签的类型，通过document.getElementById()这类方法拿到的对象就是Element类型的对象。像：div,a,img,span...都是Element类型
- 每个Element类型的对象都有以下几个属性：
  - id：对应标签中的id属性
  - title：元素的附加说明信息，鼠标移上去才可以看见
  - dir：语言方向，ltr从左到右，rtl从右到左
  - className：对应标签中的class属性，因为class是es中的关键字，所以用className
- 取得标签属性：
  - getAttribute()取得指定名称的属性值，当取style和onclick时不要使用
  - setAttribute()设置指定名称的属性值，传入属性名和要设置的属性值。没有时就新建一个属性
  - removeAttribute()删除指定的属性，不仅会清楚属性值，还会清除属性
 - 创建Element类型元素：通过document.createElement()方法可以生成一个Element类型的元素。然后将其添加到指定位置。该方法接受两种参数，一种是标签名，一种是元素全部的字符串描述。
## Text类型
- 一个Element开合标签之间的文本内容就是Text类型的节点，如：`<div>hello</div>`在div之间的内容就是Text类型
- 可以通过他的父节点的firstChild属性获取文本节点的引用
- 通过文本节点的nodeValue属性为该文本节点设置值
- 创建Text类型的节点：document.createTextNode()创建新的文本节点。然后通过插入的方式将其添加到文档树之中
## Comment类型
注释类型，引用注释元素
## Attr类型
Element元素中的属性也是一种元素，也是DOM树的一部分，但是开发人员不这么认为，他们通常使用getArrtibute()等方法进行获取，设置，删除属性的操作
# DOM扩展
## 选择符API的扩展
选择符API就是根据css的选择器选择元素。
- querySelector()方法，根据css的选择符，返回与该模式匹配的第一个元素
- querySelectorAll()方法，同样是根据css选择符，返回与该模式匹配的元素，但是返回的是一个NodeList对象。
- 能调用querySelector()方法和querySelectorAll()方法的类型包括Document和Element
## HTML5有关的扩展
- getElementByClassName()接收一个元素的类名，返回一个NodeList对象
- classList属性：用来为元素增删class属性。为了简化className的操作，使用方法为：`div.classList.remove("user")`
  - add(value)为给定个classList添加类名
  - contains(value)是否包含给定的值，包含true，不包含false
  - remove(value)删除指定的的类名
  - toggle(value)如果classList中包含此类名就删除，不包含就添加
## 自定义数据属性
h5规定元素可以添加自定义属性，但是要以data-开头。

元素的dataset属性访问定义的自定义属性，dataset包含的是一个键值对map类型的对象。
## 插入标记
- innerHtml属性：
  - 读模式：返回与调用元素的所有子节点对应的HTML标记。
  - 写模式：innerHtml会根据指定的值创建新的DOM树，然后用这个DOM树完全替换调用元素的所有子节点。
- outerHTML属性：
  - 读模式：返回调用他的元素的及所有子节点的HTML标签。
  - 写模式：根据指定的HTML字符串创建新的DOM树，然后用这个DOM树完全替代调用元素
- insertAdjacentHTML()方法：接受两个参数，一个是位置参数（如下），一个是要插入的html文本
  - beforebegin：在当前元素之前插入一个紧邻的同辈元素
  - afterend：在当前元素之后插入一个紧邻的同辈元素
  - afterbegin：在当前元素之下插入一个新的子元素或在第一个子元素之前在插入新的子元素
  - beforeend：在当前元素之下插入一个新的子元素或在最后一个子元素之后再插入新的子元素
- innerText属性：
  - 读模式：可以由浅入深的将文档树中的文本拼接起来
  - 写模式：删除元素的所有子节点，然后插入相应的文本节点
- outerText属性：
  - 读模式：与innerText属性的返回结果相同
  - 写模式：会用新的文本节点替换调用元素和他的子节点
# DOM2和DOM3
啥是dom2和dom3，不晓得，管他呢
## 访问元素的样式
- 任何支持style属性的HTML元素在js中都有一个style属性
- 当css样式中包含“-”时，则对应的在js属性是驼峰命名法，float是js的保留关键字，所以float对应的属性名是cssFloat
- style的常用方法和属性
  - length：给定元素的css属性的数量
  - removeProperty（）：从样式中删除给定的属性
# 事件
## 事件流（冒泡和捕获）
如下html中，当点击div元素时，同时也是点击了body和html元素。而事件流则讨论的是，事件在div，body，html着几个元素的触发顺序。
```html
<html>
    <head>
       <title>Event Bubbling Example</title>
    </head>
    <body>
       <div id="myDiv">Click Me</div>
    </body>
</html>
```
- 事件的冒泡阶段：从里到外触发，即先触发div再是body然后是html元素
- 事件的捕获阶段：从外向里触发，先触发html然后是body然后是div元素
- 事件的三个阶段：捕获阶段，处于目标阶段，事件冒泡阶段。如下图，1.2.3是捕获阶段，4是处于目标阶段(看作是冒泡的一个阶段)，5.6.7是冒泡阶段
  

![事件流](/img/event.png)
## 事件处理程序
- 在事件处理函数内部，this的值等于事件的目标元素
- 为元素添加事件的方式
  - 直接在元素的html代码上添加,如：`<input type="button" value="Click Me" onclick="alert('Clicked')" /> `
  - 通过元素对象的onclick属性添加，此方式添加的事件，将onclick属性设置为null可以取消事件。如：
    ```js
        var btn = document.getElementById("myBtn");
        btn.onclick = function(){
        alert("Clicked");
        }; 
    ```
  - 通过元素对象的addEventListener方法添加，该方法接收三个参数，要添加事件的元素对象，事件函数，一个布尔值，true表示在事件捕获阶段调用事件处理程序，false表示在事件冒泡阶段调用处理程序。可添加多个同类型事件，按照事件的添加顺序执行。通过该方法添加的事件只能通过removeEventListener方法进行删除。该方法和添加事件的方法接受同样的参数。通过该方式添加的事件无法取消匿名函数事件。如：
    ```js
        var btn = document.getElementById("myBtn");
        var handler = function(){
            alert(this.id);
        };
        //添加事件
        btn.addEventListener("click", handler, false); 
        //移除事件，第二个参数需要一个函数对象名，所以不能取消匿名函数
        btn.removeEventListener("click", handler, false); 
    ```
## 事件对象
事件处理程序在触发时会被传入一个event对象，给对象包含与事件有关的信息。
![event对象1](/img/event1.png)
![event对象2](/img/event2.png)
## 事件类型
### UI事件
- load:当页面加载完成后触发的事件，也可以用在img元素上，表示一个图片加载完成。
- unload:当页面完全卸载后在window上面触发
- 其他一些事件。。。
### 焦点事件
- blur：在元素失去焦点时触发，该事件不会冒泡
- focus：元素获得焦点时触发，不会冒泡
- focusin:focus的冒泡版本
- focusout：blur的冒泡版本
### 鼠标事件
- click：单击鼠标或按下回车键触发
- dbclick：双击鼠标触发
- 其他事件。。。。
### 键盘事件
可以为document和html标签注册
- keydown:当用户按下键盘的**任意键**时触发，如果按住不放，则重复此事件
- keypress:当用户按下键盘的**字符键**时触发，如果按住不放，则重复此事件
- keyup:当用户释放键盘上的键时触发
- 当用户按下一个字符键时，事件的触发顺序为：keydown，keypress,keyup
- 当发生keydown和keyup事件时，event对象的keyCode属性会包含一个代码，表示按下的哪个键
### H5事件
- beforeunload：在页面卸载之前触发的事件，可以用来询问用户是否要离开等功能。
- haschange：当URL的参数列表以及URL中“#”后面的字符串发生变化时触发该事件，该事件必须添加给window对象，事件对象中包含oldURL和newURL属性，表示新旧URL
## 模拟事件
事件不是必须要用户操作来触发，也可以通过自定义的event对象来模拟触发。
- 通过document的 create Event（）方法可以生成一个event对象，该方法接受一个表示事件类型的字符串，如：MouseEvents:鼠标事件，UIEvents:UI事件。。。
- 返回的event对象有一个init方法，不同类型的事件该方法名称不同。通过调用该方法可以设置event的属性。
- 所有支持dom事件的节点都有dispatchEvent（）方法，该方法传入一个event对象。然后就成功模拟触发了该事件
# 表单脚本
## form的属性和方法
取得form对象的方法有多种：
- 通过document.getElementById()等类似方法取得
- document.forms可以取得页面中所有表单对象，返回一个集合，可以通过索引或name值来取得对应的form对象
- js获取到form元素的对象后可直接访问以下属性和方法：
  - acceptCharset：服务器能够处理的字符集；等价于 HTML 中的 accept-charset 特性。
  - action：接受请求的 URL；等价于 HTML 中的 action 特性。
  - elements：表单中所有控件的集合（HTMLCollection）。
  - enctype：请求的编码类型；等价于 HTML 中的 enctype 特性。
  - length：表单中控件的数量。
  - method：要发送的 HTTP 请求类型，通常是"get"或"post"；等价于 HTML 的 method 特性。
  - name：表单的名称；等价于 HTML 的 name 特性。
  - reset()：将所有表单域重置为默认值。
  - submit()：提交表单。
  - target：用于发送请求和接收响应的窗口名称；等价于 HTML 的 target 特性。
## 表单字段
- 可以使用DOM原生方法访问表单元素，也可以使用form对象的elements属性，该属性包含表单中所有表单元素的集合，可以通过索引和name特性取得相应的表单元素
- 表单字段共有的属性：
  - disabled：布尔值，表示当前字段是否被禁用。
  - form：指向当前字段所属表单的指针；只读。
  - name：当前字段的名称。
  - readOnly：布尔值，表示当前字段是否只读。
  - tabIndex：表示当前字段的切换（tab）序号。
  - type：当前字段的类型，如"checkbox"、"radio"，等等。
  - value：当前字段将被提交给服务器的值。对文件字段来说，这个属性是只读的，包含着文件
在计算机中的路径。
- 每个表单字段都有两种方法：
  - focus()：将浏览器的焦点设置到该表单字段
  - blur():将浏览器的焦点从该表单字段移走
- 表单字段支持的事件
  - blur:当前字段失去焦点时触发
  - change:对于`<input>`和`<textarea>`元素，在他们失去焦点且value值改变时才触发。对于`<select>`元素，在其选项改变时触发
  - focus:当前字段获得焦点时触发
## 选择框脚本
通过`<select>`和`<option>`元素创建。

- 选择框对象具有的一些属性和方法：
  - add(newOption, relOption)：向控件中插入新`<option>`元素，其位置在相关项（relOption）
  之前。
  - multiple：布尔值，表示是否允许多项选择；等价于 HTML 中的 multiple 特性。
  - options：控件中所有`<option>`元素的 HTMLCollection。
  - remove(index)：移除给定位置的选项。
  - selectedIndex：基于 0 的选中项的索引，如果没有选中项，则值为-1。对于支持多选的控件，
  只保存选中项中第一项的索引。
  - size：选择框中可见的行数；等价于 HTML 中的 size 特性。
- 选择框的 type 属性不是"select-one"，就是"select-multiple"，这取决于 HTML 代码中有没有 multiple 特性。
- 选择框的 value 属性由当前选中项决定，相应规则如下。
  - 如果没有选中的项，则选择框的 value 属性保存空字符串。
  - 如果有一个选中项，而且该项的 value 特性已经在 HTML 中指定，则选择框的 value 属性等于选中项的 value 特性。即使 value 特性的值是空字符串，也同样遵循此条规则。
  - 如果有一个选中项，但该项的 value 特性在 HTML 中未指定，则选择框的 value 属性等于该项的文本。
  - 如果有多个选中项，则选择框的 value 属性将依据前两条规则取得第一个选中项的值。
## 表单序列化
定义了一个函数，遍历form中的字段进行序列化
# JSON
json串与js的自面量的不同：js字面量定义对象时属性名可以加双引号也可以忽略，json中属性名必须添加双引号，单引号都不行。

## JSON的解析和序列化
- es5定义了全局JSON对象
- 将js对象转换为json串，JSON.stringify(),该方法生成的json串不包含对象的函数和原型成员而且忽略值为undefined的属性
- 将js串转化为js对象，JSON.parse(),当传入的json串不合法时会报错
## 序列化选项
- 过滤结果:JSON.stringify()方法中传入第二个参数，当传入的参数是个数组，里面的值就是要序列化的属性。当出入的参数是个函数时，该函数有两个参数key和value，通过循环调用该函数决定序列化结果。
  ```js
   var book = {
            name : "tomcat",
            price : 45,
            authors : ["xia","bao","xin"]
        }
        // {"name":"tomcat","price":45}
        var s1 = JSON.stringify(book,["name","price"]);
        // {"name":"jetty","price":45,"authors":["xia","bao","xin"]}
        var s = JSON.stringify(book,function (key,value) {
            switch (key) {
                case "name":
                    return "jetty";
            
                default:
                    return value;
            }
        });
  ```
- 字符串缩进：第三个参数设置结果的缩进，没啥用
## 解析选项
- JSON.parse()方法可以接受第二个参数，是一个函数，相当于js对象序列化中过滤结果的相反过程。
  ```js
        var book = {
            name : "tomcat",
            price : 45,
            authors : ["xia","bao","xin"]
        }
        var s1 = JSON.stringify(book);
        var b1 = JSON.parse(s1,function (key,value) {
            if (key == "name") {
                return "xiabaoxin";
            }else{
                return value;
            }
        })
  ```
# AJAX
  Ajax = Asynchronous JavaScript And XML
## XMLHttpRequest 对象
为什么叫这个名，因为之前的浏览器与客户端传递数据的格式是xml格式。

**基本用法**
- 新建一个xhr对象：`var xhr  = new XMLHttpRequest()` 
- 当创建完xhr对象以后调用的第一个方法是open（）方法，接受参数：请求类型（post，get）、请求的URL、是否为异步。如：`xhr.open("get","/example",false)`，调用此方法只是打开一个请求准备发送，并不会真的发送请求。
- 要发送请求必须调用send()方法，方法参数为要发送的数据，如果不要发送数据，则要传递参数null。如：`xhr.send(null)`
- 当浏览器收到响应后，响应的数据会自动填充到xhr对象的属性：
  - responseText：作为响应主体被返回的文本。
  - responseXML：如果响应的内容类型是"text/xml"或"application/xml"，这个属性中将保存包含着响应数据的 XML DOM 文档。
  - status：响应的 HTTP 状态码。
  - statusText：HTTP 状态的说明
- 收到响应后应首先检查status属性，以确定相应是否已经返回成功。
- 当xhr发送异步请求时，可以检测xhr的redayState属性，该属性在表示xhr所在的状态，每次该属性变化时，都会触发redaystatechange事件。为xhr注册redaystatechange事件要在调用open()方法之前，redaystate属性如下：
  - 0：未初始化。尚未调用 open()方法。
  - 1：启动。已经调用 open()方法，但尚未调用 send()方法。
  - 2：发送。已经调用 send()方法，但尚未接收到响应。
  - 3：接收。已经接收到部分响应数据。
  - 4：完成。已经接收到全部响应数据，而且已经可以在客户端使用了。
```js
        //实现一个同步的xhr请求
        var xhr = new XMLHttpRequest();
        xhr.open("get","./test2.html",false);
        xhr.send(null);
        if (xhr.status == 200) {
            console.log(xhr.responseText);
        }

        //实现一个异步xhr请求
        var xhr = new XMLHttpRequest();
        //注册状态改变的监听事件
        xhr.onreadystatechange = function () {
            if (xhr.readyState == 4) {
                if (xhr .status >= 200 && xhr.status < 300 || xhr.status == 304) {
                   console.log(xhr.responseText);
                }
            }
        }
        xhr.open("get","./test2.html",true);
        xhr.send(null);
```
## 设置HTTP头部信息和获取响应头
xhr在发送请求的时候也会发送相应的头部信息：
![](/img/requestHead.png)

- 设置xhr的请求头：调用xhr的open（）方法后可以调用setRequestHead（）方法可以设置相应的头部信息。
- 获取响应头：xhr的getReponseHeader()方法传入头部字段名称可以获得相应的响应头信息。xhr的getAllReponseHeaders()方法可以所有的响应头信息的文本格式。
```js
        //实现一个同步的xhr请求
        var xhr = new XMLHttpRequest();
        xhr.open("get","./test2.html",false);
        //设置请求头
        xhr.setRequestHeader("MyHead","hello");
        xhr.send(null);
        if (xhr.status == 200) {
            console.log(xhr.responseText);
        }
        //获取指定的响应头属性
        var MyHead = xhr.getResponseHeader("MyHead");
        //获取所有的响应头内容
        var allHeads = xhr.getAllResponseHeaders();
```
## get请求
在url后面直接添加参数，但是url后的参数需要用encodeURIComponent()方法进行编码
## post请求
使用post请求发送数据需要在send方法中传入要发送的数据。

但是使用post方式提交表单时需要设置Content-Type头部信息设置为 application/x-www-form-urlencoded

application/x-www-form-urlencoded对应的提交数据格式为：key1=val1&key2=val2

application/json对应的提交数据格式为json串

## FormData
为了简化xhr提交表单的过程。FormData类型可以序列化表单和创建与表单相同格式的数据。

当xhr的send()方法传入FormData对象时，无需设置Content-Type属性即可提交表单
```js
//第一种用法
var data = new FormData();
data.append("name","tom");
//直接序列化一个表单对象
var data2 = new FormData(document.forms[0]);
```
## 进度事件
XHR的六个进度事件：
- loadstart：在接收到响应数据的第一个字节时触发。
- progress：在接收响应期间持续不断地触发。该事件的处理程序收到一个event对象，该事件对象包含三个额外的属性：lengthComputable、position 和 totalSize。其中，lengthComputable是一个表示进度信息是否可用的布尔值，position 表示已经接收的字节数，totalSize 表示根据Content-Length 响应头部确定的预期字节数。
- error：在请求发生错误时触发。
- abort：在因为调用 abort()方法而终止连接时触发。
- load：在接收到完整的响应数据时触发。
- loadend：在通信完成或者触发 error、abort 或 load 事件后触发。

每个请求都从触发 loadstart 事件开始，接下来是一或多个progress 事件，然后触发 error、abort 或 load 事件中的一个，最后以触发 loadend 事件结束。

```js
var xhr = createXHR();
//设置xhr的load方法，在接收到完整的相应数据时触发
xhr.onload = function(event){
    if ((xhr.status >= 200 && xhr.status < 300) ||
    xhr.status == 304){
        alert(xhr.responseText);
    } else {
        alert("Request was unsuccessful: " + xhr.status);
    }
};
//设置xhr的进度事件，当进度变化时就会触发 
xhr.onprogress = function(event){
    var divStatus = document.getElementById("status");
    if (event.lengthComputable){
        divStatus.innerHTML = "Received " + event.position + " of " +event.totalSize +" bytes";
    }
};
xhr.open("get", "altevents.php", true);
xhr.send(null); 
```
## WebSocket
- 创建一个websocket连接:新建一个WebSocket对象，参数为服务端的websocket的地址。`var socket = new WebSocket("ws://www.xxx.com/hello")`,
- 调用close()方法则关闭websocket连接
- 发送和接收数据：send()用来发送数据。当服务端发来消息时则触发onmessage事件，该事件event对象的data属性包含了服务端推送过来的消息内容。
- open事件当ws连接建立成功时触发，error事件当ws发送错误时触发，close事件当连接关闭时触发。
```js
        //新建一个websocket连接
        var socket = new WebSocket("ws://www.xxx.com/hello");
        //连接建立成功时触发该方法
        socket.onopen = function () {
            console.log("success");
        }
        //连接关闭时触发该方法
        socket.onclose = function () {
            console.log("close");
        }
        //发生错误时触发该方法
        socket.onerror = function () {
            console.log("eroor");
        }
        //发送消息
        socket.send("hello world");
        //当服务端推送来消息时触发
        socket.onmessage = function (event) {
            //取得消息数据
            console.log(event.data);
        }
```
# 数据存储
## cookie
- cookie是一种浏览器的存储机制。
- 为浏览器设置cookie：服务器的响应头中可以包含一个set-cookie头里面的值就是要设置到浏览器中的值。格式为`Set-Cookie:name=value`
- 浏览器给服务器传递cookie：当服务器为浏览器设置了cookie后，浏览器之后的每个请求都会包含一个cookie请求头，格式为：`Cookie:name=value`
- cookie组成：
  - 名称：cookie的名称必须是经过URL编码的
  - 值：同样也要是URL编码的
  - 域：cookie对哪个域有效，所有向该域发送的请求都会包含这个cookie信息。如果没有明确指定，那么设置cookie的域就是cookie的域
  - 路径：对于指定域中的哪个路径，应该向服务器发送cookie。
  - 失效时间：表示cookie的过期时间的时间戳。
  - 安全标志：指定以后只有ssl链接时才可以发送该cookie
- cookie组成部分之间使用分号加空格进行隔开。示例：`Set-Cookie: name=value; expires=Mon, 22-Jan-07 07:10:24;  GMTdomain=.wrox.com; path=/; secure `
- document.cookie可以获取该页面可以使用的cookie。获取到的值是一个字符串：`name1=value1;name2=value2;name3=value3`
- 当用js设置cookie时可直接为document.cookie赋值，所存的名值应该时URL编码的。格式为：`document.cookie = encodeURIComponent("name") + "=" +encodeURIComponent("Nicholas") + "; domain=.wrox.com; path=/"; `
## 其他
除了cookie的存储方式，还有Storage，sessionStroage，localStratge，indexedDB。暂时用不到，走马观花看了一下。

# 新兴的API
## File API
可以读取文件系统中的文件。

将H5的拖放事件和文件API结合起来可以实现文件拖放到浏览器指定位置的效果。

当使用XHR上传文件时，可以使用FromData对象。