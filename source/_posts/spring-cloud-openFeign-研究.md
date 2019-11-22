---
title: spring-cloud-openFeign 研究
date: 2019-10-14 20:12:41
tags: [spring-cloud, feign]
categories: 源码阅读
---

这是一篇简单的，`openFeign` 的源码分析  
会涉及到 `ribbon` `Hystrix` 
不过其中 openFeign 的内容比较简单，其实核心难度在 `Hystrix`,这个以后有时间看  
先上一张图。画的很挫。也算是分析思路吧。。  

{% asset_img feign分析.svg This is an example image %}  

主要是从代理构建和方法调用两条线进行分析，下面基本都是代码了  

spring-boot 版本 2.1.5.RELEASE
spring-cloud 版本 Greenwich.SR1
官方示例很简单
配置

```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

核心就是 @EnableFeignClients 注解，这也是配置入口，我们等会从这里开始看
使用

```java
@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}

```


核心是 @FeignClient 注解
方法里面就是 spring-mvc 里面的注解了

首先来看配置

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {

   /**
    * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation
    * declarations e.g.: {@code @ComponentScan("org.my.pkg")} instead of
    * {@code @ComponentScan(basePackages="org.my.pkg")}.
    * @return the array of 'basePackages'.
    */
   String[] value() default {};

   /**
    * Base packages to scan for annotated components.
    * <p>
    * {@link #value()} is an alias for (and mutually exclusive with) this attribute.
    * <p>
    * Use {@link #basePackageClasses()} for a type-safe alternative to String-based
    * package names.
    * @return the array of 'basePackages'.
    */
   String[] basePackages() default {};

   /**
    * Type-safe alternative to {@link #basePackages()} for specifying the packages to
    * scan for annotated components. The package of each class specified will be scanned.
    * <p>
    * Consider creating a special no-op marker class or interface in each package that
    * serves no purpose other than being referenced by this attribute.
    * @return the array of 'basePackageClasses'.
    */
   Class<?>[] basePackageClasses() default {};

   /**
    * A custom <code>@Configuration</code> for all feign clients. Can contain override
    * <code>@Bean</code> definition for the pieces that make up the client, for instance
    * {@link feign.codec.Decoder}, {@link feign.codec.Encoder}, {@link feign.Contract}.
    *
    * @see FeignClientsConfiguration for the defaults
    * @return list of default configurations
    */
   Class<?>[] defaultConfiguration() default {};

   /**
    * List of classes annotated with @FeignClient. If not empty, disables classpath
    * scanning.
    * @return list of FeignClient classes
    */
   Class<?>[] clients() default {};

}
```

