---
title: sentinel 学习-上
date: 2019-11-12 20:49:58
tags:
  - java
  - sentinel
  - 学习
categories:
  - 技术笔记
---

# sentinel 学习

## 使用
添加依赖

```xml

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
  <version>0.9.0.RELEASE</version>
</dependency>
```


这是一个 `spring-boot-start` 模块，依赖了   
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel</artifactId>
</dependency>
```

在包里面看到 `spring.factories` ，这里是集成 `spring-boot` 的入口    

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.alibaba.sentinel.SentinelWebAutoConfiguration,\
org.springframework.cloud.alibaba.sentinel.endpoint.SentinelEndpointAutoConfiguration,\
org.springframework.cloud.alibaba.sentinel.custom.SentinelAutoConfiguration,\
org.springframework.cloud.alibaba.sentinel.feign.SentinelFeignAutoConfiguration

org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker=\
org.springframework.cloud.alibaba.sentinel.custom.SentinelCircuitBreakerConfiguration

```

### SentinelWebAutoConfiguration    
这个配置默认开启，作用是添加了一个 `CommonFilter`    
这里过滤器的作用是基于 `url` 的限流   

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
    HttpServletRequest sRequest = (HttpServletRequest) request;
    Entry entry = null;

    Entry methodEntry = null;

    try {
        // 这里是获取 uri  就是排除 contextPath 后面的 uri ，剔除掉 query 参数
        String target = FilterUtil.filterTarget(sRequest);
        // Clean and unify the URL.
        // For REST APIs, you have to clean the URL (e.g. `/foo/1` and `/foo/2` -> `/foo/:id`), or
        // the amount of context and resources will exceed the threshold.
        UrlCleaner urlCleaner = WebCallbackManager.getUrlCleaner();
        // 默认的 UrlCleaner 啥都不做，可以自己做处理， 譬如上面注释里面的路径参数
        if (urlCleaner != null) {
            target = urlCleaner.clean(target);
        }

        // 解析 origin 自己配置的 RequestOriginParser 实现，如果没有，默认就是 "" 了
        // Parse the request origin using registered origin parser.
        String origin = parseOrigin(sRequest);

        // 这里就是限流控制了，context 和 resourceName 都是 uri  类型是 EntryType.IN   ，默认的类型是 OUT
        ContextUtil.enter(target, origin);
        entry = SphU.entry(target, EntryType.IN);

        // 是否基于 HTTP METHOD 区分 uri
        // Add method specification if necessary
        if (httpMethodSpecify) {
          // 如果需要的话，继续请求限流控制
            methodEntry = SphU.entry(sRequest.getMethod().toUpperCase() + COLON + target,
                    EntryType.IN);
        }

        chain.doFilter(request, response);
    } catch (BlockException e) {
        // 阻塞了，代表本次请求被限流了，限流的原因有多个，大部分是基于 QPS 的
        HttpServletResponse sResponse = (HttpServletResponse) response;
        // Return the block page, or redirect to another URL.
        WebCallbackManager.getUrlBlockHandler().blocked(sRequest, sResponse, e);
    } catch (IOException e2) {
        Tracer.trace(e2);
        throw e2;
    } catch (ServletException e3) {
        Tracer.trace(e3);
        throw e3;
    } catch (RuntimeException e4) {
        Tracer.trace(e4);
        throw e4;
    } finally {
        // 释放资源
        if (methodEntry != null) {
            methodEntry.exit();
        }
        if (entry != null) {
            entry.exit();
        }
        ContextUtil.exit();
    }
}
```

### SentinelEndpointAutoConfiguration
这个配置是对 `spring-boot-starter-actuator` 的一个扩展。添加了一个 `SentinelEndpoint`,用来返回展示一些基本配置和规则配置  

```java
@ReadOperation
public Map<String, Object> invoke() {
  final Map<String, Object> result = new HashMap<>();
  if (sentinelProperties.isEnabled()) {

    result.put("appName", AppNameUtil.getAppName());
    result.put("logDir", LogBase.getLogBaseDir());
    result.put("logUsePid", LogBase.isLogNameUsePid());
    result.put("blockPage", WebServletConfig.getBlockPage());
    result.put("metricsFileSize", SentinelConfig.singleMetricFileSize());
    result.put("metricsFileCharset", SentinelConfig.charset());
    result.put("totalMetricsFileCount", SentinelConfig.totalMetricFileCount());
    result.put("consoleServer", TransportConfig.getConsoleServer());
    result.put("clientIp", TransportConfig.getHeartbeatClientIp());
    result.put("heartbeatIntervalMs", TransportConfig.getHeartbeatIntervalMs());
    result.put("clientPort", TransportConfig.getPort());
    result.put("coldFactor", sentinelProperties.getFlow().getColdFactor());
    result.put("filter", sentinelProperties.getFilter());
    result.put("datasource", sentinelProperties.getDatasource());

    final Map<String, Object> rules = new HashMap<>();
    result.put("rules", rules);
    rules.put("flowRules", FlowRuleManager.getRules());
    rules.put("degradeRules", DegradeRuleManager.getRules());
    rules.put("systemRules", SystemRuleManager.getRules());
    rules.put("authorityRule", AuthorityRuleManager.getRules());
    rules.put("paramFlowRule", ParamFlowRuleManager.getRules());
  }
  return result;
}
```

### SentinelAutoConfiguration 
这个是比较核心的配置，与 `spring` 集成的 `bean` 基本都是在这里注册的  
还有 `sentinel` 本身的初始化，也在这里执行    
```java
@PostConstruct
private void init() {
  // 省略一堆配置
  if (StringUtils.hasText(properties.getServlet().getBlockPage())) {
    WebServletConfig.setBlockPage(properties.getServlet().getBlockPage());
  }

  urlBlockHandlerOptional.ifPresent(WebCallbackManager::setUrlBlockHandler);
  urlCleanerOptional.ifPresent(WebCallbackManager::setUrlCleaner);
  requestOriginParserOptional.ifPresent(WebCallbackManager::setRequestOriginParser);

  // earlier initialize
  if (properties.isEager()) {
    // 核心，这里使用 SPI 加载 InitFunc 的实现，然后初始
    InitExecutor.doInit();
  }

}

