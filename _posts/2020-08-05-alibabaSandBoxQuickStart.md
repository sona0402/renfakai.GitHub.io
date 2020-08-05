---
author: renfakai
layout: post
title: AlibabaSandboxQuickStart
date: 2020-08-05
categories: chanos
tags: [java，chanos]
description: 熟悉混沌工程,快速与公司现有业务整合
---

阿里巴巴ChanosMonkey案例使用测试，流程与alibaba文档一致。

##  快速入门-sandbox
./blade底层依赖sandbox，下面介绍直接通过sandbox来实现演练，如果已经克隆本项目，直接从编译阶段开始即可。
### 准备
- clone
```

    git clone https://github.com/chaosblade-io/chaosblade-exec-jvm

````
- 编译
```
    cd chaosblade-exec-jvm
    make build

```
- 启动wf服务

```
     renfakai@renfakaideMacBook-Pro  ~/github/wuxian  jps -lm
    768 org.jetbrains.idea.maven.server.RemoteMavenServer36
    643 
    869 sun.tools.jps.Jps -lm
    762 org.jetbrains.idea.maven.server.RemoteMavenServer36
    799 org.codehaus.classworlds.Launcher -Didea.version=2020.2 clean install jetty:run
     renfakai@renfakaideMacBook-Pro  ~/github/wuxian 
 
```

- 挂载agent： -p 799 是被攻击应用的jvm进程号

```
    cd ./build-target/chaosblade-0.6.0/lib/sandbox/bin/ && ./sandbox.sh -p 799
    
    # result 
                NAMESPACE : default
                          VERSION : 1.3.1
                             MODE : ATTACH
                      SERVER_ADDR : 0.0.0.0
                      SERVER_PORT : 49647
                   UNSAFE_SUPPORT : ENABLE
                     SANDBOX_HOME : /Users/renfakai/github/chaosblade-exec-jvm/build-target/chaosblade-0.6.0/lib/sandbox/bin/..
                SYSTEM_MODULE_LIB : /Users/renfakai/github/chaosblade-exec-jvm/build-target/chaosblade-0.6.0/lib/sandbox/bin/../module
                  USER_MODULE_LIB : /Users/renfakai/github/chaosblade-exec-jvm/build-target/chaosblade-0.6.0/lib/sandbox/sandbox-module;~/.sandbox-module;
              SYSTEM_PROVIDER_LIB : /Users/renfakai/github/chaosblade-exec-jvm/build-target/chaosblade-0.6.0/lib/sandbox/bin/../provider
```

- 激活模块

```
    ./sandbox.sh -p 799
    
    # result
    total 1 module activated.

```

###  混沌实验

此阶段可反复侵入和销毁，suid为一次实验的上下文唯一， 一个简单的例子，对servlet容器，api接口延迟3秒

- 创建混沌实验

```

        curl -v -X post http://127.0.0.1:49647/sandbox/default/module/http/chaosblade/create -H 'Content-Type:application/json' -d '{"action":"delay","target":"servlet","suid":"1110","time":"3000","requestpath":"/hello"}' | python -m json.tool
        
        # result
        
        Note: Unnecessary use of -X or --request, POST is already inferred.
          % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                         Dload  Upload   Total   Spent    Left  Speed
          0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying 127.0.0.1...
        * TCP_NODELAY set
        * Connected to 127.0.0.1 (127.0.0.1) port 49647 (#0)
        > post /sandbox/default/module/http/chaosblade/create HTTP/1.1
        > Host: 127.0.0.1:49647
        > User-Agent: curl/7.64.1
        > Accept: */*
        > Content-Type:application/json
        > Content-Length: 88
        >
        } [88 bytes data]
        * upload completely sent off: 88 out of 88 bytes
        < HTTP/1.1 200 OK
        < Content-Length: 177
        < Server: Jetty(8.y.z-SNAPSHOT)
        <
        { [177 bytes data]
        100   265  100   177  100    88    702    349 --:--:-- --:--:-- --:--:--  1051
        * Connection #0 to host 127.0.0.1 left intact
        * Closing connection 0
        {
            "code": 200,
            "result": "{\"action\":{\"name\":\"delay\"},\"actionName\":\"delay\",\"matcher\":{\"matchers\":{\"requestpath\":\"/hello\"}},\"target\":\"servlet\"}",
            "success": true
        }

```

编写一个Springboot项目,目录如下:

```
        ├── HELP.md
        ├── mvnw
        ├── mvnw.cmd
        ├── pom.xml
        ├── src
        │   ├── main
        │   │   ├── java
        │   │   │   └── com
        │   │   │       └── renfakai
        │   │   │           └── sona
        │   │   │               └── AlibabaChanosApplication.java
        │   │   └── resources
        │   │       ├── application.properties
        │   │       ├── static
        │   │       └── templates
        │   └── test
        │       └── java
        │           └── com
        │               └── renfakai
        │                   └── sona
```

pom.xml如下

```
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.0.1.RELEASE</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
        <groupId>com.renfakai</groupId>
        <artifactId>sona</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <name>alibabachanos</name>
        <description>chanosTest</description>
    
        <properties>
            <java.version>1.8</java.version>
        </properties>
    
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
    
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
                <exclusions>
                    <exclusion>
                        <groupId>org.junit.vintage</groupId>
                        <artifactId>junit-vintage-engine</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
        </dependencies>
    
        <build>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                </plugin>
            </plugins>
        </build>
    
    </project>

```

```java
    // 代码
    package com.renfakai.sona;
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.http.ResponseEntity;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;
    
    @RestController
    @SpringBootApplication
    public class AlibabaChanosApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(AlibabaChanosApplication.class, args);
        }
    
        @GetMapping("/hello")
        public ResponseEntity<String> hello() {
            return ResponseEntity.ok("helloWorld");
        }
    
    }
```



此时访问Java应用/index应用将延迟3秒后响应，` GET http://localhost:8080/hello` 端口在挂载agent返回，如下

```
    GET http://localhost:8080/hello
    
    HTTP/1.1 200 
    Content-Type: text/plain;charset=UTF-8
    Content-Length: 10
    Date: Wed, 05 Aug 2020 12:40:34 GMT
    
    helloWorld
    
    // 这里延迟了3秒,请求返回时间为3092ms
    Response code: 200; Time: 3092ms; Content length: 10 bytes
```

- 销毁

```
    curl -X post http://127.0.0.1:49647/sandbox/default/module/http/chaosblade/destroy -H 'Content-Type:application/json' -d '{"action":"delay","target":"servlet","suid":"110"}' | python -m json.tool
    
    # result
    
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100    98  100    47  100    51  15666  17000 --:--:-- --:--:-- --:--:-- 32666
    {
        "code": 200,
        "result": "success",
        "success": true
    }
```

### 卸载

- 卸载agent

```
    ./sandbox.sh -p 799 -S
    
    # result
    
    jvm-sandbox[default] shutdown finished.

```

