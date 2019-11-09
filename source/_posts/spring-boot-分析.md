---
title: spring-boot 分析
date: 2019-07-27 12:26:44
tags: [java,spring,源码]
categories: 源码阅读
---

{% asset_img springboot.svg This is an example image %}

## `org.springframework.context.ApplicationContextInitializer`
1. `org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer`
    这里面添加了一个 `BeanFactoryPostProcessor` 实现类 `ConfigurationWarningsPostProcessor`
    主要是打印启动过程中的警告信息的
2. `org.springframework.boot.context.ContextIdApplicationContextInitializer`
    这里处理 `ContextId`    
3. `org.springframework.boot.context.config.DelegatingApplicationContextInitializer`
    这里处理 `ApplicationContextInitializer` 的实现类
    这个接口就是一个 `initialize` 方法，参数是 `ConfigurableApplicationContext` 的子类    
    当然，不是所有的 `ApplicationContextInitializer` 都会被调用，可以通过参数 `context.initializer.classes` 指定要调用的实现类，多个用逗号分隔
4. `org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer`
    这个类也实现了 `ApplicationListener` 接口，用来监听 `WebServerInitializedEvent` 时间    
    作用是处理设置 `server.port`
    
## `org.springframework.context.ApplicationListener`    
1. `org.springframework.boot.ClearCachesApplicationListener`    
    监听的事件是 `ContextRefreshedEvent` ，也就是 `refersh` 方法完成后    
    清理反射使用的缓存
2.    `org.springframework.boot.builder.ParentContextCloserApplicationListener`
    监听的事件是 `ParentContextAvailableEvent`    
    这个是在 `ParentContextApplicationContextInitializer` 里面触发的，`ApplicationContextInitializer` 上面说过了    
    这里处理的是有 parent 的情况，使用 `SpringApplicationBuilder` 的时候，可以指定 parent    
3.    `org.springframework.boot.context.FileEncodingApplicationListener`
    监听的事件是 `ApplicationEnvironmentPreparedEvent`    
    就是实例化 `ConfigurableEnvironment` 后    
    作用是，检查 `spring.mandatory-file-encoding` 参数强制指定的字符集与 `file.encoding` 系统参数的字符集是否一样，如果不愿意，就异常
4. `org.springframework.boot.context.config.AnsiOutputApplicationListener`
    监听的事件是 `ApplicationEnvironmentPreparedEvent`    
    就是实例化 `ConfigurableEnvironment` 后    
    处理参数 `spring.output.ansi.enabled` 和 `spring.output.ansi.console-available` 就是控制彩色输出
5. `org.springframework.boot.context.config.ConfigFileApplicationListener`
    监听的事件是 `ApplicationEnvironmentPreparedEvent` 和 `ApplicationPreparedEvent`
    就是实例化 `ConfigurableEnvironment` 后和注册完主类为 BeanDefinition 后
    处理事件 `ApplicationEnvironmentPreparedEvent`
        加载 `EnvironmentPostProcessor` 的实现类
        默认配置有
        `org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor`
        `org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor`
        `org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor`
        在加上自身，调用 `postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application)` 方法
        这里 *关键* ，后面再细看
    处理事件 `ApplicationPreparedEvent`
        这里添加了一个 `BeanFactoryPostProcessor` 的实例 `PropertySourceOrderingPostProcessor`
        作用是将 `defaultProperties` 的 `PropertySource` 添加到最后
6. `org.springframework.boot.context.config.DelegatingApplicationListener`
    监听的事件是 `ApplicationEnvironmentPreparedEvent`
    在 `ApplicationEnvironmentPreparedEvent` 事件的时候，获取配置 `context.listener.classes` 指定的 `ApplicationListener` 类，如果有的话，就转发其他所有事件
7. `org.springframework.boot.context.logging.ClasspathLoggingApplicationListener`
    监听的事件是 `ApplicationEnvironmentPreparedEvent` 或者 `ApplicationFailedEvent`    
    就是打印一个 `classpath` 日志
8. `org.springframework.boot.context.logging.LoggingApplicationListener`
    监听的事件 `ApplicationStartingEvent.class, ApplicationEnvironmentPreparedEvent.class, ApplicationPreparedEvent.class, ContextClosedEvent.class, ApplicationFailedEvent.class`
    处理日志配置和等级的
9.    `org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener`
    `Liquibase` 是数据库版本控制，具体里面做啥，没看。
    
## `org.springframework.boot.env.EnvironmentPostProcessor`    
上面第 5 条会触发这里，继续看    
1.    `org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor`
    云原生相关的，后面看 `spring-cloud` 的时候再细看
2.    `org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor`
    作用就是找参数 `spring.application.json` 或者 `SPRING_APPLICATION_JSON` 指定的 `json` 文件，加载配置，优先级高于系统参数
3.    `org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor`
    处理 `systemEnvironment`
4. `org.springframework.boot.context.config.ConfigFileApplicationListener`
    这里就是加载 `application.properties` 的地方
    如果直接指定了 `spring.config.location` 那么只去加载这里
    如果指定了 `spring.config.additional-location` 额外加载的文件，先加载这里
    加上默认的 `classpath:/,classpath:/config/,file:./,file:./config/`
    
    一般上面指定的是目录，然后在目录下，找文件名，默认是 `application` , 可以通过 `spring.config.name` 参数来修改文件名
    剩下的就是加载过程了
    加载过程中，先加载 `spring.factory` 的 `org.springframework.boot.env.PropertySourceLoader` , 默认有两个实现类    
    `org.springframework.boot.env.PropertiesPropertySourceLoader`   `properties` 和 `xml`
    `org.springframework.boot.env.YamlPropertySourceLoader`      `yml` 和 `yaml`
    好吧，看名字就知道分别处理 `properteis` 和 `yaml` 文件了
    
以上就是 `spring-boot` 的核心配置
我们主要知道了，`spring-boot` 的启动过程，以及通过哪些扩展接口进行增强
接下来看看 `spring-boot-autoconfigure`

## `org.springframework.context.ApplicationContextInitializer`
1. `org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer`
    注册了一个 `BeanFactoryPostProcessor` 实现类 `CachingMetadataReaderFactoryPostProcessor`    
    作用，注册了一个 `SharedMetadataReaderFactoryBean` 的 `BeanDefinition`
2. `org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener`
    作用就是打印一些自动注册的日志
    
## `org.springframework.context.ApplicationListener`
1. `org.springframework.boot.autoconfigure.BackgroundPreinitializer`
    监听的事件是 `SpringApplicationEvent` 这是一个抽象类 ，上面说的 `spring-boot` 相关的事件类，大部分都是这个类的子类    
    作用就是启动一个线程去做一些初始化，类相关的