// 支持 @SentinelResource 注解的 Aspect 类
@Bean
@ConditionalOnMissingBean
public SentinelResourceAspect sentinelResourceAspect() {
  return new SentinelResourceAspect();
}

// 看名字就知道是处理 RestTemplate 的，支持 @SentinelRestTemplate 注解，可以指定 block 或者 fallback 处理方法
// 参数类型为 HttpRequest.class, byte[].class, ClientHttpRequestExecution.class, BlockException.class 
// 返回类型   ClientHttpResponse.class
// 静态方法
// 然后添加 SentinelProtectInterceptor
// 这里请求外部资源的时候，要经过两个 entry  1. hostResource (httpMethod):(schema)://(host):(port) 2. hostResource + path
@Bean
@ConditionalOnMissingBean
@ConditionalOnClass(name = "org.springframework.web.client.RestTemplate")
@ConditionalOnProperty(name = "resttemplate.sentinel.enabled", havingValue = "true", matchIfMissing = true)
public SentinelBeanPostProcessor sentinelBeanPostProcessor(
    ApplicationContext applicationContext) {
  return new SentinelBeanPostProcessor(applicationContext);
}

// 这里是加载配置规则源，规则可以存放在 naco ，apollo 等地方
@Bean
public SentinelDataSourceHandler sentinelDataSourceHandler(
    DefaultListableBeanFactory beanFactory) {
  return new SentinelDataSourceHandler(beanFactory);
}

// 规则序列化的地方
@ConditionalOnClass(ObjectMapper.class)
protected static class SentinelConverterConfiguration {

  private ObjectMapper objectMapper = new ObjectMapper();

  @Bean("sentinel-json-flow-converter")
  public JsonConverter jsonFlowConverter() {
    return new JsonConverter(objectMapper, FlowRule.class);
  }

  @Bean("sentinel-json-degrade-converter")
  public JsonConverter jsonDegradeConverter() {
    return new JsonConverter(objectMapper, DegradeRule.class);
  }

  @Bean("sentinel-json-system-converter")
  public JsonConverter jsonSystemConverter() {
    return new JsonConverter(objectMapper, SystemRule.class);
  }

  @Bean("sentinel-json-authority-converter")
  public JsonConverter jsonAuthorityConverter() {
    return new JsonConverter(objectMapper, AuthorityRule.class);
  }

