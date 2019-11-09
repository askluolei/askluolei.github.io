---
title: spring 源码再分析
date: 2019-07-27 12:32:26
tags: [java,spring,源码]
categories: 源码阅读
---

{% asset_img spring初始化-单例实例化过程.svg This is an example image %}


ConfigurationClassPostProcessor        BeanDefinitionRegistryPostProcessor
AutowiredAnnotationBeanPostProcessor            InstantiationAwareBeanPostProcessorAdapter
CommonAnnotationBeanPostProcessor             InstantiationAwareBeanPostProcessor

EventListenerMethodProcessor        SmartInitializingSingleton
DefaultEventListenerFactory                EventListenerFactory
ApplicationContextAwareProcessor        BeanPostProcessor

ApplicationListenerDetector        BeanPostProcessor

BeanPostProcessorChecker

1. prepareRefresh
    记录启动时间
2. prepareBeanFactory
    忽略和设置框架依赖注入
3. postProcessBeanFactory
    扩展方法，
4. invokeBeanFactoryPostProcessors
    调用 BeanFactoryPostProcessor  BeanDefinitionRegistryPostProcessor 扩展
    1. 直接挂在 context 上的 BeanDefinitionRegistryPostProcessor   -> postProcessBeanDefinitionRegistry
    2. beanFactory 里面的  BeanDefinitionRegistryPostProcessor ,PriorityOrdered ->  postProcessBeanDefinitionRegistry
    3. beanFactory 里面的  BeanDefinitionRegistryPostProcessor ,Ordered ->  postProcessBeanDefinitionRegistry
    4. beanFactory 里面的  BeanDefinitionRegistryPostProcessor ->  postProcessBeanDefinitionRegistry
    5. 上面执行过的 继续执行 BeanFactoryPostProcessor —> postProcessBeanFactory
    6. 直接挂在 context 上的 BeanFactoryPostProcessor —> postProcessBeanFactory
    7. beanFactory 里面的 BeanFactoryPostProcessor , PriorityOrdered  ->  postProcessBeanFactory
    8. beanFactory 里面的 BeanFactoryPostProcessor , Ordered ->  postProcessBeanFactory
    9. beanFactory 里面的 BeanFactoryPostProcessor  ->  postProcessBeanFactory
这里核心的是 ConfigurationClassPostProcessor

5. registerBeanPostProcessors
    添加 BeanPostProcessor
    beanFactory 自身带的
    添加 BeanPostProcessorChecker
    容器里面的
    BeanPostProcessor - PriorityOrdered
    BeanPostProcessor - Ordered
    BeanPostProcessor -
    上面三个，按照顺序添加到自身带的后面
    BeanPostProcessor - MergedBeanDefinitionPostProcessor - PriorityOrdered
    BeanPostProcessor - MergedBeanDefinitionPostProcessor - Ordered
    BeanPostProcessor - MergedBeanDefinitionPostProcessor
    再添加一次 MergedBeanDefinitionPostProcessor 类型的，按照上面的顺序
    最后添加 ApplicationListenerDetector
6. initMessageSource
    这个是国际化的地方
    DelegatingMessageSource
7. initApplicationEventMulticaster
    事件
    SimpleApplicationEventMulticaster
8.    onRefresh
    空的，扩展接口
9.    registerListeners
    添加 ApplicationListener
    先 beanFactory 上挂的，然后 beanFactory 里面的
    然后将这之前的事件发布了
10.    finishBeanFactoryInitialization
    特殊处理 LoadTimeWeaverAware 先进行初始化 （getBean）
    冻结 beanDefinition ,不再改变
    实例化非延时加载的单例
11. finishRefresh
    结束
    
实例化非延时加载的单例

spring 实例化 bean 其实就是调用 getBean
内部实现是 doGetBean
先看 普通的 bean  后面再看 factoryBean
1. 就是去手动注册的 bean 里面检查是否存在，如果存在，直接返回了  一般只有部分 spring 内部的 bean 是手动注册的
2. 在创建 bean 之前，用 InstantiationAwareBeanPostProcessor -> postProcessBeforeInstantiation 处理  
    这里参数是 class 和 beanName ，如果返回非 null ，那么就使用这里返回的 bean 然后
        BeanPostProcessor -> postProcessAfterInitialization 处理生成的 bean
3. 如果没被上面的扩展接口处理，那么进入标准的创建流程
4. 如果提供了 bean 的工厂类，那么使用工厂类创建 （可以是 Supplier 或者 factoryMethod）
5. 核心创建方法两个 autowireConstructor （有构造依赖的构建） 和 instantiateBean （标准创建）
6. 创建完成后 调用扩展 MergedBeanDefinitionPostProcessor -> postProcessMergedBeanDefinition  这里是修改 BeanDefinetion 的地方
7. populateBean  处理扩展接口 InstantiationAwareBeanPostProcessor -> postProcessAfterInstantiation   返回值决定是否继续处理 propertyValue
    后面继续处理扩展接口  InstantiationAwareBeanPostProcessor -> postProcessProperties  postProcessPropertyValues
    
BeanPostProcessor -> postProcessBeforeInitialization  

afterPropertiesSet  initMethod
BeanPostProcessor -> postProcessAfterInitialization
注册 destoryMethod

几个核心的 BeanPostProcessor
ApplicationContextAwareProcessor
ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
PostProcessorRegistrationDelegate$BeanPostProcessorChecker
CommonAnnotationBeanPostProcessor
AutowiredAnnotationBeanPostProcessor
ApplicationListenerDetector

1. ApplicationContextAwareProcessor  实现  BeanPostProcessor
作用：
在 init 方法之前，调用 bean 的 Aware 接口
EnvironmentAware
EmbeddedValueResolverAware
ResourceLoaderAware
ApplicationEventPublisherAware
MessageSourceAware
ApplicationContextAware
2. ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor 实现 InstantiationAwareBeanPostProcessor
作用:
在依赖注入前，设置 BeanFactory 接口  EnhancedConfiguration
在 init 方法前 调用
ImportAware
3. PostProcessorRegistrationDelegate$BeanPostProcessorChecker
这个扩展是用来检测 bean 是否经过了所有 BeanPostProcess 的处理 ，如果没有，会有 info 日志
因为在实例化 bean 之前，首先处理的是 BeanPostProcessor 实例，按照顺序，是内建 -> PriorityOrdered -> Ordered -> 非排序的    
如果直接存在依赖注入的问题，那么，会有一个问题，在 BeanPostProcessor 里面依赖了其他普通的 bean，那么会先触发普通 bean 的实例化，但是没法享受 BeanPostProcessor 的处理了
要知道 无论是 ioc 还是 aop 都是使用 BeanPostProcessor 扩展的
4. CommonAnnotationBeanPostProcessor 实现 InstantiationAwareBeanPostProcessor MergedBeanDefinitionPostProcessor DestructionAwareBeanPostProcessor
这里处理的是 init 和 destroy 和 JSR-250 注解依赖注入的地方
这里的注解属于标准注解 javax.annotation.PostConstruct javax.annotation.PreDestroy  javax.annotation.Resource 等
5. AutowiredAnnotationBeanPostProcessor 实现 InstantiationAwareBeanPostProcessor  MergedBeanDefinitionPostProcessor
这里是处理内置注解 @Autowired   @Value  javax.inject.Inject  的注入
6. ApplicationListenerDetector
这里是处理 ApplicationListener 的地方，单例的 ApplicationListener 实例注册，非单例，打印警告信息

ConfigurationClassPostProcessor 这个是核心类，还要仔细看看