里面最核心的一行就是 @Import(FeignClientsRegistrar.class)
注解里面的属性，都是特殊的配置，具体的含义，我们后面继续看
接下面看看 FeignClientsRegistrar 的处理，这是跟 spring 集成的关键，所有基于 spring-boot 的，只要有类似这样的注解，都是这样看的，如果没有注解，那就看 META-INF/spring.factory 文件里面的自动配置
核心方法

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
   registerDefaultConfiguration(metadata, registry);
   registerFeignClients(metadata, registry);
}
```

主要做两件事，注册对应的 配置类 为 bean 注册 client 为 bean
```java
private void registerDefaultConfiguration(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
   Map<String, Object> defaultAttrs = metadata
         .getAnnotationAttributes(EnableFeignClients.class.getName(), true);

   if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
      String name;
      if (metadata.hasEnclosingClass()) {
         name = "default." + metadata.getEnclosingClassName();
      }
      else {
         name = "default." + metadata.getClassName();
      }
      registerClientConfiguration(registry, name,
            defaultAttrs.get("defaultConfiguration"));
   }
}
```

这里就是注册全局默认配置，可以看到，从 EnableFeignClients 获取了 defaultConfiguration 属性，默认是空的
如果有，就可以注册全局默认配置，至于这个配置有什么用，可以配置什么，后面看看

```java
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
      Object configuration) {
   BeanDefinitionBuilder builder = BeanDefinitionBuilder
         .genericBeanDefinition(FeignClientSpecification.class);
   builder.addConstructorArgValue(name);
   builder.addConstructorArgValue(configuration);
   registry.registerBeanDefinition(
         name + "." + FeignClientSpecification.class.getSimpleName(),
         builder.getBeanDefinition());
}
```

这里很明确，就是注册 配置 bean ，目前默认配置，和每个 client 的单独配置都是通过这个方法注册的
可以看到具体的 bean 是 FeignClientSpecification 将 name 和自定义配置 当做构造参数传进去了
后面详细看看 FeignClientSpecification 
继续看注册 client

```java
public void registerFeignClients(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
      // 这是一个基础扫描类，过滤条件是 独立类（非内部类），非注解
   ClassPathScanningCandidateComponentProvider scanner = getScanner();
   scanner.setResourceLoader(this.resourceLoader);

   Set<String> basePackages;

   Map<String, Object> attrs = metadata
         .getAnnotationAttributes(EnableFeignClients.class.getName());
   // 这里看到，处理 client 的注解 @FeignClient
   AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
         FeignClient.class);
   final Class<?>[] clients = attrs == null ? null
         : (Class<?>[]) attrs.get("clients");
   // 这里是对 @EnableFeignClients 注解的 clients 属性支持，可以直接指定有 @FeignClient 注释的接口
   if (clients == null || clients.length == 0) {
      // 默认是空的，也就是走正常的类扫描
      scanner.addIncludeFilter(annotationTypeFilter);
      // 获取要扫描的包，从注解 @EnableFeignClients 读取 value  basePackages  basePackageClasses 所在的包
      // 如果上面没获取到（默认都是空的），就扫描 @EnableFeignClients 注解标记的类所在的包
      basePackages = getBasePackages(metadata);
   }
   else {
      // 如果自己直接指定了 client ，那么只取指定的
      final Set<String> clientClasses = new HashSet<>();
      basePackages = new HashSet<>();
      for (Class<?> clazz : clients) {
         basePackages.add(ClassUtils.getPackageName(clazz));
         clientClasses.add(clazz.getCanonicalName());
      }
      // 也需要走过滤
      AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
         @Override
         protected boolean match(ClassMetadata metadata) {
            String cleaned = metadata.getClassName().replaceAll("\\$", ".");
            return clientClasses.contains(cleaned);
         }
      };
      scanner.addIncludeFilter(
            new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
   }
   // 扫描，是 spirng 的工具类，细节不看
   for (String basePackage : basePackages) {
      Set<BeanDefinition> candidateComponents = scanner
            .findCandidateComponents(basePackage);
      for (BeanDefinition candidateComponent : candidateComponents) {
         if (candidateComponent instanceof AnnotatedBeanDefinition) {
            // verify annotated class is an interface
            AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
            // client 只能是接口
            Assert.isTrue(annotationMetadata.isInterface(),
                  "@FeignClient can only be specified on an interface");
            // 获取 @FeignClient 注解的熟悉
            Map<String, Object> attributes = annotationMetadata
                  .getAnnotationAttributes(
                        FeignClient.class.getCanonicalName());
            // 获取 name，就是 @FeignClient 里面的熟悉  contextId  value  name  serviceId  按照顺序获取
            String name = getClientName(attributes);
            // 注册 client 对应的 config ，可以 @FeignClient 配置自定义的 configuration ，默认是空
            registerClientConfiguration(registry, name,
                  attributes.get("configuration"));
            // 注册 client bean
            registerFeignClient(registry, annotationMetadata, attributes);
         }
      }
   }
}
```

上面都加了注释了，看看注册 client 的 bean

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
      AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
   String className = annotationMetadata.getClassName();
   // 注意这里的 factoryBean  FeignClientFactoryBean
   BeanDefinitionBuilder definition = BeanDefinitionBuilder
         .genericBeanDefinition(FeignClientFactoryBean.class);
   validate(attributes);
   // url 默认是空，这里应该是直连时候指定url
   definition.addPropertyValue("url", getUrl(attributes));
   // path 是 basePath
   definition.addPropertyValue("path", getPath(attributes));
   // 这里 name 的获取顺序是 serviceId  name value   支持 placeholder 也就是 ${}
   String name = getName(attributes);
   definition.addPropertyValue("name", name);
   // 取 contextId ，如果为空，那就用 getName 获取，也就是上面的 name 默认是一样的
   String contextId = getContextId(attributes);
   definition.addPropertyValue("contextId", contextId);
   // 
   definition.addPropertyValue("type", className);
   // 
   definition.addPropertyValue("decode404", attributes.get("decode404"));
   // 容错类
   definition.addPropertyValue("fallback", attributes.get("fallback"));
   // 容错工程类
   definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
   definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
   // 可以看到，别名，默认就是 自己设置的 name + "FeignClient"
   String alias = contextId + "FeignClient";
   AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

   boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
                                             // null

   beanDefinition.setPrimary(primary);
   // 相当于自己设置别名
   String qualifier = getQualifier(attributes);
   if (StringUtils.hasText(qualifier)) {
      alias = qualifier;
   }

   BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
         new String[] { alias });
   BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

先看注册的 配置类 FeignClientSpecification

```java
class FeignClientSpecification implements NamedContextFactory.Specification {

