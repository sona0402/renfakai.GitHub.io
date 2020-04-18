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

Java字节码简介  
Introduction to Java Bytecode  

继续深入研究JVM内部和Java字节码，以了解如何分解文件以进行深入检查。  
Follow along this deep dive into JVM internals and Java bytecode to see how you 
can disassemble your files for in-depth inspections.  

读编译后的字节码是很乏味的，但是对Java开发者很有用。为什么还要第一时间学习这些低级内容呢？这是我上周发生的情况，我在我的电脑上对一些代码进行了变更，
编译成Jar并部署到服务器去测试一个潜在的性能问题。不幸得是这些代码没有提交到版本控制系统，
出于一些其他原因，我电脑对代码的改变被删除了，并且找不到轨迹，过了几个月后，我需要在源代码上进行变更(这些代码花了很大力气编写),但是我找不到它们了。  

Reading compiled Java bytecode can be tedious, even for experienced Java developers. 
Why do we need to know about such low-level stuff in the first place? 
Here is a simple scenario that happened to me last week: I had made some code changes on my machine a long time ago, 
compiled a JAR, and deployed it on a server to test a potential fix for a performance issue. Unfortunately, 
the code was never checked into a version control system, and for whatever reason, the local changes were deleted without a trace.
After a couple of months, I needed those changes in source form again (which took quite an effort to come up with), 
but I could not find them!  

幸运的是这些被编译的代码还存在远程的服务器上面，所以我松了一口了气，我从服务器拉下了Jar文件并用反编译工具打开它。
有一个问题：反编译GUI不是一个完美的工具，而且有很多Classes没有在jar文件中，
出于某些原因，当我打开特定的class去看反编译代码时引起了UI的一个bug,然后我把反编译工具丢进了垃圾箱！  
Luckily the compiled code still existed on that remote server. So with a sigh of relief, I fetched the JAR again and opened it using a decompiler editor... 
Only one problem: The decompiler GUI is not a flawless tool, and out of the many classes in that JAR, 
for some reason, only the specific class I was looking to decompile caused a bug in the UI whenever I opened it, and the decompiler to crash!  

绝望的时候采取绝望的措施，幸好，我熟悉原生字节码，而且我宁愿话费一些时间手动反编译这些代码片段而不是研究更改并再次测试它们。
由于我还记得这些代码位置，读取字节码有助于我找到明确的更改并再次构造到源代码（我确保从错误中吸取教训，并保护它们）。  
Desperate times call for desperate measures. Fortunately, I was familiar with raw bytecode, and I'd rather take some time manually decompiling some pieces of the code rather than work through the changes and testing them again. 
Since I still remembered at least where to look in the code, reading bytecode helped me pinpoint the exact changes and construct them back in source form. (I made sure to learn from my mistake and preserve them this time!)  

字节码的好处是你学习它的语法一次，适用于Java直接的平台，因为它是代码的中间表示，而不是cpu下执行的机器语言，
此外，字节码由于Jvm的结构比原生机器语言更加简单，简化了指令集. 更好的事情就是Oracle文档中包含这些指令。  
The nice thing about bytecode is that you learn its syntax once, then it applies on all Java supported platforms — because it is an intermediate representation of the code, and not the actual executable code for the underlying CPU. 
Moreover, bytecode is simpler than native machine code because the JVM architecture is rather simple, hence simplifying the instruction set. Yet another nice thing is that all instructions in this set are fully documented by Oracle.  

在学习字节码指令之前，让我们熟悉一些有关JVM的先决条件。  
Before learning about the bytecode instruction set though, let's get familiar with a few things about the JVM that are needed as a prerequisite.  

### JVM Data Types  虚拟机数据类型

Java是静态类型语言，这会影响字节码指令集的设计，从而使一条指令的操作特定的类型，例如，有一个加法指令将两个数字相加： iadd, ladd, fadd, dadd. 
它们的操作数的类型类型分别为int,long,float,double.大部分字节码根据操作数类型具有不同形式但相同功能的特征。  

