---
author: renfakai
layout: post
title: SpringbootChanos源码学习
date: 2020-07-31
categories: Springboot
tags: [java，Springboot]
description: 读源码
---


混沌工程

混沌工程对应用分类

- 操作Application内部，对字节码进行操作，实现OOM，延迟。<br/>
```
        1.alibaba  sandbox
        2.de.codecentric.spring.boot.chaos.monkey
```
* 操作容器和容器环境，主要是对容器网卡，cpu打满，磁盘满等。

```
	1. Netflix Chaos Monkey
	2. alibaba chaosblade
```

    [图片来源]: https://zhuanlan.zhihu.com/p/90294032
    ![avatar](/img/chanos/chanos_1.jpg)  

Application内操作字节码常用方式  alibaba  sandbox介绍

1. **静态编织**：静态编织发生在字节码生成时根据一定框架的规则提前将AOP字节码插入到目标类和方法中，实现AOP；
1. **动态编织**：动态编织则允许在JVM运行过程中完成指定方法的AOP字节码增强.常见的动态编织方案大多采用重命名原有方法，再新建一个同签名的方法来做代理的工作模式来完成AOP的功能(常见的实现方案如CgLib)，但这种方式存在一些应用边界：
   - **侵入性**：对被代理的目标类需要进行侵入式改造。比如：在Spring中必须是托管于Spring容器中的Bean
   - **固化性**：目标代理方法在启动之后即固化，无法重新对一个已有方法进行AOP增强

要解决`无侵入`的特性需要AOP框架具备 **在运行时完成目标方法的增强和替换**。在JDK的规范中运行期重定义一个类必须准循以下原则
  1. 不允许新增、修改和删除成员变量
  1. 不允许新增和删除方法
  1. 不允许修改方法签名

静态编织代表作品de.codecentric.spring.boot.chaos.monkey，使用了SpringAop进行处理

   1. **de.codecentric.monkey**使用了Spring特性简化了字节码的操作，其使用spring-boot-auto-config来加载Bean,配置文件如下。

```
    /Users/renfakai/.m2/repository/de/codecentric/chaos-monkey-spring-boot/2.0.1/chaos-monkey-spring-boot-2.0.1.jar!/META-INF/spring.factories
    
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
      de.codecentric.spring.boot.chaos.monkey.configuration.ChaosMonkeyConfiguration
```

   2.配置文件外挂，使用了元数据，可以参考`spring-configuration-metadata.json`,

   3.使用了springboot autoconfig

```
    .
    ├── META-INF
     └── spring.factories
    └── chaos-logo.txt
    
     spring.factories
     org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
      de.codecentric.spring.boot.chaos.monkey.configuration.ChaosMonkeyConfiguration

```

  4.看下自动化Bean