   private String name;

   private Class<?>[] configuration;

   FeignClientSpecification() {
   }

   FeignClientSpecification(String name, Class<?>[] configuration) {
      this.name = name;
      this.configuration = configuration;
   }

   public String getName() {
      return this.name;
   }

   public void setName(String name) {
      this.name = name;
   }

   public Class<?>[] getConfiguration() {
      return this.configuration;
   }

   public void setConfiguration(Class<?>[] configuration) {
      this.configuration = configuration;
   }

   @Override
   public boolean equals(Object o) {
      if (this == o) {
         return true;
      }
      if (o == null || getClass() != o.getClass()) {
         return false;
      }
      FeignClientSpecification that = (FeignClientSpecification) o;
      return Objects.equals(this.name, that.name)
            && Arrays.equals(this.configuration, that.configuration);
   }

   @Override
   public int hashCode() {
      return Objects.hash(this.name, this.configuration);
   }

   @Override
   public String toString() {
      return new StringBuilder("FeignClientSpecification{").append("name='")
            .append(this.name).append("', ").append("configuration=")
            .append(Arrays.toString(this.configuration)).append("}").toString();
   }

}
```

没啥看的，
再看看 client 的 FeignClientFactoryBean

```java
@Override
public Object getObject() throws Exception {
   return getTarget();
}

