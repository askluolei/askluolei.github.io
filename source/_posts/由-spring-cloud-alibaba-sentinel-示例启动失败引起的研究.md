---
title: 由 spring-cloud-alibaba sentinel 示例启动失败引起的研究
date: 2019-11-12 20:47:26
tags:
  - java
  - spring-boot
  - sentinel
  - 笔记
categories:
  - 技术笔记
---

## 问题起因 
在看 `sentinel` 的时候，写了一个示例，添加以下配置  
```yml
# sentinel 测试
spring:
  cloud:
    sentinel:
      datasource:
        # 名称随意
        ds1:
          file:
            file: "classpath: rules/flowRules.json"
            rule-type: flow
```

这是一个本地配置文件来配置 `sentinel` 规则的示例  
然后启动应用，发现异常  

```
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to bind properties under 'spring.cloud.sentinel.datasource.ds1.file.rule-type' to org.springframework.cloud.alibaba.sentinel.datasource.RuleType:

    Property: spring.cloud.sentinel.datasource.ds1.file.rule-type
    Value: flow
    Origin: class path resource [application.yml]:23:24
    Reason: 2

Action:

Update your application's configuration
```

`sentinel` 的配置类是 `SentinelProperties`, `ruleType` 所在类为 `AbstractDataSourceProperties`,类型为枚举 `RuleType`  

上面，没有任何异常信息，只是一个绑定错误，我还以为哪里拼写错了。    
最后实在没办法，直接进 `spring` 处理 `@EnableConfigurationProperties(SentinelProperties.class)` 的地方  

在类 `ConfigurationPropertiesBindingPostProcessor` 的 `bind` 方法 
```java
	private void bind(Object bean, String beanName, ConfigurationProperties annotation) {
		ResolvableType type = getBeanType(bean, beanName);
		Validated validated = getAnnotation(bean, beanName, Validated.class);
		Annotation[] annotations = (validated != null)
				? new Annotation[] { annotation, validated }
				: new Annotation[] { annotation };
		Bindable<?> target = Bindable.of(type).withExistingValue(bean)
				.withAnnotations(annotations);
		try {
			this.configurationPropertiesBinder.bind(target);
		}
		catch (Exception ex) {
			throw new ConfigurationPropertiesBindException(beanName, bean, annotation,
					ex);
		}
	}
```

可以根据条件断点，然后在异常那里打印堆栈日志，发现是一个数组越界。    
```java
Caused by: java.lang.ArrayIndexOutOfBoundsException: 2
	at java.lang.reflect.Parameter.getAnnotatedType(Parameter.java:237)
	at org.hibernate.validator.internal.metadata.provider.AnnotationMetaDataProvider.getParameterMetaData(AnnotationMetaDataProvider.java:427)
	at org.hibernate.validator.internal.metadata.provider.AnnotationMetaDataProvider.findExecutableMetaData(AnnotationMetaDataProvider.java:300)
	at org.hibernate.validator.internal.metadata.provider.AnnotationMetaDataProvider.getMetaData(AnnotationMetaDataProvider.java:285)
	at org.hibernate.validator.internal.metadata.provider.AnnotationMetaDataProvider.getConstructorMetaData(AnnotationMetaDataProvider.java:266)
	at org.hibernate.validator.internal.metadata.provider.AnnotationMetaDataProvider.retrieveBeanConfiguration(AnnotationMetaDataProvider.java:135)
	at org.hibernate.validator.internal.metadata.provider.AnnotationMetaDataProvider.getBeanConfiguration(AnnotationMetaDataProvider.java:124)
	at org.hibernate.validator.internal.metadata.BeanMetaDataManager.getBeanConfigurationForHierarchy(BeanMetaDataManager.java:232)
	at org.hibernate.validator.internal.metadata.BeanMetaDataManager.createBeanMetaData(BeanMetaDataManager.java:199)
	at org.hibernate.validator.internal.metadata.BeanMetaDataManager.getBeanMetaData(BeanMetaDataManager.java:166)
	at org.hibernate.validator.internal.engine.ValidatorImpl.validate(ValidatorImpl.java:157)
	at org.springframework.validation.beanvalidation.SpringValidatorAdapter.validate(SpringValidatorAdapter.java:108)
```

