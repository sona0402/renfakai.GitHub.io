---

author: renfakai
layout: post
title: jvm-autobox
date: 2020-12-07
categories: Java
tags: [jvm]
description: jvm
---

# 深入理解虚拟机

```
package com.sona.jvm;

public class AutoBox {
    public static void main(String[] args) {
        Integer a = 1;
        Integer b = 2;
        Integer c = 3;
        Integer d = 3;
        Integer e = 321;
        Integer f = 321;
        Long g = 3L;
        System.out.println(c == d);
        System.out.println(e == f);
        System.out.println(c == (a + b));
        System.out.println(c.equals(a + b));
      	System.out.println(g == (a + b));
        System.out.println(g.equals(a + b));
    }
}
```

其中槽分布数据如下: ![avatar](/img/20210325/local.png)
* `System.out.println(c ==
  d);`测试结果为true，原因主要是Integer对`-128~127`使用了享元设计模式，所以底层是同一个对象。
* `System.out.println(e == f);`测试结果为false，不在享元数据内。
  ![avatar](/img/20210325/flyweight.png)
* ` System.out.println(c == (a +
  b));`结果为true，查看下面字节码，原因是使用c的int值和(a+b)的int值进行对比。

```

 83 aload_3
 84 invokevirtual #8 <java/lang/Integer.intValue>
 87 aload_1
 88 invokevirtual #8 <java/lang/Integer.intValue>
 91 aload_2
 92 invokevirtual #8 <java/lang/Integer.intValue>
 95 iadd
 96 if_icmpne 103 (+7)
 99 iconst_1
100 goto 104 (+4)
103 iconst_0

```

* ` System.out.println(c.equals(a + b));`这个为true;

```
110 aload_3
111 aload_1
112 invokevirtual #8 <java/lang/Integer.intValue>
115 aload_2
116 invokevirtual #8 <java/lang/Integer.intValue>
119 iadd
120 invokestatic #2 <java/lang/Integer.valueOf>
123 invokevirtual #9 <java/lang/Integer.equals>
    其结果使用了享元，即使不使用equals结果也为true
        if (obj instanceof Integer) {
              return value == ((Integer)obj).intValue();
          }
          return false;
    所以结果为true
    
```

* `	System.out.println(g == (a + b));`结果为true;

```
132 aload 7
134 invokevirtual #10 <java/lang/Long.longValue>
137 aload_1
138 invokevirtual #8 <java/lang/Integer.intValue>
141 aload_2
142 invokevirtual #8 <java/lang/Integer.intValue>
145 iadd
146 i2l
147 lcmp
148 ifne 155 (+7)
151 iconst_1
152 goto 156 (+4)
155 iconst_0
  将int结果变为long(i2l),进行对比
```

* `System.out.println(g.equals(a + b));`结果为false，因为类型不同。

```
162 aload 7
164 aload_1
165 invokevirtual #8 <java/lang/Integer.intValue>
168 aload_2
169 invokevirtual #8 <java/lang/Integer.intValue>
172 iadd
173 invokestatic #2 <java/lang/Integer.valueOf>
176 invokevirtual #11 <java/lang/Long.equals>
    
    /**
       * Compares this object to the specified object.  The result is
       * {@code true} if and only if the argument is not
       * {@code null} and is a {@code Long} object that
       * contains the same {@code long} value as this object.
       *
       * @param   obj   the object to compare with.
       * @return  {@code true} if the objects are the same;
       *          {@code false} otherwise.
       */
    public boolean equals(Object obj) {
          if (obj instanceof Long) {
              return value == ((Long)obj).longValue();
          }
          return false;
      } 
```