```
    
    
    /** @author Benjamin Wilms */
    @Configuration
    @Profile("chaos-monkey")
    @EnableConfigurationProperties({
      ChaosMonkeyProperties.class,
      AssaultProperties.class,
      WatcherProperties.class
    })
    @EnableScheduling
    public class ChaosMonkeyConfiguration {
    
      private static final Logger Logger = LoggerFactory.getLogger(ChaosMonkeyConfiguration.class);
    
      private final ChaosMonkeyProperties chaosMonkeyProperties;
    
      private final WatcherProperties watcherProperties;
    
      private final AssaultProperties assaultProperties;
    
      public ChaosMonkeyConfiguration(
          ChaosMonkeyProperties chaosMonkeyProperties,
          WatcherProperties watcherProperties,
          AssaultProperties assaultProperties) {
        this.chaosMonkeyProperties = chaosMonkeyProperties;
        this.watcherProperties = watcherProperties;
        this.assaultProperties = assaultProperties;
    
        // 这个是作者logo
        try {
          String chaosLogo =
              StreamUtils.copyToString(
                  new ClassPathResource("chaos-logo.txt").getInputStream(), Charset.defaultCharset());
          Logger.info(chaosLogo);
        } catch (IOException e) {
          Logger.info("Chaos Monkey - ready to do evil");
        }
      }
    
      //  数据统计使用
      @Bean
      @ConditionalOnClass(name = "io.micrometer.core.instrument.MeterRegistry")
      public Metrics metrics() {
        return new Metrics();
      }
    
      // springEvent发布事件，观察者设计模式，解耦合，推送数据到观察者，观察者统计到Metrics
      @Bean
      public MetricEventPublisher publisher() {
        return new MetricEventPublisher();
      }
    
      // 聚合所有的属性，monkey开关，攻击属性，	被攻击范围开关
      @Bean
      public ChaosMonkeySettings settings() {
        return new ChaosMonkeySettings(chaosMonkeyProperties, assaultProperties, watcherProperties);
      }
    
      // 延迟攻击
      @Bean
      public LatencyAssault latencyAssault() {
        return new LatencyAssault(settings(), publisher());
      }
    
      // 异常攻击
      @Bean
      public ExceptionAssault exceptionAssault() {
        return new ExceptionAssault(settings(), publisher());
      }
    
      // app 杀死攻击
      @Bean
      public KillAppAssault killAppAssault() {
        return new KillAppAssault(settings(), publisher());
      }
    
      // 内存攻击
      @Bean
      public MemoryAssault memoryAssault() {
        return new MemoryAssault(Runtime.getRuntime(), settings(), publisher());
      }
    
      // 设置所有的攻击是否开启，之前使用的是 @Conditional(AttackControllerCondition.class)
      @Bean
      public ChaosMonkeyRequestScope chaosMonkeyRequestScope(
          List<ChaosMonkeyRequestAssault> chaosMonkeyAssaults, List<ChaosMonkeyAssault> allAssaults) {
        return new ChaosMonkeyRequestScope(settings(), chaosMonkeyAssaults, allAssaults, publisher());
      }
    
      //  组合方式的线程
      @Bean
      public ChaosMonkeyScheduler scheduler(
          @Nullable TaskScheduler scheduler, ChaosMonkeyRuntimeScope runtimeScope) {
        ScheduledTaskRegistrar registrar = null;
        if (scheduler != null) {
          registrar = new ScheduledTaskRegistrar();
          registrar.setTaskScheduler(scheduler);
        }
        return new ChaosMonkeyScheduler(registrar, assaultProperties, runtimeScope);
      }
    
      // 设置运行范围
      @Bean
      public ChaosMonkeyRuntimeScope chaosMonkeyRuntimeScope(
          List<ChaosMonkeyRuntimeAssault> chaosMonkeyAssaults) {
        return new ChaosMonkeyRuntimeScope(settings(), chaosMonkeyAssaults);
      }
    
      //  产生切面，根据配置选择是否生效，之前版本是如果不开启则不产生相应的切面
      @Bean
      @DependsOn("chaosMonkeyRequestScope")
      public SpringControllerAspect controllerAspect(ChaosMonkeyRequestScope chaosMonkeyRequestScope) {
        return new SpringControllerAspect(chaosMonkeyRequestScope, publisher(), watcherProperties);
      }
    
      @Bean
      @DependsOn("chaosMonkeyRequestScope")
      public SpringRestControllerAspect restControllerAspect(
          ChaosMonkeyRequestScope chaosMonkeyRequestScope) {
        return new SpringRestControllerAspect(chaosMonkeyRequestScope, publisher(), watcherProperties);
      }
    
      @Bean
      @DependsOn("chaosMonkeyRequestScope")
      public SpringServiceAspect serviceAspect(ChaosMonkeyRequestScope chaosMonkeyRequestScope) {
        return new SpringServiceAspect(chaosMonkeyRequestScope, publisher(), watcherProperties);
      }
    
      @Bean
      @DependsOn("chaosMonkeyRequestScope")
      public SpringComponentAspect componentAspect(ChaosMonkeyRequestScope chaosMonkeyRequestScope) {
        return new SpringComponentAspect(chaosMonkeyRequestScope, publisher(), watcherProperties);
      }
    
      //  @ConditionalOnClass(name = "org.springframework.data.repository.Repository")
      //  很细心的地方只有出现这个类的时候才会出现切面
      @Bean
      @DependsOn("chaosMonkeyRequestScope")
      @ConditionalOnClass(name = "org.springframework.data.repository.Repository")
      // Creates aspects that match interfaces annotated with @Repository
      public SpringRepositoryAspectJPA repositoryAspectJpa(
          ChaosMonkeyRequestScope chaosMonkeyRequestScope) {
        return new SpringRepositoryAspectJPA(chaosMonkeyRequestScope, publisher(), watcherProperties);
      }
    
      @Bean
      @DependsOn("chaosMonkeyRequestScope")
      // creates aspects that match simple classes annotated with @repository
      public SpringRepositoryAspectJDBC repositoryAspectJdbc(
          ChaosMonkeyRequestScope chaosMonkeyRequestScope) {
        return new SpringRepositoryAspectJDBC(chaosMonkeyRequestScope, publisher(), watcherProperties);
      }
    
      // 资源文件rest读取和修改，使用httpclient
      @Bean
      @ConditionalOnMissingBean
      @ConditionalOnEnabledEndpoint
      public ChaosMonkeyRestEndpoint chaosMonkeyRestEndpoint(
          ChaosMonkeyRuntimeScope runtimeScope, ChaosMonkeyScheduler scheduler) {
        return new ChaosMonkeyRestEndpoint(settings(), runtimeScope, scheduler);
      }
    
      // jmx方式
      @Bean
      @ConditionalOnMissingBean
      @ConditionalOnEnabledEndpoint
      public ChaosMonkeyJmxEndpoint chaosMonkeyJmxEndpoint() {
        return new ChaosMonkeyJmxEndpoint(settings());
      }
    }
    
```