/**
* @param <T> the target type of the Feign client
* @return a {@link Feign} client created with the specified data and the context
* information
*/
<T> T getTarget() {
   // FeignContext 是在 autoconfiguration 的时候注册的
   FeignContext context = this.applicationContext.getBean(FeignContext.class);
   // 配置分为三层，默认 - 全局 - 单个client 每个配置类实际上是一个 @Configuration 标记的 spring 配置类
   // context 内部为每个 client 都构建了一个 ApplicationContext parent 指向默认的 context
   // 里面配置类的解析顺序为 单个 - 全局 - 默认 构建内部的 bean
   // 这个方法里面，构建 Feign.Builder 设置 logger encoder decoder contract
   // 然后根据 FeignClientProperties 配置，设置 loggerLevel retryer errorDecoder request.options requestInterceptor decode404 配置
   // 如果自定义配置或者全局配置里面有 logger encoder decoder contract 这里也重新设置
   Feign.Builder builder = feign(context);
   // 默认 url 是没有的
   if (!StringUtils.hasText(this.url)) {
      if (!this.name.startsWith("http")) {
         this.url = "http://" + this.name;
      }
      else {
         this.url = this.name;
      }
      this.url += cleanPath();
      // so 大部分应该走这里
      return (T) loadBalance(builder, context,
            new HardCodedTarget<>(this.type, this.name, this.url));
   }
   if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
      this.url = "http://" + this.url;
   }
   String url = this.url + cleanPath();
   Client client = getOptional(context, Client.class);
   if (client != null) {
      // url 存在，代表直连，lodbalance 就不需要了
      if (client instanceof LoadBalancerFeignClient) {
         // not load balancing because we have a url,
         // but ribbon is on the classpath, so unwrap
         client = ((LoadBalancerFeignClient) client).getDelegate();
      }
      // 获取默认的 client
      builder.client(client);
   }
   // 构建
   Targeter targeter = get(context, Targeter.class);
   return (T) targeter.target(this, builder, context,
         new HardCodedTarget<>(this.type, this.name, url));
}
```

好吧，大概看了下里面的实现，这里其实是生成一个代理的过程，但是又与标准的动态代理不一样
构建过程在 builder 里面，之前的 logger encoder deocder contract 等是按照 client 实例配置  default 全局配置  默认配置 的 顺序启用的
client 的实例配置，就是在客户端接口注解 @FeignClient 配置的，default 全局配置，则是 @EnableFeignClients 注解里面配置的
至于默认配置，是基于 spring-boot 的自动配置 ，这里面也分情况，根据是否存在 ribbon ，有不同的注入配置
等会，突然发现上面三层配置，创建 context 的机制是 spring-cloud 里面的东西
先来看下基于 ribbon的自动配置，这里默认情况的

```java
// 这里 ILoadBalancer 是 ribbon 包里面的  Feign 就是 feign 了
@ConditionalOnClass({ ILoadBalancer.class, Feign.class })
@Configuration
// 这里，要在 feign 默认配置之前先处理
@AutoConfigureBefore(FeignAutoConfiguration.class)
// 开启自定义参数配置
@EnableConfigurationProperties({ FeignHttpClientProperties.class })
// Order is important here, last should be the default, first should be optional
// see
// https://github.com/spring-cloud/spring-cloud-netflix/issues/2086#issuecomment-316281653
// 上面有注释，顺序很重要，就是 FeignLoadBalanced 的自动配置
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
      OkHttpFeignLoadBalancedConfiguration.class,
      DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration {

   @Bean
   @Primary
   @ConditionalOnMissingBean
   @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
   public CachingSpringLoadBalancerFactory cachingLBClientFactory(
         SpringClientFactory factory) {
      return new CachingSpringLoadBalancerFactory(factory);
   }

   @Bean
   @Primary
   @ConditionalOnMissingBean
   @ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
   public CachingSpringLoadBalancerFactory retryabeCachingLBClientFactory(
         SpringClientFactory factory, LoadBalancedRetryFactory retryFactory) {
      return new CachingSpringLoadBalancerFactory(factory, retryFactory);
   }

   @Bean
   @ConditionalOnMissingBean
   public Request.Options feignRequestOptions() {
      return LoadBalancerFeignClient.DEFAULT_OPTIONS;
   }

}
```

其实所有的配置类，最后都会装配到 Feign.Builder 里面
在 builder 里面构造代理，直接进去看吧。
最终，构建代理的是接口 

```java
interface Targeter {

   <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
         FeignContext context, Target.HardCodedTarget<T> target);

}
```

使用 feign 的默认实现是 
```java
class DefaultTargeter implements Targeter {

   @Override
   public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
         FeignContext context, Target.HardCodedTarget<T> target) {
      return feign.target(target);
   }

}
```

可以看到，直接调用了 builder 的 target 方法
还有另一个 实现，这个实现才是自动配置的

```java
class HystrixTargeter implements Targeter {

