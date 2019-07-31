---
title: JAVA编程思想笔记
author: XIA
categories:
  - 笔记
tags:
  - JAVA
date: 2019-07-19 21:01:32
---

# 一切都是对象

- 尽管在java中一切都是对象，但操纵的操作符实际是对象的一个引用。例如使用`String s`创建的并不是对象，而是一个引用。
- 对象存放的位置：
  - 寄存器：位于cpu内部，是最快的储存区。由于寄存器的大小限制所以我们不能直接控制他，也无法感知到他的存在。
  - 堆栈：位于RAM，通过堆栈指针的上下移动来快速分配存储空间，java对象的引用就存储在这里。
  - 堆：位于RAM，用来存放对象。
  - 常量存储：通常直接存放在代码内部，因为它是一直不会改变的。
  - 非RAM存储：有些数据存活于程序之外，他们可以不受程序的控制，比如流对象和持久化对象等。他们保存在硬盘中。
- 基本数据类型不存储在堆中，他们直接存储在堆栈中，比如：boolean、inti、long等。

# 操作符

- 当使用直接常量时，可能会产生模棱两可的情况。此时需要在常量后面添加后缀标识符：L（l）代表long类型，F（f）代表float类型，D（d）代表double类型。

# 流程控制

- java中不允许使用数字来代表true和false。即不可以使用非零的数代表true，零代表false。

- 当在循环中使用标签时，使用continue和break关键字可以中断循环，直到标签所在的位置。

  ```java
  //输出的结果中没有i==2的值。
  		outer:
          for (int i = 0; i < 10; i++) {
              inner:
              for (int j = 0; j < 5; j++) {
                  if (i == 2){
                      continue outer;
                  }
                  System.out.println("i = "+i+",j = "+j);
              }
          }
  ```

# 初始化和清理

- 类的构造器必须要与类同名，且没有返回值。

- 区分方法的重载的依据就是方法必须要有独一无二的参数列表

- this关键字：当调用方法时可以看作“发送消息给对象”，但是在发送消息时编译器也作了一些幕后工作，就是将“所操作对象的引用”作为第一个参数传递给方法。如果想在方法中使用当前对象的引用则可以使用this关键字。

- 当在方法中调用当前类的其他方法时可以直接调用无需使用this关键字。

- 可以在构造方法中使用this()调用其他的构造方法，但是必须置于起始位置。

- static方法就是没有this的方法

- 垃圾清理主要讲了finalize方法的问题和垃圾收集的原理，这里留到JAVA虚拟机部分再整理。

- 成员初始化：

  - 类成员变量如果使用前未初始化编译器则会为其设置默认值，对于方法里的局部变量，编译器则会报错。
  - 在构造器被调用之前，类成员的初始化已经完成。

  ```java
  //i先为0,后为1
  public class Init {
      int i;
      public static void main(String[] args) {
          i = 1;
      }
  }
  ```

  - 在类的内部，变量定义的顺序就是初始化的顺序。即使变量定义在方法中间，它们也会在任何方法调用之前得到初始化。
  - 无论创建多少个对象static数据只占用一份存储区域。static关键字不允许用在局部变量。static修饰的变量初始化的值同非static修饰的值。
  - 静态代码块，在首次生成该类的对象或访问属于这个类的静态数据成员时，这段代码就会执行一次。而且只执行一次。
  - 非静态代码块，在每次创建对象时都会执行一次。

- 数组也是对象。数组对象有一个成员变量length，用来查看数组中元素的个数。

- 每个java源代码文件被称为一个编译单元。编译单元必须为.java后缀名。每个编译单元内必须有且只有一个public类，这个类要与编译单元的文件名相同（为了提高编译效率）。

- 默认权限为包访问权限，protected访问权限也提供包访问权限也提供子类的访问权限。
- 类访问权限只可以为public和默认包访问权限。

# 复用类

- 在子类中创建的对象会隐含的包含一个父类的对象。
- 组合就是在一个类中包含另一个类的对象。
- 选择继承还是组合时is-a和has-a的问题。另一个判断标准是是否需要向上转型。
- final关键字的两个场景：
  - 一个永不可变的编译时常量
  - 一个在运行时被初始化的值，而你不希望它被改变
- final允许进行留白，即在声明final时不给出初始值。只要在空白的final使用前被初始化即可。
- final也可以修饰方法中的参数
- final修饰方法，是为了将方法锁定，防止继承类修改他的含义。当方法用private修饰时也隐式的使用了final。因为他都无法取用了，当然也无法被覆盖了呀～～～
- final修饰类，表示这个类无法被继承。
- 子类不继承父类的私有成员
- 继承类的对象初始化顺序：
  - 首先初始化父类的static变量和static代码块，然后是子类的static变量和static代码块。
  - 然后是父类的成员变量，非静态代码块，父类构造方法执行。子类的成员变量初始化，非静态代码块，子类构造方法执行。