Java is statically typed, which affects the design of the bytecode instructions such that an instruction expects itself to operate on values of specific types.
For example, there are several add instructions to add two numbers: iadd, ladd, fadd, dadd. They expect operands of type, respectively, int, long, float, and double. 
The majority of bytecode has this characteristic of having different forms of the same functionality depending on the operand types.  

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

2.Reference types:
* Class types
* Array types
* Interface types

boolean类型被字节码限制，例如，这里没有指令集直接操作boolean类型，Boolean值将有编译器转换成int，并使用相应的int指令。  

The boolean type has limited support in bytecode. For example, there are no instructions that directly operate on boolean values. 
Boolean values are instead converted to int by the compiler and the corresponding int instruction is used.  

Java 开发者应该很熟悉上面的类型，除了返回地址，没有等效的编程语言类型。  
Java developers should be familiar with all of the above types, except returnAddress, which has no equivalent programming language type.  

###  Stack-Based Architecture 基于堆栈的架构

字节码指令集的简单性很大归功与Sun设计了基于基于堆栈的VM体系结构，而不是基于寄存器的。这里有一系列的内存组建应用与Jvm进程，从本质上将只有Jvm堆栈需要被检查遵循字节码指令：  
The simplicity of the bytecode instruction set is largely due to Sun having designed a stack-based VM architecture, as opposed to a register-based one. There are various memory components used by a JVM process, 
but only the JVM stacks need to be examined in detail to essentially be able to follow bytecode instructions:  

PC寄存器：对于Java程序的每一个运行的线程，PC寄存器都存储当前指令的地址。  
PC register: for each thread running in a Java program, a PC register stores the address of the current instruction.  

虚拟机栈：对于每一个线程，栈都分配本地向量表，方法参数和返回值。这是显示3个线程的堆栈插图。  
JVM stack: for each thread, a stack is allocated where local variables, method arguments, and return values are stored. Here is an illustration showing stacks for 3 threads.  

![avatar](/img/20200418/jvm_stacks.png)

堆内存：内存共享所有线程和被加载的对象（实例对象和数组）,对象的回收被垃圾回收器管理。  
Heap: memory shared by all threads and storing objects (class instances and arrays). Object deallocation is managed by a garbage collector.  
![avatar](/img/20200418/heap.png)

方法区:每一个被加载的class,它存储方法代码和符号表。(例如引用类型和方法) 和长量池。
Method area: for each loaded class, it stores the code of methods and a table of symbols (e.g. references to fields or methods) and constants known as the constant pool.
![avatar](/img/20200418/method_area.png)

JVM栈由栈帧组成，当方法调用时候推入栈帧，并在方法完成时（通过正常返回或引发异常）从堆栈中弹出。每个栈帧还包括:  
A JVM stack is composed of frames, each pushed onto the stack when a method is invoked and popped from the stack when the method completes (either by returning normally or by throwing an exception).   
Each frame further consists of:

1.局部变量表,索引从0~length-1,长度有编辑器计算，局部向量表可以保存任意类型,但是long和double除外，因为它占两个位置。  
1.An array of local variables, indexed from 0 to its length minus 1. The length is computed by the compiler. A local variable can hold a value of any type, except long and double values, which occupy two local variables.  

2.一个操作栈用于储存用于存储用作指令的操作数,将参数推送到方法调用。  
2.An operand stack used to store intermediate values that would act as operands for instructions, or to push arguments to method invocations.  
![avatar](/img/20200418/stack_frame_zoom.png)


### Bytecode Explored 节码探索
关于JVM内部想法,我们可以看一些字节码基础案例,这些案例生成于简单的代码。在Java class 文件中每一个访达都会有一些代码段，它包含一系列的指令集,主要包含以下格式:  

