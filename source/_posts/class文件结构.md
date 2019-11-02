---
title: class文件结构
author: XIA
categories:
  - jvm
tags:
  - jvm
date: 2019-10-30 20:47:07
---

# 概述

java文件通过编译命令javac编译为class文件，class文件直接在虚拟机上运行。正是因为jvm直接运行class文件，其他语言经过相应的编译器编译后都可以获取到相应的class文件，然后运行在jvm上，像kotlin、groovy等。所以class文件是jvm语言无关性的基石。

class文件是一组以8位字节位基础单位的二进制流，各个数据项目之间严格按照顺序排列，中间没有分隔符。

class文件只有两种数据结构：无符号数和表。

无符号数属于基本数据类型，以u1、u2、u4、u8分别表示1个字节、2个字节、4个字节、8个字节的无符号数。表是由多个无符号数构成的符合数据结构，通常我们说起表会联想到纵列的表格，其实这里的表更像是一个Map数据类型，class文件规范定义了这个Map的键和值的特定含义。表结构一般以_info结尾。

**class文件结构：**

![](https://xbxblog.bj.bcebos.com/classStructure.png)

# 示例代码

我们准备一个简单的java类，然后通过javac命令编译位class文件，使用notepad++打开。notepad++如果要查看16进制需要安装HEX-Editor插件。

```java
public class Animal{
	private int age;
	public int addAge(){
		return age++;
	}
}
```

然后编译为class文件后使用notepad++打开

![classHex](https://xbxblog.bj.bcebos.com/classHex.png)

# 魔数和class文件版本

class文件的前4个字节称为魔数，用来确定这个文件能否被虚拟机接受。可以看到class文件的魔数为cafebabe（咖啡宝贝）。

魔数后面的4个字节表示class文件的版本信息，其中前两个字节表示次版本号，后两个字节表示主版本号。

![classVersionNum](https://xbxblog.bj.bcebos.com/classVersionNum.png)

# 常量池

常量池是class文件中的资源仓库，而且常量池是一个表结构。

由于常量池中常量数量不是固定的，所以在常量池入口有一个u2类型的数据表示常量池中常量的数量。该值是从1开始计数的，如上面示例代码代码的class文件这个值为0x0013，转化为十进制就是19，所以一共有18个常量。

常量池中的常量分为两种：字面量和符号引用。字面量就是字符串或声明为final的常量值等，符号引用属于编译原理的概念，包括：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符。java在编译时不会像C++那样有连接的步骤，而需要在类创建时或运行时解析、翻译到具体内存地址。

常量池中每一项常量都是一个表，每个常量的第一位都是一个u1类型的标志位，代表当前这个常量属于哪种类型。

![](https://xbxblog.bj.bcebos.com/%E5%B8%B8%E9%87%8F%E7%B1%BB%E5%9E%8B.png)

根据这个表我们来尝试分析以下实例代码的常量池

![](https://xbxblog.bj.bcebos.com/%E5%B8%B8%E9%87%8F%E5%AE%9E%E4%BE%8B.png)

如上第一个常量的tag为0a，对应为CONSTANT_Methodref_info类型，其结构包括两个2bit的索引，0004和000f，有了索引再根据索引找到常量池中对应的值。javap命令可以直接查看class文件的字节码内容。

可以看到常量池中的数量与每个常量的内容。

![](https://xbxblog.bj.bcebos.com/javap.png)

# 访问标志

在常量池结束后紧跟着两个为访问标识符，用来标志类或者接口层次的一些信息。包括该class是类还是接口、是否为public、是否为abstract类型等。

在class文件中寻找着两个字节的技巧：根据常量池中最后一个字面量的值然后根据javap的结构对照着找到常量池的结尾。

|    标志名称    | 标志值 |                             描述                             |
| :------------: | :----: | :----------------------------------------------------------: |
|   ACC_PUBLIC   | 0x0001 |                       是否为public类型                       |
|   ACC_FINAL    | 0x0010 |                       是否为final类型                        |
|   ACC_SUPER    | 0x0020 | 是否允许使用invokespcial字节码指令的新语义，jdk1.0.2之后编译出来的类，此标志都为真 |
| ACC_INTERFACE  | 0x0200 |                          是否为接口                          |
|  ACC_ABSTRACT  | 0x0400 |     是否为abstract类型(对接口和抽象类来说，此标志都为真)     |
| ACC_SYNTHETIC  | 0x1000 |                 标识这个类并非由用户代码产生                 |
| ACC_ANNOTATION | 0x2000 |                          是否是注解                          |
|    ACC_ENUM    | 0x4000 |                          是否是枚举                          |

在实例代码中访问标志位0x0021，对应为public类型。

# 类索引、父类索引、接口索引集合

这三个索引用来描述类的层次关系，包括类的全限定名称、父类名称、实现的接口名称。

类索引和父类索引是一个u2类型的数据，接口索引集合在入口有一个u2类型的计数器，用来对实现的接口进行计数，当没有实现接口时则为0。

示例代码中这节个数据为 `00 03 00 04 00 00`，对应类的全限定名为索引3，父类名为索引04，没有实现接口。

# 字段表集合

字段表集合用来描述class文件中声明的变量,只包含当前类的不包含父类中的字段。字段表集合的开头有一个u2类型的数据用来表示字段表的大小。

接着是字段表的结构：

|      类型      |       名称       |       数量       |                含义                |
| :------------: | :--------------: | :--------------: | :--------------------------------: |
|       u2       |   access_flags   |        1         |             字段修饰符             |
|       u2       |    name_index    |        1         | 字段和方法简单名称在常量池中的引用 |
|       u2       | descriptor_index |        1         |  字段和方法描述符在常量池中的引用  |
|       u2       | attributes_count |        1         |     描述字段额外信息属性的个数     |
| attribute_info |    attributes    | attributes_count |     具体描述字段的额外信息属性     |

**access_flags**表示字段的描述符信息，如访问修饰符、是否为静态类型等：

|   标志名称    | 标志值 |         描述         |
| :-----------: | :----: | :------------------: |
|  ACC_PUBLIC   | 0x0001 |   是否为public类型   |
|  ACC_PRIVATE  | 0x0002 |  是否为private类型   |
| ACC_PROTECTED | 0x0004 | 是否为protected类型  |
|  ACC_STATIC   | 0x0008 |   是否为static类型   |
|   ACC_FINAL   | 0x0010 |   是否为final类型    |
| ACC_VOLATILE  | 0x0040 |   是否volatile类型   |
| ACC_TRANSIENT | 0x0080 |  是否transient类型   |
| ACC_SYNTHETIC | 0x1000 | 是否由编译器自动产生 |
|   ACC_ENUM    | 0x4000 |     是否enum类型     |

**name_index**表示字段的名称

**descriptor_index**表示字段的数据类型，class文件中数据类型的对应规则：

| 描述符 |             含义              |
| :----: | :---------------------------: |
|   B    |         基本类型byte          |
|   C    |         基本类型char          |
|   D    |        基本类型double         |
|   F    |         基本类型float         |
|   I    |          基本类型int          |
|   J    |         基本类型long          |
|   S    |         基本类型short         |
|   Z    |        基本类型boolean        |
|   V    |         基本类型void          |
|   L    | 对象类型，如Ljava/lang/Object |

如果是数组类型，每一个维度用一个`[`表示。如：`[[Ljava/lang/String`表示`java.lang.String[]][]`。

如示例代码的这部分为`00 01 00 02 00 05 00 06 00 00 `表示只有一个字段，访问标识符为private，字段名的索引为常量池中第5个常量，描述索引为常量池中第6个索引，属性集合长度为0。

# 方法表集合

与字段表集合类似，也包括访问标志、名称索引、描述符索引、属性表集合这几项。

对方法描述的方式与对字段描述的方式基本一致，方法表的结构也与字段表的结构完全一致，不同之处在于方法的访问标志与字段的访问标志有所区别。例如volatile与transient不能修饰方法，但是方法却有synchronized、native、strictfp和abstract等属性。其具体访问标志如下：

|     标志名称     | 标志值 |         描述         |
| :--------------: | :----: | :------------------: |
|    ACC_PUBLIC    | 0x0001 |   是否为public类型   |
|   ACC_PRIVATE    | 0x0002 |  是否为private类型   |
|  ACC_PROTECTED   | 0x0004 | 是否为protected类型  |
|    ACC_STATIC    | 0x0008 |   是否为static类型   |
|    ACC_FINAL     | 0x0010 |   是否为final类型    |
| ACC_SYNCHRONIZED | 0x0020 | 是否synchronized类型 |
|    ACC_BRIDGE    | 0x0040 |     是否桥接方法     |
|   ACC_VARARGS    | 0x0080 |   是否接收不定参数   |
|    ACC_NATIVE    | 0x0100 |    是否native方法    |
|   ACC_ABSTRACT   | 0x0400 |     是否abstract     |
|   ACC_STRICTFP   | 0x0800 |     是否strictfp     |
|  ACC_SYNTHETIC   | 0x1000 | 是否由编译器自动产生 |

描述符索引在表示方法时，按照先参数，后返回值的顺序描述，参数列表按照参数的顺序方法`()`中。

在方法表集合中有一个隐藏方法`<init>`，对应源码中的构造方法。

方法表集合中只是对方法的声明进行了保存，方法的执行命令则保存在方法的属性集合的code属性中。

方法表集合同样也不会保存父类中的方法。

`00 02 00 01 00 07 00 01`表示有两个方法。第一个方法的访问标志为public、方法名为`<init>`，方法描述为()V。第一个方法的属性表集合中有一个属性。接下来就是属性表集合了~

# 属性表集合

在字段表和方法表的结构中都有一项attributes，这一项就是属性表。

属性表并不像其他数据项目由严格的要求，只要不与已有属性名重复，编译器可以向属性表中写入自己定义的属性信息。但是java虚拟机规范中还是确定了21项预定义属性信息。一些常用的如：

|      属性名称      |      使用位置      |               含义               |
| :----------------: | :----------------: | :------------------------------: |
|        Code        |       方法表       |    java代码编译成的字节码指令    |
|   ConstantValue    |       字段表       |     final关键字定义的常量值      |
|     Deprecated     | 类、字段表、方法表 |  被声明为deprecated的方法和字段  |
|    InnerClasses    |       类文件       |            内部类列表            |
|     Exception      |       方法表       |          方法抛出的异常          |
|  LineNumberTable   |      Code属性      | 源代码行号与字节码指令的对应关系 |
| LocalVariableTable |     Codes属性      |        方法的局部变量描述        |
|     Signature      | 类、字段表、方法表 |           记录泛型信息           |

对于每个属性，它的名称都需要从常量池中引用一个常量来表示，属性值则是自定义的，只需要通过一个u4长度的长度属性说明即可。

| 类型 |         名称         |       数量       |
| :--: | :------------------: | :--------------: |
|  u2  | attribute_name_index |        1         |
|  u4  |   attribute_length   |        1         |
|  u2  |         info         | attribute_length |

**Code属性**

* Code属性的属性信息包含最大操作数栈、编译后的字节码指令等。字节码指令通过一个u1类型的单字节表示，然后虚拟机根据该字节的表示找出对应的操作指令

**ConstantValue属性**

当一个字段被static修饰时才会使用这个属性。对于实例变量的赋值是在实例构造器`<init>`中，而类变量则是在类构造器`<clinit>`方法或使用ConstantValue属性。javac规定当一个基本类型或字符串变量使用static和final修饰时就使用ConstantValue属性进行初始化。