# 多态

- 方法调用时绑定：将一个方法调用同一个方法主体关联起来被称作绑定。若在程序执行前进行绑定叫做前期绑定，在运行时根据对象的类型进行绑定的被称作后期绑定或动态绑定或运行时绑定。

- java中除了static和final（private方法属于final方法）都是后期绑定的。

- 在类中将一个方法使用final声明可以防止其他人覆盖方法，也可以说成是关闭动态绑定。

- 只有普通的方法调用可以是多态的。

- 当在父类的构造器中调用多态方法时：

  ```java
  //父类
  public class A {
      void say(){
          System.out.println("A say()");
      }
      public A(){
          System.out.println("instance A before");
          //调用多态方法
          say();
          System.out.println("instance A after");
      }
  }
  //子类
  public class B extends A {
      private int i = 1;
      public B(int j){
          i = j;
          System.out.println("instance B "+i);
      }
      void say(){
          System.out.println("B say() "+i);
      }
  }
  
  public class Main {
      public static void main(String[] args) {
          new B(2);
      }
  }
  
  输出:
  instance A before
  B say() 0
  instance A after
  instance B 2
  ```

  这种情况可以用初始化的过程来解释：

  1. 在对象初始化之前将分配给对象的初始化空间初始化成二进制零
  2. 调用父类的构造器。调用被覆盖的方法B中的方法say()，因为第一步的原因i为0.
  3. 调用子类的构造器主体

- 编写构造器的一条原则：尽可能的使用简单的方法使对象进入正常状态，如果可以的话避免使用其他方法。

# 接口

- 将一个不含抽象方法的类使用abstract修饰后可以防止其创建对象
- 接口中的方法默认都是public的，如果省略则默认为public的，且必须为public 的。
- 接口可以进行多继承，一个接口可以继承多个接口。
- 使用接口的两个核心原因：为了能向上转型为多个基类型和防止创建创建该类的对象。
- 在接口中声明的字段默认都是static final的。接口中定义的域不允许出现“空final”

# 内部类

- 内部类就是将一个类定义在一个类的内部

- 当在外部类的非静态方法之外的任意位置创建某个内部类对象时，应这样指定对象的类型：OuterClassName.InnerClassName

- 内部类拥有一个外部类对象的引用，所以内部类拥有对其外部类所有成员的访问权

- 当需要外部类对象的引用时可以使用`OuterClassName.this`获取。

- 要创建内部类对象时，必须拥有外部类的对象。根据外部类对象创建内部类对象时使用`.new`

  ```java
  public class Outer{
      public class Inner{}
      public static void main(String[] args){
          Outer outer = new Outer();
          //创建内部类对象
          Outer.Inner inner = outer.new Inner();
      }
  }
  ```

- 局部内部类就是在方法中定义的类。局部内部类不能有访问控制符，因为他不是外部类的一部分。但是他可以访问当前代码块内的常量和外部类的所有成员。

- 匿名内部类就相当于新建一个接口或抽象类的具体实现的对象，或具体的类的子类的对象，然后使用多态使其返回接口或抽象类的引用，因为中间的实现类没有名字，所以叫做匿名内部类。

- 当定义一个匿名内部类，并且希望他使用一个在其外部定义的对象，那么编译器会要求其参数引用为final

- 普通内部对象都和其外部类对象间存在联系，如果不需要其存在联系可以使用static修饰内部类。这样的内部类叫做静态内部类或嵌套类。

- 普通内部类中不可以包含static修饰的数据字段。

- 接口中也可以包含内部类，默认为public static修饰的。

- 有点晕。。。。。。

# 持有对象

- 自javase5之后引入了泛型

- 可以通过Arrays和collections类中的方法添加一组元素

- 迭代器的设计模式，遍历序列中的对象，而不必关心该序列底层的结构。Iterator只能正向遍历，而ListIterator既可以正向遍历也可以反向遍历。

- java1.0的stack类的设计者犯了一个错误，他将Stack类继承了Vector类，导致添加了很多无用的方法。

- HashSet由于哈希算法的原因插入的元素会失去顺序。TreeSet底层红黑树实现，可以在插入的时候进行排序，排序规则为指定的Comparator方法。LinkedHashSet顺序为插入的顺序。

- Map

- Collection接口继承了Ierator接口，所以所有collection都可以使用迭代器

