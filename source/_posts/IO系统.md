---
title: IO系统
author: XIA
categories:
  - null
tags:
  - null
date: 2018-04-26 20:35:06
---

# File类

File类既可以表示一个文件也可以表示一个文件夹。在表示文件夹时可以使用list()方法返回一个字符数组，该方法还可以接受一个FilenameFilter类型对象对文件名做过滤。

File类提供了丰富的api，例如创建文件或文件夹、检测是否为文件等。

# 输入和输出

Java类库种的IO分为输入和输出两部分。任何自InputStream或Reader派生而来的类都有read()方法，用于读取。任何自OutputStream或Writer派生来的类都含有write()方法，用于写。

Java中的IO类库很多用到了装饰器模式，这使得创建一个流对象，却需要创建多个别的对象。

在Java1.0中，类库的设计者限定与输入流有关的所有类都应该从InputStream继承，与输出有关的所有类都应该从OutputStream继承。

## InputStream类型

InputStream表示那些从不同数据源产生输出的类，这些数据源包括：字节数组、String对象、文件、网络等。每一种数据源都有与之对应的InputStream子类。

![image-20200427194230995](https://xbxblog2.bj.bcebos.com/IO%2Fimage-20200427194230995.png)

## OutputStream类型

该类决定输出所要去往的目标：字节数组、文件等。

![image-20200427194922853](https://xbxblog2.bj.bcebos.com/IO%2Fimage-20200427194922853.png)

## FilterInputStream与FilterOutputStream

这两个类派生自InputStream与OutputStream。使用装饰器模式，内部封装一个InputStream或OutputStream实现类。

因为InputStream与OutputStream的子类大多只是与特定的输入或输出位置有关，传输与处理的单位都是字节，而且每次的读写操作都会对应到实际在磁盘上的读写。

FilterInputStream则提供了更方便使用的方法，比如，DataInputStream：读取一个int型数据、BufferedInputStream：在读取时使用缓冲区来提高效率等。FilterOutputStream提供了在写方面更方便的子类，比如：DataOutPutStream可以直接写int型数据、字符串等，PrientStream可以直接写一行数据等。

**FilterInputStream主要实现类**

![image-20200427201323772](https://xbxblog2.bj.bcebos.com/IO%2Fimage-20200427201323772.png)

**FilterOutPutStream主要实现类**

![image-20200427201632650](https://xbxblog2.bj.bcebos.com/IO%2Fimage-20200427201632650.png)

**小结**

输入输出的顶层父类是InputStream与OutputStream，这两个类的直接子类大多表示输入或输出目的地，其处理传输的单位都是字节，比如：FileInputStream。

InputStream与OutputStream有一个比较特殊的子类FIlterInputStream与FilterOutputStream，他们使用装饰器模式对InputStream与OutputStream的直接子类进行包装，提供更方便使用的功能。

## Read与Writer

Java1.1增加了Read与Writer类，这两个类是面向字符操作的类。需要注意的是，这两个类与InputStream、OutputStrem一样都是顶层类。他们可以看作两套独立的系统，一个面向字节、一个面向字符。

在处理字符型数据时应尽量使用字符流。有时我们需要将字节流转换为字符流使用，这时需要使用适配器类，InputStreamReader可以把InputStream转换为Reader，OutputStreamWriter可以把OutputStream转换为Writer。

**InputStream、OutputStream与Read、Writer对应关系**

![image-20200427210545148](https://xbxblog2.bj.bcebos.com/IO%2Fimage-20200427210545148.png)

在InputStream、OutputStream的字节流结构中有一个用于提供更方便方法的包装器类FilterInputStream与FilterOutputStream类。在Read、Writer结构的字符流中这种结构做了一些调整。粗略的对应关系如下：

![image-20200427211251323](https://xbxblog2.bj.bcebos.com/IO%2Fimage-20200427211251323.png)

**Read与Writer类结构**

![image-20200427211456843](https://xbxblog2.bj.bcebos.com/IO%2Fimage-20200427211456843.png)

## RandomAccessFile

适用于已知大小的文件，可以用seek()方法将记录从一处转移到另一处，然后读取或修改记录。

RandomAccessFile不是InputStream与OutputStream继承层次结构中的一部分。