但是。越界发生在内部，我也无法干预，为了具体定位越界到底在哪里出现，跟踪调试  
最后定位到 `org.hibernate.validator.internal.metadata.provider.AnnotationMetaDataProvider`  的方法  
```java
private List<ConstrainedParameter> getParameterMetaData(Executable executable) {
  if ( executable.getParameterCount() == 0 ) {
    return Collections.emptyList();
  }

  // ****  这里返回的构造方法参数是 4 引起了疑惑 ****
  Parameter[] parameters = executable.getParameters();

  List<ConstrainedParameter> metaData = new ArrayList<>( parameters.length );

  int i = 0;
  for ( Parameter parameter : parameters ) {
    Annotation[] parameterAnnotations;
    try {
      parameterAnnotations = parameter.getAnnotations();
    }
    catch (ArrayIndexOutOfBoundsException ex) {
      LOG.warn( MESSAGES.constraintOnConstructorOfNonStaticInnerClass(), ex );
      parameterAnnotations = EMPTY_PARAMETER_ANNOTATIONS;
    }

    Set<MetaConstraint<?>> parameterConstraints = newHashSet();

    if ( annotationProcessingOptions.areParameterConstraintsIgnoredFor( executable, i ) ) {
      Type type = ReflectionHelper.typeOf( executable, i );
      metaData.add(
          new ConstrainedParameter(
              ConfigurationSource.ANNOTATION,
              executable,
              type,
              i,
              parameterConstraints,
              Collections.emptySet(),
              CascadingMetaDataBuilder.nonCascading()
          )
      );
      i++;
      continue;
    }

    ConstraintLocation location = ConstraintLocation.forParameter( executable, i );

    for ( Annotation parameterAnnotation : parameterAnnotations ) {
      // collect constraints if this annotation is a constraint annotation
      List<ConstraintDescriptorImpl<?>> constraints = findConstraintAnnotations(
          executable, parameterAnnotation, ElementType.PARAMETER
      );
      for ( ConstraintDescriptorImpl<?> constraintDescriptorImpl : constraints ) {
        parameterConstraints.add(
            MetaConstraints.create( typeResolutionHelper, valueExtractorManager, constraintDescriptorImpl, location )
        );
      }
    }

    // **** 问题出在这里  ****
    AnnotatedType parameterAnnotatedType = parameter.getAnnotatedType();

    Set<MetaConstraint<?>> typeArgumentsConstraints = findTypeAnnotationConstraintsForExecutableParameter( executable, i, parameterAnnotatedType );
    CascadingMetaDataBuilder cascadingMetaData = findCascadingMetaData( executable, parameters, i, parameterAnnotatedType );

    metaData.add(
        new ConstrainedParameter(
            ConfigurationSource.ANNOTATION,
            executable,
            ReflectionHelper.typeOf( executable, i ),
            i,
            parameterConstraints,
            typeArgumentsConstraints,
            cascadingMetaData
        )
    );
    i++;
  }

  return metaData;
}
```

上面有两个中文注释，就是问题所在。  
首先，第一个问题，参数是 4 引起了我的疑惑。我们看下 `RuleType`  
```java
public enum RuleType {

	/**
	 * flow
	 */
	FLOW("flow", FlowRule.class),
	/**
	 * degrade
	 */
	DEGRADE("degrade", DegradeRule.class),
	/**
	 * param flow
	 */
	PARAM_FLOW("param-flow", ParamFlowRule.class),
	/**
	 * system
	 */
	SYSTEM("system", SystemRule.class),
	/**
	 * authority
	 */
	AUTHORITY("authority", AuthorityRule.class);

	/**
	 * alias for {@link AbstractRule}
	 */
	private final String name;

	/**
	 * concrete {@link AbstractRule} class
	 */
	private final Class clazz;

	RuleType(String name, Class clazz) {
		this.name = name;
		this.clazz = clazz;
	}

	public String getName() {
		return name;
	}

	public Class getClazz() {
		return clazz;
	}

	public static Optional<RuleType> getByName(String name) {
		if (StringUtils.isEmpty(name)) {
			return Optional.empty();
		}
		return Arrays.stream(RuleType.values())
				.filter(ruleType -> name.equals(ruleType.getName())).findFirst();
	}

	public static Optional<RuleType> getByClass(Class clazz) {
		return Arrays.stream(RuleType.values())
				.filter(ruleType -> clazz == ruleType.getClazz()).findFirst();
	}

}

```

只有一个构造方法，两个构造参数，为啥这里是 4 呢？     
调试的时候可以看到 前面两个参数 ，一个是枚举的 name ,一个是 ordinal 坐标  
说实话，以前还真没注意这个细节  

继续找问题所在  
```java
AnnotatedType parameterAnnotatedType = parameter.getAnnotatedType();
```

上面就是跑异常的地方  
```java
public AnnotatedType getAnnotatedType() {
    // no caching for now
    return executable.getAnnotatedParameterTypes()[index];
}
```

这里已经是 `jdk` 源码了 
最后发现，问题在这里 `executable.getAnnotatedParameterTypes()` 这个方法返回的数组长度为 2   
但是，我们可以看到，外层是在遍历 `executable.getParameters()` 这里返回的是 4 个元素，因此越界 

去获取 `spring-cloud-alibaba` 的源码，运行 `sentinel` 示例，也出现同样的问题  

写了一个测试例子  
```java
static enum EnumTest {

    A(0, "A"),

    B(1, "B")

    ;
    private final int code;
    private final String msg;

    EnumTest(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }
}

static enum Enum2 {
    A
    ;

}

public static void show(Class<?> aClass) {
    Constructor<?>[] declaredConstructors = aClass.getDeclaredConstructors();
    for (Constructor<?> constructor : declaredConstructors) {
        int parameterCount = constructor.getParameterCount();
        System.out.printf("parameterCount:%s\n", parameterCount);
        int length = constructor.getAnnotatedParameterTypes().length;
        System.out.printf("length:%s\n", length);
    }
}

    @Test
    public void test4() {
        show(EnumTest.class);
        System.out.println("---");
        show(Enum2.class);
    }
```

怀疑可能是 `jdk` 小版本的问题,目前用的 `jdk` 版本为 `1.8.25`  
因此去网上下载一个 `1.8.172` 发现没有问题 
```
1.8.25    parameterCount = 4,  length = 2
1.8.172   parameterCount = 4,  length = 4
```

到此，问题定位完毕。  
然而 `1.8.25` 是公司定的版本。    
通常搜索查询，这个 bug 在 `JDK 1.8.0_40` 之后修复了 