- javase5增加了foreach语法，只要类实现了Interable接口，那么这个类的对象就可以使用foreach语法。

  ```java
  //实现Iterable接口
  public class IterableClass implements Iterable {
      int[] arr ={1,8,9,6,5,1,9,0,3};
      @Override
      public Iterator iterator() {
          return new Iterator() {
              private int index = 0;
              @Override
              public boolean hasNext() {
                  return arr.length > index;
              }
              @Override
              public Object next() {
                  return arr[index++];
              }
          };
      }
  }
  
  //使用foreach语法
  public class Main {
      public static void main(String[] args) {
          IterableClass iterableClass = new IterableClass();
          for (Object aClass : iterableClass) {
              System.out.println(aClass);
          }
      }
  }
  ```

# 异常

- 所有的标准异常类一般包含两个构造器：一个默认构造器，一个接受字符串作为参数的构造器
- 通过继承Exception类可以自定义异常
- 异常声明，当方法中有跑出异常时，可以在方法上对要抛出的异常进行声明。也可以在抽象类或接口的方法上声明异常，这样可以这样可以先占个位子，在以后实现这些方法时候不用再去修改已有的代码。
- 在catch捕获到异常后可以重新将其抛出。
- 异常链可以将捕获到的异常重新封装到一个新的异常中，并重新抛出。异常信息中`Caused by`部分就是这么来的
- RunTimeException是`不受检查异常`，而其他的异常是`受检查异常`
- 异常时覆盖的注意事项：

    - 子类在覆盖父类方法时，父类方法抛出异常，那么只能抛出的异常只能抛出父类异常或父类异常的子类或不抛出异常。
    - 如果父类抛出多个异常，那么子类只能抛出父类的子集。
    - 如果父类的方法没有抛出异常，那么子类覆盖时决不能抛出，只有try。

# 字符串

- String对象是不可变的。String中用来操纵内容的方法最后返回的都是一个新的String对象，原来的对象并不会改变。

- Java重载了“+”操作符，使用“+”连接符对字符串进行连接实际上是调用了StringBuilder类。

  ```java
     //使用‘+’，在每个循环中都会创建一个StringBuilder对象
  	public String implicit(String...strings){
          String result="";
          for (String string : strings) {
              result+= string;
          }
          return result;
      }
  	//只创建一个StringBuilder对象
      public String explicit(String...strings){
          StringBuilder result = new StringBuilder();
          for (String string : strings) {
              result.append(string);
          }
          return result.toString();
      }
  ```

- 无意识的递归：在toString()方法中使用this关键字，this也会点用toSting()方法，所以就形成了递归。

- 格式化工具：想c语言中printf()函数，根据占位符进行填充。

  - System.out.format()：

    ```java
            String s="xia";
            System.out.format("%s is nice", s);
    		//out
    		xia is nice
    ```

  - Formatter类：构造时可传入输出路径。输出时还可以对个是进行设置（缩进、对其等）

    ```java
            Formatter f = new Formatter(System.out);
            String s="xia";
            f.format("%s is a great boy!",s);
    		//out
    		xia is a great boy!
    ```

    | 说明符 | 适用于                                                  | 输出                                                         |
    | :----- | :------------------------------------------------------ | :----------------------------------------------------------- |                                                             |
    | %b     | boolean                                                 | 如果为非空则为“true”，为空则为“false”                        |
    | %c     | 字符                                                    | Unicode字符                                                  |
    | %d     | 整型(包括byte, short, int, long, bigint)                | 十进制整数                                                   |
    | %e     | 浮点数                                                  | 科学计数的十进制数                                           |
    | %f     | 浮点数                                                  | 十进制数                                                     |
    | %g     | 浮点数                                                  | 十进制数，根据值和精度可能以科学计数法显示                   |
    | %h     | 任何类型                                                | 通过hashCode()方法输出的16进制数                             |
    | %n     | 无                                                      | 平台相关的换行符                                             |
    | %o     | 整数(包括byte, short, int, long, bigint)                | 八进制数                                                     |
    | %s     | 任何类型                                                | 字符串                                                       |
    | %t     | 日期/时间 (包含long, Calendar, Date 和TemporalAccessor) | %t是日期/时间转换的前缀。后面还需要跟其他的标识，请参考下面的日期/时间转换。 |
    | %x     | 整数(包含byte, short, int, long, bigint)                | 十六进制字符串                                               |
    
  - String.format():同上面的格式化类似，该方法返回一个字符串对象
  
    ```java
            String s="xia";
            System.out.println(String.format("%s is nice", s));
    ```
  
- 正则表达式，待这本书看完单独拿出一天整理

- 扫描输入相关：Scanner类

# 类型信息