   @Override
   public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
         FeignContext context, Target.HardCodedTarget<T> target) {
      if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
         return feign.target(target);
      }
      feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
      SetterFactory setterFactory = getOptional(factory.getName(), context,
            SetterFactory.class);
      if (setterFactory != null) {
         builder.setterFactory(setterFactory);
      }
      Class<?> fallback = factory.getFallback();
      if (fallback != void.class) {
         return targetWithFallback(factory.getName(), context, target, builder,
               fallback);
      }
      Class<?> fallbackFactory = factory.getFallbackFactory();
      if (fallbackFactory != void.class) {
         return targetWithFallbackFactory(factory.getName(), context, target, builder,
               fallbackFactory);
      }

      return feign.target(target);
   }

   private <T> T targetWithFallbackFactory(String feignClientName, FeignContext context,
         Target.HardCodedTarget<T> target, HystrixFeign.Builder builder,
         Class<?> fallbackFactoryClass) {
      FallbackFactory<? extends T> fallbackFactory = (FallbackFactory<? extends T>) getFromContext(
            "fallbackFactory", feignClientName, context, fallbackFactoryClass,
            FallbackFactory.class);
      return builder.target(target, fallbackFactory);
   }

   private <T> T targetWithFallback(String feignClientName, FeignContext context,
         Target.HardCodedTarget<T> target, HystrixFeign.Builder builder,
         Class<?> fallback) {
      T fallbackInstance = getFromContext("fallback", feignClientName, context,
            fallback, target.type());
      return builder.target(target, fallbackInstance);
   }

   private <T> T getFromContext(String fallbackMechanism, String feignClientName,
         FeignContext context, Class<?> beanType, Class<T> targetType) {
      Object fallbackInstance = context.getInstance(feignClientName, beanType);
      if (fallbackInstance == null) {
         throw new IllegalStateException(String.format(
               "No " + fallbackMechanism
                     + " instance of type %s found for feign client %s",
               beanType, feignClientName));
      }

      if (!targetType.isAssignableFrom(beanType)) {
         throw new IllegalStateException(String.format("Incompatible "
               + fallbackMechanism
               + " instance. Fallback/fallbackFactory of type %s is not assignable to %s for feign client %s",
               beanType, targetType, feignClientName));
      }
      return (T) fallbackInstance;
   }

   private <T> T getOptional(String feignClientName, FeignContext context,
         Class<T> beanType) {
      return context.getInstance(feignClientName, beanType);
   }

}
```

可以看到，主要处理了 SetterFactory fallback fallbackFactory
将这几个配置设置在builder 里面后，调用 builder 的 target
默认先看默认的 builder 实现，  Hystrix 有熔断配置的实现是对默认 builder 做了一个扩展

```java
public <T> T target(Target<T> target) {
  return build().newInstance(target);
}
public Feign build() {
  // 这里，构建这些对象，可以认为将配置分组到不同类
  SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
      new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
          logLevel, decode404, closeAfterDecode, propagationPolicy);
  ParseHandlersByName handlersByName =
      new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
          errorDecoder, synchronousMethodHandlerFactory);
  return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}
```

build 方法就是把配置分组了，构建了一个 ReflectiveFeign 
核心是里面的 newInstance 方法

```java
public <T> T newInstance(Target<T> target) {
   // 这里，解析 type 接口的方法
  Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
  Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
  List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
  // 下面是处理 默认方法和过滤掉 object 的方法
  for (Method method : target.type().getMethods()) {
    if (method.getDeclaringClass() == Object.class) {
      continue;
    } else if (Util.isDefault(method)) {
      DefaultMethodHandler handler = new DefaultMethodHandler(method);
      defaultMethodHandlers.add(handler);
      methodToHandler.put(method, handler);
    } else {
      methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
    }
  }
  // jdk 动态代理的 InvocationHandler 实现，默认是  new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
  InvocationHandler handler = factory.create(target, methodToHandler);
  T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
      new Class<?>[] {target.type()}, handler);
  // 默认方法绑定到代理对象
  for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
    defaultMethodHandler.bindTo(proxy);
  }
  // so 返回的是一个 JDK 的动态代理对象
  return proxy;
}
```

so 继续看看 InvocationHandler handler = factory.create(target, methodToHandler); 创建的东西

```java
static class FeignInvocationHandler implements InvocationHandler {

  private final Target target;
  private final Map<Method, MethodHandler> dispatch;

  FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
    this.target = checkNotNull(target, "target");
    this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }

    return dispatch.get(method).invoke(args);
  }

