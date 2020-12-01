---
author: renfakai
layout: post
title: 为什么Doug Lea喜欢将锁放到局部变量表
date: 2020-12-01
categories: Java
tags: [java]
description: 对Doug Lea代码风格猜想
---

Doug Lea 我心目中的神，凭借一己之力编写了并发包，牛皮，牛皮，牛皮。

##  问题
 
 为什么Doug Lea 喜欢将不可变锁引用放到局部变量里面
 
### 源代码
本文章使用java.util.concurrent.DelayQueue为案例

```java

 public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
     implements BlockingQueue<E> {
 
     private final transient ReentrantLock lock = new ReentrantLock();
     private final PriorityQueue<E> q = new PriorityQueue<E>();

     // 其他代码暂时忽略

     // 仅仅这一段代码进行展示
     public int size() {
    
        // 成员变量移到局部变量表
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return q.size();
        } finally {
            lock.unlock();
        }
    }
}


````
### 字节码级别优化

- 使用自己编写案例，查看字节码
```java
public class Queue {

    private final transient ReentrantLock rt = new ReentrantLock();

    public void t() {
        rt.lock();
        try {
            System.out.println(1);
        } finally {
            rt.unlock();
        }
    }
}
```
使用编译器进行编译，查看未优化之前字节码

```java

      // 编译class
      // javac Queue.java 

      // 查看字节码
      // javap -p -v Queue

       public void t();
         descriptor: ()V
         flags: ACC_PUBLIC
         Code:
           stack=2, locals=2, args_size=1
              0: aload_0
              1: getfield      #4                  // Field rt:Ljava/util/concurrent/locks/ReentrantLock;
              4: invokevirtual #5                  // Method java/util/concurrent/locks/ReentrantLock.lock:()V
              7: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
             10: iconst_1
             11: invokevirtual #7                  // Method java/io/PrintStream.println:(I)V
             14: aload_0
             15: getfield      #4                  // Field rt:Ljava/util/concurrent/locks/ReentrantLock;
             18: invokevirtual #8                  // Method java/util/concurrent/locks/ReentrantLock.unlock:()V
             21: goto          34
             24: astore_1
             25: aload_0
             26: getfield      #4                  // Field rt:Ljava/util/concurrent/locks/ReentrantLock;
             29: invokevirtual #8                  // Method java/util/concurrent/locks/ReentrantLock.unlock:()V
             32: aload_1
             33: athrow
             34: return
           Exception table:
              from    to  target type
                  7    14    24   any
           LineNumberTable:
             line 15: 0
             line 17: 7
             line 19: 14
             line 20: 21
             line 19: 24
             line 20: 32
             line 21: 34
           StackMapTable: number_of_entries = 2
             frame_type = 88 /* same_locals_1_stack_item */
               stack = [ class java/lang/Throwable ]
             frame_type = 9 /* same */

```
从上面可以看到每次获取锁的时候，都先从局部变量表里面获取index=0的this进栈后，在获取锁，内容如下
```java 
              0: aload_0
              1: getfield      #4                  // Field rt:Ljava/util/concurrent/locks/ReentrantLock;
              4: invokevirtual #5                  // Method java/util/concurrent/locks/ReentrantLock.lock:()V
```
修改代码，将代码进行修改进行指令集优化
```java 

public class Queue {

    private final transient ReentrantLock r = new ReentrantLock();

    public void t() {
        ReentrantLock rt = r;
        rt.lock();
        try {
            System.out.println(1);
        } finally {
            rt.unlock();
        }
    }
}


```

进行编译和反编译查看字节码

```
 public void t();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #4                  // Field r:Ljava/util/concurrent/locks/ReentrantLock;
         4: astore_1
         // 获取到锁
         5: aload_1
         // 将锁放到index = 1  局部变量表 locals
         6: invokevirtual #5                  // Method java/util/concurrent/locks/ReentrantLock.lock:()V
         9: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
        12: iconst_1
        13: invokevirtual #7                  // Method java/io/PrintStream.println:(I)V
        16: aload_1
        17: invokevirtual #8                  // Method java/util/concurrent/locks/ReentrantLock.unlock:()V
        20: goto          30
        23: astore_2
        24: aload_1
        25: invokevirtual #8                  // Method java/util/concurrent/locks/ReentrantLock.unlock:()V
        28: aload_2
        29: athrow
        30: return
      Exception table:
         from    to  target type
             9    16    23   any
      LineNumberTable:
        line 14: 0
        line 15: 5
        line 17: 9
        line 19: 16
        line 20: 20
        line 19: 23
        line 20: 28
        line 21: 30
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 23
          // index = 1 这里放了锁
          locals = [ class com/example/zk/Queue, class java/util/concurrent/locks/ReentrantLock ]
          stack = [ class java/lang/Throwable ]
        frame_type = 6 /* same */

```
从这里发现指令集发生了变化,如下，每次获取锁的时候都使用aload_1从局部变量表里面获取，这里是Doug lea对指令集的优化，HiKariCP也是靠字节码优化提升速度的。
可以自行查看源码。
```
         0: aload_0
         1: getfield      #4                  // Field r:Ljava/util/concurrent/locks/ReentrantLock;
         4: astore_1
         // 获取到锁
         5: aload_1

```
从这里又出现了两个问题，为何Doug Lea会将锁在方法体内写为final而数据却未写为final。经过google也没找到很好的答案，以下内容仅仅为猜想。
```
  // 这里为什么会是final
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 这里的q也是final的，为啥不局部变量话
            return q.size();
        } finally {
            lock.unlock();
        }
```

-  局部变量表的重复利用问题，可以参考深入理解虚拟机或[JVM Anatomy Quark #8()](https://shipilev.net/jvm/anatomy-quarks/8-local-var-reachability/)
   Google查到如果局部变量为final,权限变成了ReadOnly access,为了防止局部向量嘈被重复利用问题
-  为何这里` private final PriorityQueue<E> q = new PriorityQueue<E>();` 也是final为啥不在局部变量也命名为final,原因`PriorityQueue`不可变，
   但是其底层数据`transient Object[] queue`在扩容时候会变地址(数组特性)。源码如下：
``` java 

// 对象地址不会变，但是内部数据汇编
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    private static final long serialVersionUID = -7720805057305804111L;

    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    // 这里内存地址汇编
    transient Object[] queue; // non-private to simplify nested class access
}

// 内部数据不会变
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    // 不会改变内存地址
    private final Sync sync;
}

```
从这里猜，使用了final可能会减少线程缓存数据和主内存数据比较及拉取主内存数据到线程缓存中，也就是经典的内存模型
可以参考Java并发编程艺术 方腾飞　魏鹏　程晓明著

### 字节码级别优化
Doug lea大神就是牛逼，无敌的存在，代码看不明白啊。
著作权归作者所有，商业转载请联系作者获得授权，非商业转载请注明出处。