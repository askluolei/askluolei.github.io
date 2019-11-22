---
title: java 泛型
date: 2019-09-04 00:51:45
tags: [java, spring, 泛型]
categories: 技术笔记
---

用了这么久的 `spirng` ,以为对 `ioc` 的原理和实现机制已经非常了解了  
但是，突然想到泛型的依赖注入是怎么实现的，这就很纠结了，因为普通的依赖注入，自己想一下，实现起来简单，但是涉及到泛型，实现就比较复杂了，需要对 `jdk` 的 `Type` 有了解   
这里举几个例子，看看泛型依赖注入的效果：
```java
public class Book {
}

public class User {
}

public class BaseDao<T, ID> {

    public ID save(T t) {
        System.out.println(t.getClass().getName());
        return null;
    }
}

@Component
public class BookDao extends BaseDao<Book, Integer> {
}

@Component
public class UserDao extends BaseDao<User, Integer> {
}

@Service
public class MyService {

    @Autowired
    private BaseEnhanceDao<Book> bookBaseDao;

    @Autowired
    private BaseEnhanceDao<User> userBaseDao;

    public void hello() {
        bookBaseDao.save(new Book());
        System.out.println("-=-=-= 我是分割线 -=-=-=");
        userBaseDao.save(new User());
    }
}

```

这是一个常用的，很简单的泛型注入，可以看到 `spring` 在 `MyService` 类的字段 `BaseEnhanceDao<Book>` 跟 `BookDao` 看做同一个类型，而跟 `UserDao` 不是同一个类型（如果是，依赖注入就会报错,多个满足条件的 `bean`,但是他们在原生类型上，是存在继承关系的）   
如果要我们自己实现，如何判断这两个类型是同一个呢？  
再扩展一点，上面的只是一个简单的继承体系，一个最终的实现类，他的泛型可能来自 `superClass` 或者 `interface` ，而且都可以向上延伸 
举例  
```java
public class A1 {
}

public class A2 {
}

public class A3 {
}

// 泛型顶层类
public class Ab2<T, F, H> {
}

public class AB3<T, F> extends Ab2<T, F, A1> {
}

public class AB4<T> extends AB3<T, A2> {
}

public class AB5 extends AB4<A3> {
}

```
这里例子，很明显的看到 `AB5` 有3个泛型类型都是通过继承来的, 如果 `AB5` 注册为 `bean`，他可以代表的类型是 `Ab2<A3, A2, A1>` , `AB3<A3, A2>` , `AB4<A3>` ,如果中间的泛型顺序不对，那么就不是同一个类型  

再来看个接口的例子：
```java
// 省略 OBJ1,2,3,4
public interface I1<A, B, C, D> {
}
public interface I2<A, B, C> extends I1<A, B, C, OBJ4> {
}
public interface I3<A, B> extends I2<A, B, OBJ3> {
}
public interface I4<A> extends I3<A, OBJ2> {
}
public interface I5 extends I4<OBJ1> {
}
public class Impl1 implements I5 {
}
public class Impl2 extends Impl1 {
}
```

无论是 `Impl1` 还是 `Impl2` 都应该是类型 `I1 I2 I3 I4 I5` 类型的（原生类型），如果考虑到泛型，那么泛型的具体类型，数量和顺序也是关键，可以自然的想到，当需要注入一个  
```java

@Autowired
private I1<OBJ1, OBJ2, OBJ3, OBJ4> i1;
```

字段的时候， `Impl1` `Impl2` 都是满足条件的， `spring` 也能够处理   
现在我们思考一下，如果要我们自己去实现这个功能，要怎么做呢？要同时满足接口和超类  
测试的时候不需要考虑注入问题，只要能够判断 `Impl1` 是类型 `I1<OBJ1, OBJ2, OBJ3, OBJ4>`  而不是类型 `I1<OBJ4, OBJ3, OBJ2, OBJ1>` 就可以了  


所有，我们有这么一个方法：  
```java
public static boolean isSameType(Type sourceType, Type toMatchedType)
```

其中 `sourceType` 的具体例子就是 `I1<OBJ1, OBJ2, OBJ3, OBJ4>` , `toMatchedType` 的例子就是 `Impl1`  
返回是否为同一个类型  
看到，参数给的是 `Type` 类型，而不是 `Class` 如果传原生的 `Class` 可能会丢到泛型信息，当通过 `field` 获取字段类型的时候，使用 `getGenericType()` 方法获取 `Type` 类型 
为了实现这个功能，就需要对 `jdk` 的 `Type` 体系有点了解了，毕竟，我们通常使用的是 `Class` 而不是 `Type`， `getGenericType()` 返回的是什么呢？ 
JDK 里面的 Type 定义如下
```java
public interface Type {
    /**
     * Returns a string describing this type, including information
     * about any type parameters.
     *
     * @implSpec The default implementation calls {@code toString}.
     *
     * @return a string describing this type
     * @since 1.8
     */
    default String getTypeName() {
        return toString();
    }
}
```