```

非常，简单，其实核心还是 apply 方法解析出来的 methodHandler 

```java
public Map<String, MethodHandler> apply(Target key) {
    // 这里使用 Contract 解析相关接口
    List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
    Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
    for (MethodMetadata md : metadata) {
      // 这个是根据参数构建 request 请求的解析类
      BuildTemplateByResolvingArgs buildTemplate;
      if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
        buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
      } else if (md.bodyIndex() != null) {
        buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
      } else {
        buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
      }
      // 这里的 configKey 是 Feign.configKey 方法生成的 类 simpleName + methodName + 参数
      // factory 构建的是 SynchronousMethodHandler 实例 ，可以想象到，具体的调用逻辑就是在这里完成的
      result.put(md.configKey(),
          factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
    }
    return result;
  }
}
```

看看具体调用的地方

```java
public Object invoke(Object[] argv) throws Throwable {
  // 第一步 就是用 请求参数构建 RequestTemplate 具体有三种，后面详细看看看 BuildFormEncodedTemplateFromArgs  BuildEncodedTemplateFromArgs  BuildTemplateByResolvingArgs
  RequestTemplate template = buildTemplateFromArgs.create(argv);
  // 重试 clone 就是将计数重置为 1
  Retryer retryer = this.retryer.clone();
  while (true) {
    try {
      // 正常调用，如果 ok 就返回了
      // 这里是调用的核心逻辑
      return executeAndDecode(template);
    } catch (RetryableException e) {
      // 如果抛出的是 可重试异常
      try {
        // 重试策略判断一下，如果不允许，就把异常抛出来，如果允许，内部可以记录一下，或者 等待一会再尝试
        retryer.continueOrPropagate(e);
      } catch (RetryableException th) {
        // 异常是否需要解开包装
        Throwable cause = th.getCause();
        if (propagationPolicy == UNWRAP && cause != null) {
          throw cause;
        } else {
          throw th;
        }
      }
      if (logLevel != Logger.Level.NONE) {
        logger.logRetry(metadata.configKey(), logLevel);
      }
      continue;
    }
  }
}
```

接下来，就可以详细看看具体的调用逻辑了

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
  // 这里将请求封装为 request 内部首先经过 RequestInterceptor 处理  然后构建 request
  Request request = targetRequest(template);
  // 请求日志
  if (logLevel != Logger.Level.NONE) {
    logger.logRequest(metadata.configKey(), logLevel, request);
  }

  Response response;
  // 记录请求时间
  long start = System.nanoTime();
  try {
    // 使用 Client 执行请求，带上 options 参数，options 也是可配置的，通常是连接超时时间之类的
    // 这里就是核心了，请求控制丢给 client 去了
    response = client.execute(request, options);
  } catch (IOException e) {
    // 异常日志
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
    }
    // 抛异常，这里是将异常包装为 可重试异常
    throw errorExecuting(request, e);
  }
  // 正常结果
  long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

  boolean shouldClose = true;
  try {
    // 响应日志
    if (logLevel != Logger.Level.NONE) {
      response =
          logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
    }
    // return 类型就是 Response
    if (Response.class == metadata.returnType()) {
      if (response.body() == null) {
        return response;
      }
      if (response.body().length() == null ||
          response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
        shouldClose = false;
        return response;
      }
      // Ensure the response body is disconnected
      byte[] bodyData = Util.toByteArray(response.body().asInputStream());
      return response.toBuilder().body(bodyData).build();
    }
    // 正常响应
    if (response.status() >= 200 && response.status() < 300) {
      if (void.class == metadata.returnType()) {
        return null;
      } else {
        Object result = decode(response);
        shouldClose = closeAfterDecode;
        return result;
      }
    } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
      // 404 并且有需要返回结果
      Object result = decode(response);
      shouldClose = closeAfterDecode;
      return result;
    } else {
      // 其他的就根据 errorDecoder 解析抛异常
      throw errorDecoder.decode(metadata.configKey(), response);
    }
  } catch (IOException e) {
    // io 异常
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
    }
    // 封装为 FeignException
    throw errorReading(request, response, e);
  } finally {
    // 
    if (shouldClose) {
      ensureClosed(response.body());
    }
  }
}

```

那么接下来看看 client

```java
public interface Client {
   /**
* Executes a request against its {@link Request#url() url} and returns a response.
*
* @param request safe to replay.
* @param options options to apply to this request.
* @return connected response, {@link Response.Body} is absent or unread.
* @throws IOException on a network error connecting to {@link Request#url()}.
*/
Response execute(Request request, Options options) throws IOException;
}

```