5.assaults代码解读,攻击主要分为3类，其符合ocp原则，所以看一个实现即可。

 ![avatar](/img/chanos/ChaosMonkeyAssault.png)  

* 运行时攻击 继承ChaosMonkeyAssault，实现为ChaosMonkeyRuntimeAssault子类
* 请求攻击	继承ChaosMonkeyAssault，实现为ChaosMonkeyRequestAssault子类
* 延迟攻击  ChaosMonkeyLatencyAssaultExecutor子类

```
         public interface ChaosMonkeyAssault {
        
          // 是否启用
          boolean isActive();
        
          // 攻击
          void attack();
        }
        
        
        
        // 延迟攻击
        public class LatencyAssault implements ChaosMonkeyRequestAssault {
        
          private static final Logger Logger = LoggerFactory.getLogger(LatencyAssault.class);
        
          // 配置文件
          private final ChaosMonkeySettings settings;
        
          // 执行器
          private final ChaosMonkeyLatencyAssaultExecutor assaultExecutor;
        
          // 数据埋点生产者
          private MetricEventPublisher metricEventPublisher;
        
          private AtomicInteger atomicTimeoutGauge;
        
          public LatencyAssault(
              ChaosMonkeySettings settings,
              MetricEventPublisher metricEventPublisher,
              ChaosMonkeyLatencyAssaultExecutor executor) {
            this.settings = settings;
            this.metricEventPublisher = metricEventPublisher;
            this.atomicTimeoutGauge = new AtomicInteger(0);
            this.assaultExecutor = executor;
          }
        
         // 产生bean的时候传入了配置
          public LatencyAssault(ChaosMonkeySettings settings, MetricEventPublisher metricEventPublisher) {
            this(settings, metricEventPublisher, new LatencyAssaultExecutor());
          }
        
        // 配置是否启用
          @Override
          public boolean isActive() {
            return settings.getAssaultProperties().isLatencyActive();
          }
        
          @Override
          public void attack() {
            Logger.debug("Chaos Monkey - timeout");
        
            atomicTimeoutGauge.set(determineLatency());
        
            // metrics 发送统计数据事件
            if (metricEventPublisher != null) {
              metricEventPublisher.publishMetricEvent(MetricType.LATENCY_ASSAULT);
              metricEventPublisher.publishMetricEvent(MetricType.LATENCY_ASSAULT, atomicTimeoutGauge);
            }
        
            assaultExecutor.execute(atomicTimeoutGauge.get());
          }
        
          private int determineLatency() {
            final int latencyRangeStart = settings.getAssaultProperties().getLatencyRangeStart();
            final int latencyRangeEnd = settings.getAssaultProperties().getLatencyRangeEnd();
        
            if (latencyRangeStart == latencyRangeEnd) {
              return latencyRangeStart;
            } else {
              return ThreadLocalRandom.current().nextInt(latencyRangeStart, latencyRangeEnd);
            }
          }
        }
        
          // 延迟睡眠
          @Override
          public void execute(long durationInMillis) {
            try {
              Thread.sleep(durationInMillis);
            } catch (InterruptedException e) {
              // do nothing
            }
          }
```
  

7.component包