可以调用 `getTypeName()` 方法，来看里面包含什么泛型信息（这里只是调试看的） 
`Type` 跟 `Class` 是什么关系？继承关系，`Class` 也是一个 `Type`   
`Type` 下面有几个比较重要的体系 `ParameterizedType`  `TypeVariable`  `Class`  这三个比较重要，另外还有 `GenericArrayType` 和 `WildcardType` 
其中 `GenericArrayType` 代表泛型数组，就是字段类型是 `T[] array` 这样的   
`WildcardType` 是通配符表达式 类似 `List<? extends Number> list` 这样的，可以看成是特殊的 TypeVariable  
本次研究不看 `GenericArrayType` 和 `WildcardType`   

首先要明确的是，最终的实现类，一定要确认所有的泛型类型才能注册为 `bean`，所以，他的泛型信息肯定是在 `superClass` 或者 `interface(s)` 里面的   
先说简单情况，只有继承关系，没有接口的干预  
通常，我们获取父类调用 `Class` 的 `getSuperClass` 这个方法会丢掉泛型信息，我们需要换一个方法 `getGenericSuperclass` 这个方法返回一个 `Type` 
如果父类不是泛型类，没有泛型标签，那么返回的是 `Class` 类型（实际），否则返回的是 `ParameterizedType` 类型(实现类)  
而 `ParameterizedType` 里面有几个重要方法 `getRawType` 返回原生类型，就跟 `getSuperClass` 返回一样, `getActualTypeArguments()` 返回泛型参数列表 `Type[]`  
这里的 `Type[]` 数组，里面具体是什么类型的呢？ 可能是 `Class` 可能是 `TypeVariable` 和 `WildcardType` 
如果是 `Class` 好说，就是这本层指定了具体的泛型类型，譬如 `public class AB4<T> extends AB3<T, A2>`  如果是 `AB4` 调用 `getActualTypeArguments` 那么泛型参数列表 
就是 `<T, A2, A1>` 有3个泛型参数，其中后两个确定了，那么就是 `Class` 类型， `T` 代表没确定，就是 `TypeVariable` 暂时先不考虑限定符的问题，这里的 `T` 可能是子类指定的，所以很自然的想到 `Ab2<A3, A2, A1>` 跟 `AB5` 是不是同一个类型的时候，需要判断的是 `AB5` 原生类型是不是 `Ab2` 的子类，以及他们的泛型参数列表是不是相同的，其中 `Ab2` 的泛型参数列表好获取，重点在于 `AB5` 的，他的泛型参数是从不同的实现类指定了。   