只有一个接口要实现，有一个默认的内部类实现，就是使用 HttpURLConnection 一看就没有软负载， feign 的负载功能是通过 fibbon 实现了，有一个实现 LoadBalancerFeignClient

```java
@Override
public Response execute(Request request, Request.Options options) throws IOException {
   try {
      URI asUri = URI.create(request.url());
      String clientName = asUri.getHost();
      URI uriWithoutHost = cleanUrl(request.url(), clientName);
      // 转为 ribbon 的 request
      FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
            this.delegate, request, uriWithoutHost);
      // 配置类
      IClientConfig requestConfig = getClientConfig(options, clientName);
      // 获取 FeignLoadBalancer 然后执行
      return lbClient(clientName)
            .executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
   }
   catch (ClientException e) {
      IOException io = findIOException(e);
      if (io != null) {
         throw io;
      }
      throw new RuntimeException(e);
   }
}
private FeignLoadBalancer lbClient(String clientName) {
   return this.lbClientFactory.create(clientName);
}
public FeignLoadBalancer create(String clientName) {
   FeignLoadBalancer client = this.cache.get(clientName);
   if (client != null) {
      return client;
   }
   IClientConfig config = this.factory.getClientConfig(clientName);
   ILoadBalancer lb = this.factory.getLoadBalancer(clientName);
   ServerIntrospector serverIntrospector = this.factory.getInstance(clientName,
         ServerIntrospector.class);
   client = this.loadBalancedRetryFactory != null
         ? new RetryableFeignLoadBalancer(lb, config, serverIntrospector,
               this.loadBalancedRetryFactory)
         : new FeignLoadBalancer(lb, config, serverIntrospector);
   // 默认是走 FeignLoadBalancer  可以看到传了一个 ILoadBalancer  ServerIntrospector 和 IClientConfig基本配置
   this.cache.put(clientName, client);
   return client;
}

```

然后又到了 FeignLoadBalancer 的 executeWithLoadBalancer 方法

```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    // 获取 负载指令，其实就是负载策略
    LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);
    try {
        // ribbon 内部使用 rxjava 
        // submit 内部控制 负载均衡和重试次数， 包括同 server 重试 和 不同 server 重试
        // 这里的 ServerOperation 就是一次实际的请求，参数是已经选择好的 server 然后调用 execute，本质就是 feign 的 client 请求
        return command.submit(
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    S requestForServer = (S) request.replaceUri(finalUri);
                    try {
                        return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                    }
                    catch (Exception e) {
                        return Observable.error(e);
                    }
                }
            })
            .toBlocking()
            .single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
    
}
protected LoadBalancerCommand<T> buildLoadBalancerCommand(final S request, final IClientConfig config) {
   // 重试策略
   RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, config);
   LoadBalancerCommand.Builder<T> builder = LoadBalancerCommand.<T>builder()
         .withLoadBalancerContext(this)
         .withRetryHandler(handler)
         .withLoadBalancerURI(request.getUri());
   // 其他自定义配置
   customizeLoadBalancerCommandBuilder(request, config, builder);
   return builder.build();
}
public RequestSpecificRetryHandler getRequestSpecificRetryHandler(
      RibbonRequest request, IClientConfig requestConfig) {
   if (this.ribbon.isOkToRetryOnAllOperations()) {
      return new RequestSpecificRetryHandler(true, true, this.getRetryHandler(),
            requestConfig);
   }
   if (!request.toRequest().httpMethod().name().equals("GET")) {
      return new RequestSpecificRetryHandler(true, false, this.getRetryHandler(),
            requestConfig);
   }
   else {
      return new RequestSpecificRetryHandler(true, true, this.getRetryHandler(),
            requestConfig);
   }
}

```

然后最复杂的方法来了