- RTTI（Runtime Type Information，运行时类型信息。

  > 我们经常讲运行时类型判定即RTTI有两种形式：（1）传统的RTTI;（2）反射reflection机制。RTTI的初期想法非常简单：当有一个指向基础型别（父类）的reference（引用）时，RTTI机制让你找出其所指的确切型别，不过当拓展到java.lang.reflection的时候,展现了全新的功能。这里我们可以这样理解：RTTI是一种思想，Java中多态和反射的使用都利用了这一思想。什么时候用到传统的RTTI呢？我认为是在使用多态的时候，Java的所有方法绑定都采用“后期绑定”技术，若一种语言实现了后期绑定，那么同时还要提供一些机制，以便在运行时间正确判断对象类型，并调用适当的方法。也就是说，编译器此时仍然不知道对象的类型，但方法调用机制能自己去调查，找到正确的方法主体。反射是RTTI发展产生的概念和技术，反射的作用是分析类的结构，并可以创建类的对象。总结为一句话：多态是隐式地利用RTTI，反射则是显式地使用RTTI。

- 生成类的class对象使用了JVM中的‘类加载器’。所有的类都是在其第一次使用时，动态的加载到JVM中。当程序创建第一个对类的静态成员（包括使用new对构造方法的调用）引用时，就会加载这个类。

- 创建类的对象可以使用：

  + Class.forName("完整类名")
  + object.getClass():一个类的实例对象获取类的对象
  + ClassName.class:类明.class就可以获取这个类的对象。这中方法不仅适用于普通的类，还适用于接口、数组、基本数据类型。

- 当使用“.class”创建Class对象时不会初始化该Class对象。通常使用类所做的准备工作包含三部分：

  1. 加载：由类加载器完成。该步骤将查找字节码，从字节码中创建Class对象。
  2. 链接：验证类中的字节码，为静态域分配存储空间，如果需要的话将解析这个类创建的对其他类的所有引用
  3. 初始化：如果该类具有父类，则对其初始化，执行静态初始化器和静态初始化块。

- 当使用“.class”语法获取对类的引用时不会发生初始化，但是使用Class.forName()将立即进行初始化。

- 如果一个static final字段的值是一个“编译期”常量，那么这个值不需要初始化就可以读取。但是当他不是一个“编译期”常量时就需要初始化类以后才可以读取

- 如果一个字段是static而不是final。那么在读取之前需要先进行链接和初始化。

- 可以使用泛型对Class对象进行限定

- 当进行向下转型时，为防止转型中报错。可以使用instaceof运算符或isInstance()方法。

  ```java
  class A{}
  class B extends A{}
  class Main{
     A a = new B();
     System.out.println(a instanceof B);
     System.out.println(A.class.isInstance(a));
  }
  ```

- 反射：传统的RTTI需要在编译时就要知道class文件内容。而反射则可以在运行时通过本地文件、网络加载class文件。反射工具可以获取class文件中包含的方法，构造，字段等信息。

- 动态代理：[动态代理](../../27/动态代理/)

# 泛型

- 简单泛型：

  ````java
  public class Animal<T> {
      private T name;
      public Animal(T name) {
          this.name = name;
      }
      @Override
      public String toString() {
          return "Animal{" +
                  "name=" + name +
                  '}';
      }
  }
  
  public class Main {
      public static void main(String[] args) {
          Animal<String> animal = new Animal<String>("dog");
          System.out.println(animal);
      }
  }
  ````

- 泛型接口：泛型也可以作用于接口。

  ```java
  //定义泛型接口，一个对象生成器接口
  public interface Generator<T> {
      T next();
  }
  ```

- 泛型方法：泛型方法可以独立于类存在，即无论泛型方法所在的类是不是泛型类都可以存在泛型方法。另外对于static方法而言，无法访问泛型类的类型参数，所以static方法需要使用泛型能力时就必须使用泛型方法。泛型方法的使用不像泛型类那样每次都要到泛型参数进行赋值，泛型方法可以根据方法调用情况进行泛型参数的推断。

  ````java
  public class MyMethod {
      //定义泛型方法
      public <T> void f(T t){
          System.out.println(t.getClass().getSimpleName());
      }
  
      public static void main(String[] args) {
          MyMethod myMethod = new MyMethod();
          //自动泛型类型的推断
          myMethod.f("hello");
      }
  }
  ````