  @Bean("sentinel-json-param-flow-converter")
  public JsonConverter jsonParamFlowConverter() {
    return new JsonConverter(objectMapper, ParamFlowRule.class);
  }

}
```

上面的 `InitExecutor.doInit()` 触发的是 `sentinel` 核心的初始化，后面再看 

### SentinelFeignAutoConfiguration  
这里是跟 `feign` 集成，核心类是 `SentinelInvocationHandler`    
`feign`,`ribbon`,`hystrix` 这里几个是 `netflix` 的组件，其中 `sentinel` 替换掉的是 `hystrix`  
详细的后面梳理完 `feign` 后再看


## 小结  

总结一下上面的自动配置  
1. 系统开放的 `rest` 接口，根据 `path` 会过一遍限流（如果区分 `http method` 则是两遍）
2. 系统注册的 `RestTempate` 的时候，如果加上了 `@SentinelRestTemplate` 注解，则会添加拦截器 `SentinelProtectInterceptor`,经过两个限流
3. 使用 `feign`，`@FeignClient` 标记的 `bean`,这里会经过一层限流
4. 使用 `@SentinelResource` 注解标记的方法  

那么问题来了，限流规则是在哪里配置的呢？  


首先看看 `sentinel` 支持哪些规则配置  

### AuthorityRule 
权限配置，就是配置黑白名单  
先看下示例配置  
```json
[
  {
    "resource": "good",
    "limitApp": "abc",
    "strategy": 0
  },
  {
    "resource": "bad",
    "limitApp": "bcd",
    "strategy": 1
  },
  {
    "resource": "terrible",
    "limitApp": "aaa",
    "strategy": 1
  }
]
```
对应的规则类为 `AuthorityRule`,经过的 `slot` 为 `AuthoritySlot`
其中:
1. resource  为资源名，每个规则都需要配置
2. limitApp  限制的 app，其实就是针对 `origin` 来限制的，可以用来限制接入系统，需要提供一个 `RequestOriginParser` 来获取 `origin`,如果没有，那么规则配置是无效的
3. strategy  不同规则配置，含义不一样。这里 0：白名单 1：黑名单



### SystemRule
配置示例  
```json
[
  {
    "highestSystemLoad": -1,
    "highestCpuUsage": -1,
    "qps": 100,
    "avgRt": -1,
    "maxThread": 10
  }
]
```

这里虽然是一个数组，但是配置多个规则其实没啥用，总是使用最小有效配置来判断的  
经过的 `slot` 为 `SystemSlot`,根据系统的负载来限流的，只针对 `EntryType.IN` 类型才有效，默认的都是 `OUT`,在 `CommonFilter` 里面为 `IN`    
配置含义其实看名字就知道了： 
1. highestSystemLoad  系统负载
2. highestCpuUsage   cpu 使用率
3. qps 每秒请求数，这里是有效的，被拦截的请求不算在里面
4. avgRt 平均响应时间
5. maxThread 当前出现线程数

这里开看成应用总的负载限流  

### FlowRule  
配置示例：
```json
[
  {
    "resource": "/hello",
    "controlBehavior": 0,
    "count": 1,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0
  },
  {
    "resource": "/test",
    "controlBehavior": 0,
    "count": 0,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0
  },
  {
    "resource": "GET:http://www.taobao.com",
    "controlBehavior": 0,
    "count": 0,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0
  }
]
```

这个规则是针对每个资源，独立的限流配置  
1. resource  资源名，可以看上面，默认的资源名生产规则
2. grade   配置类型，0：线程数量 1：QPS
3. count   配置数量
4. strategy  0：用于直接流量控制 默认，1：相关流量控制，2：链流量控制  配置了 1 或者 2，需要配置 refResource,如果配 1，那么直接取对应资源的 ClusterNode，2的话，需要 contextName == refResource
5. limitApp  这个必须配，默认 default 如果不配，规则不生效，如果不为 default 或者 other 那么，限流配置单独适应 origin 的限流，例如 配置 app1，那么 app1 的请求信息单独记录，也只根据这个记录来限流控制，不需要管资源的整体调用情况
6. refResource  与 strategy 共同起作用
7. controlBehavior  限流器的类型 0. default(reject directly), 1. warm up, 2. rate limiter, 3. warm up + rate limiter

还有一些其他的，在需要特殊用法的时候再看

### DegradeRule 
降级规则  
```json
[
  {
    "resource": "abc0",
    "count": 20.0,
    "grade": 0,
    "passCount": 0,
    "timeWindow": 10
  },
  {
    "resource": "abc1",
    "count": 15.0,
    "grade": 0,
    "passCount": 0,
    "timeWindow": 10
  }
]
```
这个规则是用来断路的，当超过配置的规则时间，直接断路 `timeWindow` 时间后再开启
规则配置：
1. resource  资源名
2. grade     0: 平均响应时间, 1: 异常比例
3. count     配置的阈值
4. timeWindow  时间窗口



除了上面的规则外，还可以有自定义的规则。。    

刚刚上面的属于本地文件配置，在应用中如下配置  
```properties
spring.cloud.sentinel.datasource.ds1.file.file=classpath: flowrule.json
spring.cloud.sentinel.datasource.ds1.file.data-type=json
spring.cloud.sentinel.datasource.ds1.file.rule-type=flow

spring.cloud.sentinel.datasource.ds2.file.file=classpath: degraderule.json
spring.cloud.sentinel.datasource.ds2.file.data-type=json
spring.cloud.sentinel.datasource.ds2.file.rule-type=degrade

spring.cloud.sentinel.datasource.ds3.file.file=classpath: authority.json
spring.cloud.sentinel.datasource.ds3.file.rule-type=authority

spring.cloud.sentinel.datasource.ds4.file.file=classpath: system.json
spring.cloud.sentinel.datasource.ds4.file.rule-type=system

spring.cloud.sentinel.datasource.ds5.file.file=classpath: param-flow.json
spring.cloud.sentinel.datasource.ds5.file.rule-type=param_flow
```


还支持 `apollo`, `nacos`, `redis`, `zk` 等  