如果实现了上面说的超类的，那么请再考虑一下 `interface` 的，想想也知道，更为复杂，因为接口允许多继承，各位可以自己想一想，不需要考虑限定符 
下面给出参考实现(简单测试过没啥问题，不过不保证):
```java
public class GenericUtil {

    /**
     * 判断两个类型是否为同一个类型
     * @param sourceType 目标类型
     * @param toMatchedType 待匹配类型
     * @return
     */
    public static boolean isSameType(Type sourceType, Type toMatchedType) {
        // 为 null，就不是
        if (sourceType == null || toMatchedType == null) {
            return false;
        }
        Class<?> sourceClass = null;
        Class<?> toMatchedClass = null;
        ParameterizedType sourceParameterizedType = null;
        ParameterizedType toMatchedParameterizedType = null;
        if (Class.class.isInstance(sourceType)) {
            sourceClass = (Class<?>) sourceType;
        } else if (ParameterizedType.class.isInstance(sourceType)){
            sourceParameterizedType = (ParameterizedType) sourceType;
            sourceClass = (Class<?>) sourceParameterizedType.getRawType();
        }
        if (Class.class.isInstance(toMatchedType)) {
            toMatchedClass = (Class<?>) toMatchedType;
        } else if (ParameterizedType.class.isInstance(toMatchedType)){
            toMatchedParameterizedType = (ParameterizedType) toMatchedType;
            toMatchedClass = (Class<?>) toMatchedParameterizedType.getRawType();
        }
        if (sourceClass == null | toMatchedClass == null || !sourceClass.isAssignableFrom(toMatchedClass)) {
            return false;
        }
        // 正常情况下，到这里，sourceType 应该就是泛型类型了，如果不是，直接返回
        if (sourceParameterizedType == null) {
            return false;
        }
        // 这里是待匹配的泛型类型
        Type[] actualTypeArguments = sourceParameterizedType.getActualTypeArguments();
        // 现在的重点是拿 toMatched 的泛型类型，但是，这个类可能就是单纯的 class，他的泛型可能是从 superClass 和 interface 获取到的
        // 思路是，从 matchedType 向上（接口线，或者super线，或者 super + 接口线）到 sourceType 的 rawType 类型,并找到这条线上的泛型参数，对比 actualTypeArguments

        List<ParameterizedType> list = new ArrayList<>();
        Type parent = toMatchedClass;
        Type result = searchVertical(sourceClass, parent, list);
        if (result != null) {
            addResult(result, list);
        }
        // 这样查询后，最顶层的类在前面
        //
        if (!list.isEmpty()) {
            // 第一个类或者接口就是 sourceClass
            ParameterizedType parameterizedType = list.get(0);
            // 这里就待匹配的泛型了，但是里面可能是 T 这样的 TypeVariable ，具体的泛型在下面
            Type[] matchedTypeArguments = parameterizedType.getActualTypeArguments();
            Type[] realMatched = new Type[matchedTypeArguments.length];
            for (int i = 0; i < matchedTypeArguments.length; i++) {
                Type type = matchedTypeArguments[i];
                if (TypeVariable.class.isInstance(type)) {
                    TypeVariable<?> typeVariable = (TypeVariable<?>) type;
                    // 这里是泛型变量
                    for (int j = 1; j < list.size(); j++) {
                        ParameterizedType p2 = list.get(j);
                        Class<?> rawType = (Class<?>) p2.getRawType();
                        TypeVariable<? extends Class<?>>[] typeParameters = rawType.getTypeParameters();
                        if (typeParameters.length == 0) {
                            continue;
                        }
                        boolean find = false;
                        for (int k = 0; k < typeParameters.length; k++) {
                            if (typeParameters[k].getName().equals(typeVariable.getName())) {
                                Type t2 = p2.getActualTypeArguments()[k];
                                if (Class.class.isInstance(t2)) {
                                    realMatched[i] = t2;
                                    find = true;
                                    break;
                                }
                            }
                        }
                        if (find) {
                            break;
                        }
                    }
                } else {
                    realMatched[i] = type;
                }
            }
            if (actualTypeArguments.length != realMatched.length) {
                return false;
            }
            for (int i = 0; i < actualTypeArguments.length; i++) {
                Class<?> actualType = (Class<?>) actualTypeArguments[i];
                Class<?> toMatchType = (Class<?>) realMatched[i];
                if (!actualType.isAssignableFrom(toMatchType)) {
                    return false;
                }
            }
            return true;
        }
        return false;
    }

    /**
     * 垂直查询继承体系
     * 递归调用
     * @param sourceClass
     * @param current
     * @param list
     * @return
     */
    private static Type searchVertical(Class<?> sourceClass, Type current, List<ParameterizedType> list) {
        if (Class.class.isInstance(current)) {
            // 如果是 class 代表没泛型
            Class<?> currentClass = (Class<?>) current;
            if (currentClass == Object.class) {
                // 代表搜寻到 Object 了
                return null;
            }
            return searchClass(sourceClass, currentClass, current, list);
        } else if (ParameterizedType.class.isInstance(current)) {
            ParameterizedType parameterizedType = (ParameterizedType) current;
            Class<?> rawClass = (Class<?>) parameterizedType.getRawType();
            if (rawClass == sourceClass) {
                // 代表搜寻完毕
                return parameterizedType;
            } else {
                return searchClass(sourceClass, rawClass, current, list);
            }
        }
        return null;
    }

    /**
     * 继续向上，搜索父类或者接口
     * @param sourceClass
     * @param searchClass
     * @param current
     * @param list
     * @return
     */
    private static Type searchClass(Class<?> sourceClass, Class<?> searchClass, Type current, List<ParameterizedType> list) {
        for (Type type : searchClass.getGenericInterfaces()) {
            Type result = searchVertical(sourceClass, type, list);
            if (result != null ) {
                addResult(result, list);
                return type;
            }
        }
        Type superclass = searchClass.getGenericSuperclass();
        Type result = searchVertical(sourceClass, superclass, list);
        if (result != null) {
            addResult(result, list);
            return current;
        }
        return null;
    }

    /**
     * 如果是泛型，那么添加到列表里面
     * @param result
     * @param list
     */
    private static void addResult(Type result, List<ParameterizedType> list) {
        if (ParameterizedType.class.isInstance(result)) {
            list.add((ParameterizedType) result);
        }
    }

}
```




给出测试代码  
```java

    // 修改泛型列表的具体类型  
    private Ab2<A3, A2, A1> ab;

    @Test
    public void test9() throws Exception {
        Field field = getClass().getDeclaredField("ab");
        Type genericType = field.getGenericType();

        AB5 ab5 = new AB5();
        System.out.println();
        GenericTest test = new GenericTest();
        boolean sameType = GenericUtil.isSameType(genericType, ab5.getClass());
        if (sameType) {
            System.out.println("same type");
            field.set(test, ab5);
            System.out.println("set field success");
        } else {
            System.out.println("diff type");
        }

    }

    // 修改泛型列表的具体类型  
    private I1<OBJ1, OBJ2, OBJ3, OBJ4> i1;
    @Test
    public void test10() throws Exception {
        Field field = getClass().getDeclaredField("i1");
        Type genericType = field.getGenericType();

        Impl2 impl1 = new Impl2();

        GenericTest test = new GenericTest();
        boolean sameType = GenericUtil.isSameType(genericType, impl1.getClass());
        if (sameType) {
            System.out.println("same type");
            field.set(test, impl1);
            System.out.println("set field success");
        } else {
            System.out.println("diff type");
        }
    }
```