- 泛型类型推断的三种方式：

  + 根据赋值操作对泛型进行推断
  + 根据方法调用进行推断
  + 显示指定

  ````java
  public class New {
      public static <K,V> Map<K,V> map(){
          return new HashMap<K,V>();
      }
      public static <K> List<K> list(){
          return new ArrayList<K>();
      }
      public static Map<String,Integer> reMap(Map<String,Integer> map){
          return map;
      }
      public static void main(String[] args) {
          //根据赋值操作对泛型进行推断
          Map<String, Integer> map = New.map();
          List<String> list = New.list();
          //根据方法调用传递参数进行类型推断
          New.reMap(New.map());
          //显示指定类型
          New.<String,Integer>map();
      }
  }
  ````

- 泛型擦除：在Java中擦除是实现泛型的方式。在编译结束后，泛型将被擦除到它的第一边界（当泛型定义边界时将擦除到边界，没有定义时将擦除到Object）。擦除到边界时所产生的效果就像是边界代替了实际的泛型，在下面例子中就是HasF替代了T。那这样还要泛型干啥？直接多态不就可以了吗？我觉得使用泛型主要是可以指定确切的类，而避免多态带来的“模糊性”。

  ````java
  public class HasF {
      public void f(){};
  }
  
  public class Main<T extends HasF> {
      private T obj;
      public void main(){
          //若不定义边界，则无法使用该方法，因为编译结束后泛型已经被擦除了
          obj.f();
      }
  }
  ````

  java使用擦除实现泛型看书上写的一页也没搞懂，应该就是为了向下兼容吧。

  我现在理解的泛型，其实功能很局限，也就是做做编译时的类型检验而已。一旦编译完成被擦除若不设置边界还不如多态来的实在。

  ```java
  public static void main(String[] args) {
      Generic<Integer> genericInteger = new Generic<>(123);
      Generic<String> genericString= new Generic<>("Hello");
      System.out.println(genericInteger.getClass() == genericString.getClass());  // 返回 true
  }
  ```

  结果返回`true`，说明虽然编译时`Generic<Integer>`和`Generic<String>`是不同的类型，但因为泛型的类型擦除，所以编译后`genericInteger`和`genericString`为相同的类型

- 边界：泛型定义的边界有两个作用：1. 强制规定泛型可以应用的类型。2.由于java的泛型擦除，所以没有设置边界的泛型只能调用Object的方法，设置边界后可以按照自己定义的边界来调用方法。extends关键字表示泛型类应继承自后面的类。

  ```java
  public interface Animal {
      void eat();
      void say();
  }
  //接口实现类
  public class Mammal implements Animal {
      @Override
      public void eat() {
          System.out.println("mammal eat...");
      }
  
      @Override
      public void say() {
          System.out.println("mammal say...");
      }
      public void walk(){
          System.out.println("mammal walk...");
      }
  }
  //包含泛型你方法的类
  public class Main {
      public static <T extends Animal> void f(T t){
          //由于设置了边界，所以可以调用边界类中的方法
          t.eat();
          t.say();
      }
  
      public static void main(String[] args) {
          f(new Mammal());
      }
  }
  ```

- 通配符（待复习，这一节有点虎）：

  - 数组具有协变性，这也就印证了数组的类型检查发生在程序的运行期。而泛型的检查是在编译期间。
  - 通配符是在使用泛型是指定的，使用“?”。而边界是在定义时使用的。
  - 通配符有点类似于对编译器进行类型检查造成的协变性丢失的一点补充。也有点多态的味道。
  - < ? extends MyClass >表示泛型所代表的类都是继承自MyClass。使用这种声明列表的引用无法进行增加操作，因为编译时只是知道引用所表示的对象列表的内容都是继承自MyClass，但是无法知道具体是那个实现，如果贸然进行添加会导致类型转化异常。
  - < ? super MyClass >表示泛型所代表的类都是MyClass的基类，这样声明的引用无法进行查看但是可以进行添加。

# 数组

- 数组的效率比List要高

- length只表示数组的大小，而不是实际保存的元素个数

- 不能实例化具有参数化类型的数组：

  ```java
  Peel<Banana> peels = new Peel<Banana>[10];   //这是不合法的
  ```

- Arrays.fill()可以对数组进行数据填充

- System.arraycopy()可以对数组进行复制，但是在复制对象时只是复制了对象的引用而不是对象本身，这种方式被称为“潜复制”。

- Arrays.equals()方法可以对数组进行比较，当数组长度相同，且里面的元素也都相同（元素间的比较也是调用元素对象的equals方法）时即返回true。

- Arrays.sort()方法可以对数组进行排序，要排序的对象要么实现comparable接口，要么需要在方法中传入comparator接口的实现类。

- 对已排序的数组使用Arrays.binarySearch()进行二分查找。















参考：

> 数组的协变性和泛型的不可变性：<https://blog.csdn.net/magi1201/article/details/45127487>















