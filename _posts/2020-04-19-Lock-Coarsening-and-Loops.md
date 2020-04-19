---
author: @shipilev
layout: post
title: 翻译JVM Anatomy Quark #1: Lock Coarsening and Loops
date: 2020-04-18
categories: jvm
tags: [java，jvm]
description:  翻译一些自己喜欢的文章，由于能力有限，敬请见谅，此章节主要为左耳听风79节练级。
---


## [JVM Anatomy Quark #1: Lock Coarsening and Loops 原文地址](https://shipilev.net/jvm/anatomy-quarks/1-lock-coarsening-for-loops/)


### 问题
众所周知，Hotspot对锁的粗话优化，可以有效地合并几个相邻的锁定块，从而减少了锁定开销。它有效地转换为：  
  
```java
synchronized (obj) {
  // statements 1
}
synchronized (obj) {
  // statements 2
}
```
  
…​into:  
  
```java
synchronized (obj) {
  // statements 1
  // statements 2
}
```
  
现在，今天提出一个有趣的问题，Hotspot是否对循环也做这种优化？例如:  
  
```java
for (...) {
  synchronized (obj) {
    // something
  }
}
```
  
能够优化成这个样子吗？  
  
```java
synchronized (this) {
  for (...) {
     // something
  }
}
```
  
从理论上讲，没有什么可以阻止我们这样做。甚至可能会看到优化，就像仅在锁上打开循环不切换那样在类上。
但是，不利的一面是可能使锁变得过于粗糙，特定线程将在执行大循环时占用锁。  

### 实验

解决此问题的最简单方法是找到当前Hotspot对这类优化的测试。幸运的是，JMH非常简单。
它不仅对建立基准非常有用，甚至对工程也很重要。让我们从一个简单基准开始：  
  
```java
@Fork(..., jvmArgsPrepend = {"-XX:-UseBiasedLocking"})
@State(Scope.Benchmark)
public class LockRoach {
    int x;

    @Benchmark
    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public void test() {
        for (int c = 0; c < 1000; c++) {
            synchronized (this) {
                x += 0x42;
            }
        }
    }
}
```
  
(所有代码都在这里)  
  
这里有一些中能够逃的技巧：    
1.使用-XX禁用偏向锁定:-UseBiasedLocking避免长时间预热，因为偏置锁定不会立即启动，在初始化的5秒后开始启动（请参考BiasedLockingStartupDelay选项）  
2.禁用@Benchmark方法内联有助于在反汇编中将其分开。  
3.加一个魔数，0x42有助于快速找到反汇编中的加法。  
  
Running at i7 4790K, Linux x86_64, JDK EA 9b156:  
  
```java 
    Benchmark            Mode  Cnt      Score    Error  Units
    LockRoach.test       avgt    5   5331.617 ± 19.051  ns/op
```
  
你能从这些数字中看到什么？你什么都说不出对吧，我们需要研究下面实际发生的情况，`-prof perfasm`是非常有效的。
它显示了生成代码中最热的部分，如果使用默认设置运行能告诉`lock cmpxchg`执行锁定时的实际热指令集并且打印它们。
使用`-prof perfasm：mergeMargin = 1000`来将这些热门区域合并为一张完整的图片，就会得到这一显而易见输出。    
  
将其进一步分解-跳转的级联是锁定/解锁-并注意累积最多循环的代码（第一列），我们可以看到最热循环如下所示：    
  
```asm

​↗  0x00007f455cc708c1: lea    0x20(%rsp),%rbx
​│          < blah-blah-blah, monitor enter >     ; <--- coarsened!
​│  0x00007f455cc70918: mov    (%rsp),%r10        ; load $this
​│  0x00007f455cc7091c: mov    0xc(%r10),%r11d    ; load $this.x
​│  0x00007f455cc70920: mov    %r11d,%r10d        ; ...hm...
​│  0x00007f455cc70923: add    $0x42,%r10d        ; ...hmmm...
​│  0x00007f455cc70927: mov    (%rsp),%r8         ; ...hmmmmm!...
​│  0x00007f455cc7092b: mov    %r10d,0xc(%r8)     ; LOL Hotspot, redundant store, killed two lines below
​│  0x00007f455cc7092f: add    $0x108,%r11d       ; add 0x108 = 0x42 * 4 <-- unrolled by 4
​│  0x00007f455cc70936: mov    %r11d,0xc(%r8)     ; store $this.x back
​│          < blah-blah-blah, monitor exit >      ; <--- coarsened!
​│  0x00007f455cc709c6: add    $0x4,%ebp          ; c += 4   <--- unrolled by 4
​│  0x00007f455cc709c9: cmp    $0x3e5,%ebp        ; c < 1000?
​╰  0x00007f455cc709cf: jl     0x00007f455cc708c1

```
  
嗯，循环似乎展开了4次，然后在这4次迭代中加了粗锁！为了排除循环展开对锁粗化的影响，
我们可以量化这种有限的粗化对性能的好处，可以使用-XX：LoopUnrollLimit = 1来减少展开：    
  
```java 

Benchmark            Mode  Cnt      Score    Error  Units

# Default
LockRoach.test       avgt    5   5331.617 ± 19.051  ns/op

# -XX:LoopUnrollLimit=1
LockRoach.test       avgt    5  20679.043 ±  3.133  ns/op

```
  
哇，性能提高了4倍！这是有道理的，因为我们已经观察到最热的东西是锁定cmpxchg和锁定。
自然，4倍的粗化锁意味着4倍的吞吐量。很酷，我们可以宣称成功并继续前进吗？
尚未，我们必须验证禁用循环展开是否确实可以提供我们要比较的内容。
perfasm似乎表明它执行了类似的热循环，但是步幅很短。  
  
```java

​↗  0x00007f964d0893d2: lea    0x20(%rsp),%rbx
​│          < blah-blah-blah, monitor enter >
​│  0x00007f964d089429: mov    (%rsp),%r10        ; load $this
​│  0x00007f964d08942d: addl   $0x42,0xc(%r10)    ; $this.x += 0x42
​│          < blah-blah-blah, monitor exit >
​│  0x00007f964d0894be: inc    %ebp               ; c++
​│  0x00007f964d0894c0: cmp    $0x3e8,%ebp        ; c < 1000?
​╰  0x00007f964d0894c6: jl     0x00007f964d0893d2 ;

```
  
所有问题已经查证。  
  
### 观察结果
尽管锁粗化并不能在整个循环中起作用，一旦中间表示看起来有N个序列进行加锁/解锁，另一个循环优化为循环展开设置常规锁定粗化的阶段，
这样可以提高性能，并有助于限制锁粗化范围，避免范围过大导致的性能问题。  
 