With an idea about the internals of a JVM, we can look at some basic bytecode example generated from sample code. 
Each method in a Java class file has a code segment that consists of a sequence of instructions, each having the following format:  

opcode (1 byte)      operand1 (optional)      operand2 (optional)      ...  

该指令由一个字节的操作码和零个或多个包含要操作的数据去操作。  
That is an instruction that consists of one-byte opcode and zero or more operands that contain the data to operate.  

当前执行方法的栈帧,一个指令可以从操作栈中推入或者推出值，而且它可以潜在的加载或存储值到局部变量表,让我门来看一个简单的案例:  
Within the stack frame of the currently executing method, an instruction can push or pop values onto the operand stack,
and it can potentially load or store values in the array local variables. Let's look at a simple example:  
  
`public static void main(String[] args) {
     int a = 1;
     int b = 2;
     int c = a + b;
 }`
  
为了能够打印被编辑的class(假定这个文件是Test.class)字节码结果,我们可以使用javap工具:  
In order to print the resulting bytecode in the compiled class (assuming it is in a file Test.class), we can run the javap tool:    
  
`javap -v Test.class`
  
And we get:  
  
`1  public static void main(java.lang.String[]);
 2  descriptor: ([Ljava/lang/String;)V
 3  flags: (0x0009) ACC_PUBLIC, ACC_STATIC
 4  Code:
 5  stack=2, locals=4, args_size=1
 6  0: iconst_1
 7  1: istore_1
 8  2: iconst_2
 9  3: istore_2
 10 4: iload_1
 11 5: iload_2
 12 6: iadd
 13 7: istore_3
 14 8: return
 ...`
  
