---
author: renfakai
layout: post
title: 原创-spring 
date: 2020-04-20
categories: spring
tags: [java，spring]
description: policy 。
---

### 问题
我们是否中了Spring的毒，写代码什么时候开始编程了Controller -> Service -> Dao （ 1:1:1 比例）编写，案例代码布局如下:

```java
.
├── service
│   ├── OtherService.java
│   └── impl
│       └── OtherServiceImpl.java
└── web
    ├── OtherController.http
    └── OtherController.java

```

先看一眼dto:
```java 

@Data
public class HelloRequest {

    private String policy;

    private String name;
}

```
让我门来看一下Controller代码:  

```java

@RequestMapping("/other")
@RestController
public class OtherController {

    @Autowired
    private OtherService otherService;

    @GetMapping("/hello")
    public String hello(HelloRequest request) {
        return otherService.print(request);
    }
}

```

让我门来看一下Service代码:  

```java

public interface OtherService {


    /**
     * @param helloRequest 请求
     */
    String print(HelloRequest helloRequest);
}


@Service
public class OtherServiceImpl implements OtherService {

    @Override
    public String print(HelloRequest helloRequest) {

        if ("hello".equalsIgnoreCase(helloRequest.getPolicy())) {
            return helloRequest.getName() + "hello";
        }

        return helloRequest.getName() + "world";
    }
}

```

然后进行测试，完全符合预期，测试结果如下:

```java 

// 进行测试
GET http://localhost:8080/other/world?name=ren&policy=hello


###
GET http://localhost:8080/other/hello?name=ren&policy=world


Testing started at 10:01 ...

HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 8
Date: Mon, 20 Apr 2020 02:01:50 GMT
Keep-Alive: timeout=60
Connection: keep-alive

> 2020-04-20T100150.200.txt > renhello

Response code: 200; Time: 54ms; Content length: 8 bytes
HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 8
Date: Mon, 20 Apr 2020 02:01:50 GMT
Keep-Alive: timeout=60
Connection: keep-alive

> 2020-04-20T100150-1.200.txt > renworld

Response code: 200; Time: 56ms; Content length: 8 bytes

```

注意Service的实现，这里如果请求的policy为hello，则进行hello处理，如果这里为world处理，则进行world返回，
未来进行扩展业务，我们也就编程了下面这个样子:


```java

  // 这段代码完全违背了OCP原则，扩展和测试增加了很大的难度。
  if ("hello".equalsIgnoreCase(helloRequest.getPolicy())) {
            return helloRequest.getName() + "hello";
        } else if () {
            // dosomething
        } else if () {
            // dosomething ....
        }

        return helloRequest.getName() + "world";

```

我怎么把Java的多态丢了，好像是这个样子，然后你进行了重构。

```javas
.
├── service
│   ├── SimpleService.java
│   └── impl
│       ├── HelloSimpleServiceImpl.java
│       └── WorldSimpleServiceImpl.java
└── web
    ├── SimpleController.http
    └── SimpleController.java

```

让我们来看一下一个Controller:

```java

@RequestMapping("/simple")
@RestController
public class SimpleController {

    
    // Resource = Autowired+Qualifier 

    /**
     * style1 Autowired+Qualifier
     */
    @Autowired
    @Qualifier("helloSimpleServiceImpl")
    private SimpleService helloServvice;

    /**
     *  style2 Resource = Autowired+Qualifier
     */
    @Resource(name = "worldSimpleServiceImpl")
    private SimpleService worldServvice;

    @GetMapping("/hello")
    public String hello(HelloRequest request) {
        return helloServvice.print(request);
    }

    @GetMapping("/world")
    public String world(HelloRequest request) {
        return worldServvice.print(request);
    }
}

```

然后看一下Service:  

```java

public interface SimpleService {


    /**
     * @param helloRequest 请求
     */
    String print(HelloRequest helloRequest);
}

@Service(value = "helloSimpleServiceImpl")
public class HelloSimpleServiceImpl implements SimpleService {

    @Override
    public String print(HelloRequest helloRequest) {
        return helloRequest.getName() + "hello";
    }
}


@Service(value = "worldSimpleServiceImpl")
public class WorldSimpleServiceImpl implements SimpleService {

    @Override
    public String print(HelloRequest helloRequest) {
        return helloRequest.getName() + "world";
    }
}

```

进行测试:  

```

Testing started at 10:08 ...

HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 8
Date: Mon, 20 Apr 2020 02:08:36 GMT
Keep-Alive: timeout=60
Connection: keep-alive

> 2020-04-20T100836.200.txt > renworld

Response code: 200; Time: 19ms; Content length: 8 bytes
HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 8
Date: Mon, 20 Apr 2020 02:08:36 GMT
Keep-Alive: timeout=60
Connection: keep-alive

> 2020-04-20T100836-1.200.txt > renhello

Response code: 200; Time: 53ms; Content length: 8 bytes

```

但是我们发现好像问题没有解决，只是把Service问题转移到了Controller，添加一个策略我们都需要添加一个控制器，无法根据数据进行动态选择策略。  
下一章我们将用策略(多态)+享元模式解决这个问题。

### [Github地址](https://github.com/sona0402/Polymorphism)
 