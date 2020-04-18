---
layout: post
title: 翻译Introduction to Java Bytecode
date: 2020-04-18
categories: jvm
tags: [java,jvm]
description: 翻译文章。
---


## 翻译一些自己喜欢的文章,由于能力有限，敬请见谅。
## [原文地址](https://dzone.com/articles/introduction-to-java-bytecode)

Java字节码简介</br>
Introduction to Java Bytecode</br>

继续深入研究JVM内部和Java字节码，以了解如何分解文件以进行深入检查。</br>
Follow along this deep dive into JVM internals and Java bytecode to see how you 
can disassemble your files for in-depth inspections.</br>

读编译后的字节码是很乏味的，但是对Java开发者很有用。为什么还要第一时间学习这些低级内容呢？这是我上周发生的情况，我在我的电脑上对一些代码进行了变更，
编译成Jar并部署到服务器去测试一个潜在的性能问题。不幸得是这些代码没有提交到版本控制系统，
出于一些其他原因，我电脑对代码的改变被删除了，并且找不到轨迹，过了几个月后，我需要在源代码上进行变更(这些代码花了很大力气编写),但是我找不到它们了。</br>

Reading compiled Java bytecode can be tedious, even for experienced Java developers. 
Why do we need to know about such low-level stuff in the first place? 
Here is a simple scenario that happened to me last week: I had made some code changes on my machine a long time ago, 
compiled a JAR, and deployed it on a server to test a potential fix for a performance issue. Unfortunately, 
the code was never checked into a version control system, and for whatever reason, the local changes were deleted without a trace.
After a couple of months, I needed those changes in source form again (which took quite an effort to come up with), 
but I could not find them!</br>

幸运的是这些被编译的代码还存在远程的服务器上面，所以我送了一口了气，我从服务器拉下了Jar文件并用反编译工具打开它。
有一个问题：反编译GUI不是一个完美的工具，而且有很多Classes没有在jar文件中，
出于某些原因，当我打开特定的class去看反编译代码时引起了UI的一个bug,然后我把反编译工具丢进了垃圾箱！</br>
Luckily the compiled code still existed on that remote server. So with a sigh of relief, I fetched the JAR again and opened it using a decompiler editor... 
Only one problem: The decompiler GUI is not a flawless tool, and out of the many classes in that JAR, 
for some reason, only the specific class I was looking to decompile caused a bug in the UI whenever I opened it, and the decompiler to crash!</br>

绝望的时候采取绝望的措施，幸好，我熟悉原生字节码，而且我宁愿话费一些时间手动反编译这些代码片段而不是研究更改并再次测试它们。
由于我还记得这些代码位置，读取字节码有助于我找到明确的更改并再次构造到源代码（我确保从错误中吸取教训，并保护它们）。</br>
Desperate times call for desperate measures. Fortunately, I was familiar with raw bytecode, and I'd rather take some time manually decompiling some pieces of the code rather than work through the changes and testing them again. 
Since I still remembered at least where to look in the code, reading bytecode helped me pinpoint the exact changes and construct them back in source form. (I made sure to learn from my mistake and preserve them this time!)</br>

字节码的好处是你学习它的语法一次，适用于Java直接的平台，因为它是代码的中间表示，而不是cpu下执行的机器语言，
此外，字节码由于Jvm的结构比原生机器语言更加简单，简化了指令集. 更好的事情就是Oracle文档中包含这些指令。</br>
The nice thing about bytecode is that you learn its syntax once, then it applies on all Java supported platforms — because it is an intermediate representation of the code, and not the actual executable code for the underlying CPU. 
Moreover, bytecode is simpler than native machine code because the JVM architecture is rather simple, hence simplifying the instruction set. Yet another nice thing is that all instructions in this set are fully documented by Oracle.</br>

在学习字节码指令之前，让我们熟悉一些有关JVM的先决条件。</br>
Before learning about the bytecode instruction set though, let's get familiar with a few things about the JVM that are needed as a prerequisite.</br>

### JVM Data Types  虚拟机数据类型

Java是静态类型语言，这会影响字节码指令集的设计，从而使一条指令的操作特定的类型，例如，有一个加法指令将两个数字相加： iadd, ladd, fadd, dadd. 
它们的操作数的类型类型分别为int,long,float,double.大部分字节码根据操作数类型具有不同形式但相同功能的特征。</br>

Java is statically typed, which affects the design of the bytecode instructions such that an instruction expects itself to operate on values of specific types.
For example, there are several add instructions to add two numbers: iadd, ladd, fadd, dadd. They expect operands of type, respectively, int, long, float, and double. 
The majority of bytecode has this characteristic of having different forms of the same functionality depending on the operand types.</br>

数据类型在JVM定义为:
The data types defined by the JVM are:


1.主要类型
* 数字类型：byte(8位 2的补码),short(16位 2的补码),
          int (32位 2的补码),long(64位 2的补码）
          char(16位 无符号编码）,
          float (32位 IEEE 754单精度FP),double (64位 IEEE 754双精度FP)
* boolean 类型
* returnAddress 指示指针

1.Primitive types:
* Numeric types: byte (8-bit 2's complement), short (16-bit 2's complement), 
  int (32-bit 2's complement), long (64-bit 2's complement), 
  char (16-bit unsigned Unicode), float (32-bit IEEE 754 single precision FP), 
  double (64-bit IEEE 754 double precision FP)
* boolean type
* returnAddress: pointer to instruction


2.引用类型
* Class类型
* 数组类型
* 接口类型

2. Reference types:
* Class types
* Array types
* Interface types

boolean类型被字节码限制，例如，这里没有指令集直接操作boolean类型，Boolean值将有编译器转换成int，并使用相应的int指令。</br>

The boolean type has limited support in bytecode. For example, there are no instructions that directly operate on boolean values. 
Boolean values are instead converted to int by the compiler and the corresponding int instruction is used.</br>

Java 开发者应该很熟悉上面的类型，除了返回地址，没有等效的编程语言类型。</br>
Java developers should be familiar with all of the above types, except returnAddress, which has no equivalent programming language type.</br>

###  Stack-Based Architecture 基于堆栈的架构

字节码指令集的简单性很大归功与Sun设计了基于基于堆栈的VM体系结构，而不是基于寄存器的。这里有一系列的内存组建应用与Jvm进程，从本质上将只有Jvm堆栈需要被检查遵循字节码指令：</br>
The simplicity of the bytecode instruction set is largely due to Sun having designed a stack-based VM architecture, as opposed to a register-based one. There are various memory components used by a JVM process, 
but only the JVM stacks need to be examined in detail to essentially be able to follow bytecode instructions:</br>

PC寄存器：对于Java程序的每一个运行的线程，PC寄存器都存储当前指令的地址。</br>
PC register: for each thread running in a Java program, a PC register stores the address of the current instruction.</br>

虚拟机栈：对于每一个线程，栈都分配本地向量表，方法参数和返回值。这是显示3个线程的堆栈插图。</br>
JVM stack: for each thread, a stack is allocated where local variables, method arguments, and return values are stored. Here is an illustration showing stacks for 3 threads.</br>

![avatar](/img/20200418/jvm_stacks.png)

堆内存：内存共享所有线程和被加载的对象（实例对象和数组）,对象的回收被垃圾回收器管理。</br>
Heap: memory shared by all threads and storing objects (class instances and arrays). Object deallocation is managed by a garbage collector.</br>
![avatar](/img/20200418/heap.png)


