我们能看到主方法的签名,形容和表明方法需要一个字符串数组([Ljava/lang/String; ),并且有一个返回类型(V )。  
后续有一系列的方法形容,例如公有的(ACC_PUBLIC)和静态的(ACC_STATIC)。  
We can see the method signature for the main method, a descriptor that indicates that the method takes an array of Strings ([Ljava/lang/String; ), 
and has a void return type (V ). A set of flags follow that describe the method as public (ACC_PUBLIC) and static (ACC_STATIC).  

更重要的部分是代码,它包含了方法的说明和信息,例如操作数栈的最大深度(这种情况下为2,stack=2),和一定数量的局部变量表被分配在栈帧中(locals=4),
以上指令引用了上面的除0索引外的所有位置，它包含了参数的引用，其他3个局部变量对应源码中的参数a,b,c。    
The most important part is the Code attribute, which contains the instructions for the method along with information such as the maximum depth of the operand stack (2 in this case), 
and the number of local variables allocated in the frame for this method (4 in this case). 
All local variables are referenced in the above instructions except the first one (at index 0),
which holds the reference to the args argument. The other 3 local variables correspond to variables a, b and c in the source code.  

从0-8的指令集操作如下:  
The instructions from address 0 to 8 will do the following:  

iconst_1: 推送常数1到操作数栈。  
iconst_1: Push the integer constant 1 onto the operand stack.  
![avatar](/img/20200418/iconst_12.png)  
istore_1: 从操作数栈推出(一个int值)并储存到局部变量表索引1处,它对应的是变量a。    
istore_1: Pop the top operand (an int value) and store it in local variable at index 1, which corresponds to variable a.     
![avatar](/img/20200418/istore_11.png)  
iconst_2: 推送数字常量2到操作数栈。    
iconst_2: Push the integer constant 2 onto the operand stack.   
![avatar](/img/20200418/iconst_2.png)  
istore_2: 从操作数栈推出int并储存到局部变量索引2,它对应向量b。  
istore_2: Pop the top operand int value and store it in local variable at index 2, which corresponds to variable b.  
![avatar](/img/20200418/istore_2.png)   
iload_1: 将第二个int型本地变量推送至栈顶   
iload_1: Load the int value from local variable at index 1 and push it onto the operand stack.  
![avatar](/img/20200418/iload_1.png)  

**iload_2 将第三个int型本地变量推送至栈顶**  
**这里请注意原文与原图不符**  
iload_2: Load the int value from the local variable at index 1 and push it onto the operand stack.  
![avatar](/img/20200418/iload_2.png)  
iadd: 将栈顶两int型数值相加并将结果压入栈顶。  
iadd: Pop the top two int values from the operand stack, add them, and push the result back onto the operand stack.  
![avatar](/img/20200418/iload_2.png)   

istore_3: 从操作数栈推出int并储存到局部变量索引3,它对应向量c。 
istore_3: Pop the top operand int value and store it in local variable at index 3, which corresponds to variable c.  
![avatar](/img/20200418/istore_3.png)  

return: 从void方法返回。  
return: Return from the void method. 

上面的指令集都是都仅有一个操作码，操作码精确的指示了JVM要执行的操作。  
Each of the above instructions consists of only an opcode, which dictates exactly the operation to be executed by the JVM.  

### Method Invocations 方法调用
在上面试例中,这里只有一个主方法，让我们假定我们需要更京西的计算来计算变量c，并且我们决定将其放到calc方法中:  
In the above example, there is only one method, the main method.
Let's assume that we need to a more elaborate computation for the value of variable c,
and we decide to place that in a new method called calc:

`
public static void main(String[] args) {
    int a = 1;
    int b = 2;
    int c = calc(a, b);
}
static int calc(int a, int b) {
    return (int) Math.sqrt(Math.pow(a, 2) + Math.pow(b, 2));
}`

让我们看字节码结果:  
Let's see the resulting bytecode:  

`public static void main(java.lang.String[]);
   descriptor: ([Ljava/lang/String;)V
   flags: (0x0009) ACC_PUBLIC, ACC_STATIC
   Code:
     stack=2, locals=4, args_size=1
        0: iconst_1
        1: istore_1
        2: iconst_2
        3: istore_2
        4: iload_1
        5: iload_2
        6: invokestatic  #2         // Method calc:(II)I
        9: istore_3
       10: return
 static int calc(int, int);
   descriptor: (II)I
   flags: (0x0008) ACC_STATIC
   Code:
     stack=6, locals=2, args_size=2
        0: iload_0
        1: i2d
        2: ldc2_w        #3         // double 2.0d
        5: invokestatic  #5         // Method java/lang/Math.pow:(DD)D
        8: iload_1
        9: i2d
       10: ldc2_w        #3         // double 2.0d
       13: invokestatic  #5         // Method java/lang/Math.pow:(DD)D
       16: dadd
       17: invokestatic  #6         // Method java/lang/Math.sqrt:(D)D
       20: d2i
       21: ireturn`
       

与主方法唯一的差别是它没有iadd指令,我们可以看到一个invokestatic指令，它只是调用了静态方法calc。
关键要注意操作数栈包含了calc方法的两个参数，换句话说，调用方法将被调用方法的所有参数以正确的顺序压入操作数堆栈，从而准备这些参数。
invokestatic(类似的调用指令,后续在讲)随后讲弹出这些参数,一些新的栈帧将会被调用方法创建,并且参数会被放到局部变量表中。  
The only difference in the main method code is that instead of having the iadd instruction, 
we now an invokestatic instruction, which simply invokes the static method calc. 
The key thing to note is that the operand stack contained the two arguments that are passed to the method calc. 
In other words, the calling method prepares all arguments of the to-be-called method by pushing them onto the operand stack in the correct order. 
invokestatic (or a similar invoke instruction, as will be seen later) will subsequently pop these arguments, 
and a new frame is created for the invoked method where the arguments are placed in its local variable array.


我们注意到invokestatic指令使用了3个字节进行寻址,
它从6到9,这是因为，它与我们之前看到的指令不同,
invokestatic包含两个额外的字节构造方法引用(额外的操作码)。
这个引用被展示为#2，它是calc方法的符号引用，可以从前面讲到的常量池找到。  
We also notice that the invokestatic instruction occupies 3 bytes by looking at the address, 
which jumped from 6 to 9. This is because, unlike all instructions seen so far, 
invokestatic includes two additional bytes to construct the reference to the method to be invoked (in addition to the opcode). 
The reference is shown by javap as #2, which is a symbolic reference to the calc method,
which is resolved from the constant pool described earlier.

显然，其他新信息是calc方法本身的代码。它先将第一个integer参数加载到操作数栈(iload_0)。
下一个指令,i2d,扩大它的范围将它转换成一个double，并将结果推到操作数栈顶。  
The other new information is obviously the code for the calc method itself. 
It first loads the first integer argument onto the operand stack (iload_0). 
The next instruction, i2d, converts it to a double by applying widening conversion. 
The resulting double replaces the top of the operand stack.

下一个指令推送一个double常量2.0d(从常量池获取)到操作数栈,然后使用操作数栈准备的两个操作数值调用Math.pow静态方法(calc的第一个参数和常量2.0d),
当Math.pow方法返回，它的结果会被储存到调用程序的操作数栈中，可以在下面说。  
The next instruction pushes a double constant 2.0d  (taken from the constant pool) onto the operand stack. 
Then the static Math.pow method is invoked with the two operand values prepared so far (the first argument to calc and the constant 2.0d). 
When the Math.pow method returns, its result will be stored on the operand stack of its invoker. 
This can be illustrated below.

![avatar](/img/20200418/math_pow2.png)  

相同的过程适用于计算Math.pow（b，2）:  
The same procedure is applied to compute Math.pow(b, 2):  

![avatar](/img/20200418/math_pow21.png)  

下一个指令,dadd,推出两个中间结果，进行相加，并将结果退回到栈顶，最终，静态调用调用数学平方，然后使用缩小转换（d2i）将结果从double转换为int。
生成的int返回到main方法，该方法将其存储回c（istore_3）。  
The next instruction, dadd, pops the top two intermediate results, adds them, and pushes the sum back to the top. Finally,
invokestatic invokes Math.sqrt on the resulting sum, and the result is cast from double to int using narrowing conversion (d2i).
The resulting int is returned to the main method, which stores it back to c (istore_3).  

### Instance Creations 创建实例
让我们修改示例，并引入一个Point类来封装XY坐标。  
Let's modify the example and introduce a class Point to encapsulate XY coordinates.  

`public class Test {
     public static void main(String[] args) {
         Point a = new Point(1, 1);
         Point b = new Point(5, 3);
         int c = a.area(b);
     }
 }
 class Point {
     int x, y;
     Point(int x, int y) {
         this.x = x;
         this.y = y;
     }
     public int area(Point b) {
         int length = Math.abs(b.y - this.y);
         int width = Math.abs(b.x - this.x);
         return length * width;
     }
 }`  
 

主要方法的字节码如下:  
The compiled bytecode for the main method is shown below:  

`public static void main(java.lang.String[]);
   descriptor: ([Ljava/lang/String;)V
   flags: (0x0009) ACC_PUBLIC, ACC_STATIC
   Code:
     stack=4, locals=4, args_size=1
        0: new           #2       // class test/Point
        3: dup
        4: iconst_1
        5: iconst_1
        6: invokespecial #3       // Method test/Point."<init>":(II)V
        9: astore_1
       10: new           #2       // class test/Point
       13: dup
       14: iconst_5
       15: iconst_3
       16: invokespecial #3       // Method test/Point."<init>":(II)V
       19: astore_2
       20: aload_1
       21: aload_2
       22: invokevirtual #4       // Method test/Point.area:(Ltest/Point;)I
       25: istore_3
       26: return`
       

这里遇到的新的指令new,dup和invokespecial。类似与编程语言的新的运算符,new 指令创建传递给它的操作数中指定类型的对象(Point class的符号引用)
对象分配在堆内存上,并且将对象引用推动到操作数栈上。   
The new instructions encountereted here are new , dup, and invokespecial. 
Similar to the new operator in the programming language, 
the new instruction creates an object of the type specified in the operand passed to it (which is a symbolic reference to the class Point). 
Memory for the object is allocated on the heap, and a reference to the object is pushed on the operand stack.

dup指令复制操作数栈顶的值,这意味着现在我们堆栈顶部有两个Point对象。
接下来的三条指令将构造函数的参数（用于初始化对象）压入操作数堆栈，然后调用与构造函数相对应的特殊初始化方法，
下一个方法是将字段x和y初始化。方法完成后，将使用前三个操作数堆栈值，剩下的就是对创建对象的原始引用（到目前为止，该对象已成功初始化）。  
The dup instruction duplicates the top operand stack value, 
which means that now we have two references the Point object on the top of the stack. 
The next three instructions push the arguments of the constructor (used to initialize the object) onto the operand stack, 
and then invoke a special initialization method, which corresponds with the constructor. 
The next method is where the fields x and y will get initialized. After the method is finished, 
the top three operand stack values are consumed, and what remains is the original reference to the created object (which is, by now, successfully initialized).  
 
 
![avatar](/img/20200418/init.png)  

接下来，astore_1弹出该Point引用并将其分配给索引1处的局部变量（astore_1中的a表示这是一个引用值）。  
Next, astore_1 pops that Point reference and assigns it to the local variable at index 1 (the a in astore_1 indicates this is a reference value).  

![avatar](/img/20200418/init_store.png)  

重复相同的创建和初始化Point，并将它赋于到b。   
The same procedure is repeated for creating and initializing the second Point instance, which is assigned to variable b.  

![avatar](/img/20200418/init2.png)  
![avatar](/img/20200418/init_store2.png)  

最后一步加载两个Point对象引用到局部变量表索引1和2处(分别使用aload_1和aload_2)，
并且使用invokevirtual调用面积计算方法,它根据对象的实际类型处理将调用分配给适当的方法。
例如，如果变量a包含扩展Point的SpecialPoint类型的实例，并且子类型覆盖area方法，则将调用重写方法。
在这种情况下，如果没有子类，因此仅一个区域方法可用。  
The last step loads the references to the two Point objects from local variables at indexes 1 and 2 (using aload_1 and aload_2 respectively), 
and invokes the area method using invokevirtual, 
which handles dispatching the call to the appropriate method based on the actual type of the object. For example, 
if the variable a contained an instance of type SpecialPoint that extends Point, and the subtype overrides the area method, 
then the overriden method is invoked. In this case, there is no subclass, and hence only one area method is available.

### The Other Way Around 另一种方式
您无需掌握每条指令的理解和确切的执行流程，就可以根据手头的字节码了解程序的功能。
例如，以我为例，我想检查代码是否使用Java流读取文件，以及该流是否已正确关闭。现在给出以下字节码，
相对容易地确定确实使用了流，并且很可能作为try-with-resources语句的一部分将其关闭。  
You don't need to master the understanding of each instruction and the exact flow of execution to gain an idea about 
what the program does based on the bytecode at hand. 
For example, in my case, I wanted to check if the code employed a Java stream to read a file, 
and whether the stream was properly closed. Now given the following bytecode, 
it is relatively easy to determine that indeed a stream is used and most likely it is being closed as part of a try-with-resources statement.  
`public static void main(java.lang.String[]) throws java.lang.Exception;
  descriptor: ([Ljava/lang/String;)V
  flags: (0x0009) ACC_PUBLIC, ACC_STATIC
  Code:
    stack=2, locals=8, args_size=1
       0: ldc           #2                  // class test/Test
       2: ldc           #3                  // String input.txt
       4: invokevirtual #4                  // Method java/lang/Class.getResource:(Ljava/lang/String;)Ljava/net/URL;
       7: invokevirtual #5                  // Method java/net/URL.toURI:()Ljava/net/URI;
      10: invokestatic  #6                  // Method java/nio/file/Paths.get:(Ljava/net/URI;)Ljava/nio/file/Path;
      13: astore_1
      14: new           #7                  // class java/lang/StringBuilder
      17: dup
      18: invokespecial #8                  // Method java/lang/StringBuilder."<init>":()V
      21: astore_2
      22: aload_1
      23: invokestatic  #9                  // Method java/nio/file/Files.lines:(Ljava/nio/file/Path;)Ljava/util/stream/Stream;
      26: astore_3
      27: aconst_null
      28: astore        4
      30: aload_3
      31: aload_2
      32: invokedynamic #10,  0             // InvokeDynamic #0:accept:(Ljava/lang/StringBuilder;)Ljava/util/function/Consumer;
      37: invokeinterface #11,  2           // InterfaceMethod java/util/stream/Stream.forEach:(Ljava/util/function/Consumer;)V
      42: aload_3
      43: ifnull        131
      46: aload         4
      48: ifnull        72
      51: aload_3
      52: invokeinterface #12,  1           // InterfaceMethod java/util/stream/Stream.close:()V
      57: goto          131
      60: astore        5
      62: aload         4
      64: aload         5
      66: invokevirtual #14                 // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
      69: goto          131
      72: aload_3
      73: invokeinterface #12,  1           // InterfaceMethod java/util/stream/Stream.close:()V
      78: goto          131
      81: astore        5
      83: aload         5
      85: astore        4
      87: aload         5
      89: athrow
      90: astore        6
      92: aload_3
      93: ifnull        128
      96: aload         4
      98: ifnull        122
     101: aload_3
     102: invokeinterface #12,  1           // InterfaceMethod java/util/stream/Stream.close:()V
     107: goto          128
     110: astore        7
     112: aload         4
     114: aload         7
     116: invokevirtual #14                 // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
     119: goto          128
     122: aload_3
     123: invokeinterface #12,  1           // InterfaceMethod java/util/stream/Stream.close:()V
     128: aload         6
     130: athrow
     131: getstatic     #15                 // Field java/lang/System.out:Ljava/io/PrintStream;
     134: aload_2
     135: invokevirtual #16                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
     138: invokevirtual #17                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     141: return
    ...`
    
 我们可以看到java/util/stream/Stream被迭代调用，在调用InvokeDynamic之前，先引用一个Consumer。
 然后，我们看到了一个字节代码块，它调用Stream.close以及调用Throwable.addSuppressed的分支。
 这是编译器为try-with-resources语句生成的基本代码。  
 We see occurrences of java/util/stream/Stream where forEach is called,
 preceded by a call to InvokeDynamic with a reference to a Consumer. 
 And then we see a chunk of bytecode that calls Stream.close along with branches that call Throwable.addSuppressed.
 This is the basic code that gets generated by the compiler for a try-with-resources statement.  
 
 这是完整性的原始来源：  
 Here's the original source for completeness:  

`
public static void main(String[] args) throws Exception {
    Path path = Paths.get(Test.class.getResource("input.txt").toURI());
    StringBuilder data = new StringBuilder();
    try(Stream lines = Files.lines(path)) {
        lines.forEach(line -> data.append(line).append("\n"));
    }
    System.out.println(data.toString());
}`


### Conclusion  结论
由于字节码指令集的简单性以及生成指令时几乎没有编译器优化的原因，如果需要，反汇编类文件可能是检查源代码中应用程序代码更改的一种方法。  
Thanks to the simplicity of the bytecode instruction set and the near absence of compiler optimizations when generating its instructions,
disassembling class files could be one way to examine changes into your application code without having the source, 
if that ever becomes a need.

