```java
public Observable<T> submit(final ServerOperation<T> operation) {
    final ExecutionInfoContext context = new ExecutionInfoContext();
    
    if (listenerInvoker != null) {
        try {
            listenerInvoker.onExecutionStart();
        } catch (AbortExecutionException e) {
            return Observable.error(e);
        }
    }

    final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer();
    final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer();

    // Use the load balancer
    Observable<T> o =
            // server 不为null ，那就代表是直连了，否则使用 selectServer 获取可用列表
            (server == null ? selectServer() : Observable.just(server))
            // rxjava 的api 做相当于做一次转换，T 是响应类型
            .concatMap(new Func1<Server, Observable<T>>() {
                @Override
                // Called for each server being selected
                public Observable<T> call(Server server) {
                    context.setServer(server);
                    final ServerStats stats = loadBalancerContext.getServerStats(server);
                    
                    // Called for each attempt and retry
                    Observable<T> o = Observable
                            .just(server)
                            .concatMap(new Func1<Server, Observable<T>>() {
                                @Override
                                public Observable<T> call(final Server server) {
                                    context.incAttemptCount();
                                    loadBalancerContext.noteOpenConnection(stats);
                                    
                                    if (listenerInvoker != null) {
                                        try {
                                            listenerInvoker.onStartWithServer(context.toExecutionInfo());
                                        } catch (AbortExecutionException e) {
                                            return Observable.error(e);
                                        }
                                    }
                                    
                                    final Stopwatch tracer = loadBalancerContext.getExecuteTracer().start();
                                    // 注意这里，就是调用传 实际的请求，上面一堆都是记录和重试
                                    return operation.call(server).doOnEach(new Observer<T>() {
                                        private T entity;
                                        @Override
                                        public void onCompleted() {
                                            recordStats(tracer, stats, entity, null);
                                            // TODO: What to do if onNext or onError are never called?
                                        }

                                        @Override
                                        public void onError(Throwable e) {
                                            recordStats(tracer, stats, null, e);
                                            logger.debug("Got error {} when executed on server {}", e, server);
                                            if (listenerInvoker != null) {
                                                listenerInvoker.onExceptionWithServer(e, context.toExecutionInfo());
                                            }
                                        }

                                        @Override
                                        public void onNext(T entity) {
                                            this.entity = entity;
                                            if (listenerInvoker != null) {
                                                listenerInvoker.onExecutionSuccess(entity, context.toExecutionInfo());
                                            }
                                        }                            
                                        
                                        private void recordStats(Stopwatch tracer, ServerStats stats, Object entity, Throwable exception) {
                                            tracer.stop();
                                            loadBalancerContext.noteRequestCompletion(stats, entity, exception, tracer.getDuration(TimeUnit.MILLISECONDS), retryHandler);
                                        }
                                    });
                                }
                            });
                    
                    // retry 也是 rxjava api ，retryPolicy 方法判断是否可以重试
                    if (maxRetrysSame > 0)
                        o = o.retry(retryPolicy(maxRetrysSame, true));
                    return o;
                }
            });
    // 上面重试是针对同一个 server 的，这里是针对不同的 server 的   
    if (maxRetrysNext > 0 && server == null)
        o = o.retry(retryPolicy(maxRetrysNext, false));
    
    return o.onErrorResumeNext(new Func1<Throwable, Observable<T>>() {
        @Override
        public Observable<T> call(Throwable e) {
            if (context.getAttemptCount() > 0) {
                if (maxRetrysNext > 0 && context.getServerAttemptCount() == (maxRetrysNext + 1)) {
                    e = new ClientException(ClientException.ErrorType.NUMBEROF_RETRIES_NEXTSERVER_EXCEEDED,
                            "Number of retries on next server exceeded max " + maxRetrysNext
                            + " retries, while making a call for: " + context.getServer(), e);
                }
                else if (maxRetrysSame > 0 && context.getAttemptCount() == (maxRetrysSame + 1)) {
                    e = new ClientException(ClientException.ErrorType.NUMBEROF_RETRIES_EXEEDED,
                            "Number of retries exceeded max " + maxRetrysSame
                            + " retries, while making a call for: " + context.getServer(), e);
                }
            }
            if (listenerInvoker != null) {
                listenerInvoker.onExecutionFailed(e, context.toFinalExecutionInfo());
            }
            return Observable.error(e);
        }
    });
}

```