```
        .
        // 请求范围配置
        ├── ChaosMonkeyRequestScope.java
        // 运行配置范围
        ├── ChaosMonkeyRuntimeScope.java
        // 调度线程池
        ├── ChaosMonkeyScheduler.java
        // 元数据推送，生产者
        ├── MetricEventPublisher.java
        // 元数据类型
        ├── MetricType.java
        // 统计
        └── Metrics.java
```

8.configuration包

```
        .
          // 攻击异常
        ├── AssaultException.java
          // 异常注解
        ├── AssaultExceptionConstraint.java
          // 异常验证
        ├── AssaultExceptionValidator.java
          // 配置文件
        ├── AssaultProperties.java
          // 攻击延迟范围
        ├── AssaultPropertiesLatencyRangeConstraint.java
          // 攻击延迟范围验证
        ├── AssaultPropertiesLatencyRangeValidator.java
          // 自动配置文件
        ├── ChaosMonkeyConfiguration.java
          // 启动编制
        ├── ChaosMonkeyLoadTimeWeaving.java
          // 配置文件
        ├── ChaosMonkeyProperties.java
        ├── ChaosMonkeySettings.java
        └── WatcherProperties.java
```

9.endpoints

```
        //  资源文件更新
        ├── AssaultPropertiesUpdate.java
        // jmx开启的话，可以修改文件
        ├── ChaosMonkeyJmxEndpoint.java
        // http主要是资源文修改
        ├── ChaosMonkeyRestEndpoint.java
        ├── ChaosMonkeySettingsDto.java
        // 修改文件类
        └── WatcherPropertiesUpdate.java
```

10.event

```
    
        事件 springevent事件
        .
        └── MetricEvent.java
         
```

11.watcher

```
        // 基类
        ├── ChaosMonkeyBaseAspect.java
        // 不同实现，分别支持Spring各个层的处理
        ├── SpringComponentAspect.java
        ├── SpringControllerAspect.java
        ├── SpringRepositoryAspectJDBC.java
        ├── SpringRepositoryAspectJPA.java
        ├── SpringRestControllerAspect.java
        └── SpringServiceAspect.java
```

![avatar](/img/chanos/ChaosMonkeyBaseAspect.png)  


```

        abstract class ChaosMonkeyBaseAspect {
        
          // 子类排除到自己使用
          @Pointcut("within(de.codecentric.spring.boot.chaos.monkey..*)")
          public void classInChaosMonkeyPackage() {}
        
          // 所有公共方法
          @Pointcut("execution(* *.*(..))")
          public void allPublicMethodPointcut() {}
        
          String calculatePointcut(String target) {
            return target.replaceAll("\\(\\)", "").replaceAll("\\)", "").replaceAll("\\(", ".");
          }
        
          String createSignature(MethodSignature signature) {
            return signature.getDeclaringTypeName() + "." + signature.getMethod().getName();
          }
        }
        
        
        
        @Aspect
        @AllArgsConstructor
        @Slf4j
        public class SpringControllerAspect extends ChaosMonkeyBaseAspect {
        
          private final ChaosMonkeyRequestScope chaosMonkeyRequestScope;
        
          private MetricEventPublisher metricEventPublisher;
        
          private WatcherProperties watcherProperties;
        
          // 包含controller注解
          @Pointcut("within(@org.springframework.stereotype.Controller *)")
          public void classAnnotatedWithControllerPointcut() {}
        
          // 该类是Controller, 公共方法，把chaosmoney包排除掉
          @Around(
              "classAnnotatedWithControllerPointcut() && allPublicMethodPointcut() && !classInChaosMonkeyPackage()")
          public Object intercept(ProceedingJoinPoint pjp) throws Throwable {
        
            // 配置开启
            if (watcherProperties.isController()) {
              log.debug("Watching public method on controller class: {}", pjp.getSignature());
        
              // 推送事件
              if (metricEventPublisher != null) {
                metricEventPublisher.publishMetricEvent(
                    calculatePointcut(pjp.toShortString()), MetricType.CONTROLLER);
              }
        
              MethodSignature signature = (MethodSignature) pjp.getSignature();
        
              // 进行攻击
              chaosMonkeyRequestScope.callChaosMonkey(createSignature(signature));
            }
            // 执行原方法
            return pjp.proceed();
          }
        }
    
```

总结：
* 整体很简单，支持资源文件配置，也可以使用jms和http动态修改资源文件。
* 支持运行时和请求攻击

     


