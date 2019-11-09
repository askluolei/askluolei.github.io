---
title: artha 学习
date: 2019-07-27 00:57:02
tags: [java, artha]
categories: 技术笔记
---

[artha 文档地址](https://alibaba.github.io/arthas/index.html)
---
## artha 能做什么
官网摘抄：
>Arthas 是Alibaba开源的Java诊断工具
>当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：
>1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
>2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
>3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
>4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
>5. 是否有一个全局视角来查看系统的运行状况？
>6. 有什么办法可以监控到JVM的实时运行状态？
说简单点，就是针对正在运行的 `jvm` 进程,当无法使用日志分析出问题，本地没法或者很难重现的情况下，观察正在运行的 `jvm` 内部信息，包括整体情况，方法执行，变量信息等
---
## 原理
基于 `JDK6` 开始的 `Instrumentation` 功能
关于 `Instrumentation` 的介绍可以看 [Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)
摘抄:
>总的来说，基于 `Instrumentation` 功能，开发者可以构建一个独立于应用程序的代理程序（Agent），用来监测和协助运行在 JVM 上的程序，甚至能够替换和修改某些类的定义。
>有了这样的功能，开发者就可以实现更为灵活的运行时虚拟机监控和 Java 类操作了，这样的特性实际上提供了一种虚拟机级别支持的 AOP 实现方式，使得开发者无需对 JDK 做任何升级和改动，就可以实现某些 AOP 的功能了。
独立开发的代理程序，可以通过应用启动的时候指定，或者对运行中的 `JVM` 进行 `attach`。
基于 `artha` 大部分时候都是动态载入的，允许远程载入（没做测试）
其实，简单点说，就是在应用程序运行中的时候，加载代理（`artha`），然后 `artha` 启动 `tcp` 监听,接受各种命令，根据命令对相应的类做修改（本质就是 `aop` ）。
至于有什么功能，就看代理的实现了，`Instrumentation` 的 `api` 完全可以做到修改线程代码逻辑，运行时的数据。不过大部分这类代理通常只做局部修改查询，不修改数据，防止对线上应用造成严重影响。
---
## 安装
参见官方文档
不过文档推荐是访问各类镜像网站，下载，针对不能访问这些网站的时候，直接全量下载就可以了
[下载地址](http://repository.sonatype.org/service/local/artifact/maven/redirect?r=central-proxy&g=com.taobao.arthas&a=arthas-packaging&e=zip&c=bin&v=LATEST)
解压到应用服务器上面，`as.sh` 设置可执行权限
执行 `./as.sh` ,选择 `jvm` 进程，就可以了。
---
## 使用
`artha` 功能和命令挺多的，但是常用的就是几个，这里简单介绍一下，如果需要用其他命令，可以去看官方文档
---
### ognl
执行 `ognl` 表达式
格式为 `ognl '表达式' 对象展开层次`
```sh
# 调用静态函数：
$ ognl '@java.lang.System@out.println("hello")'
null
# 获取静态类的静态字段：
$ ognl '@demo.MathGame@random'
@Random[
  serialVersionUID=@Long[3905348978240129619],
  seed=@AtomicLong[125451474443703],
  multiplier=@Long[25214903917],
  addend=@Long[11],
  mask=@Long[281474976710655],
  DOUBLE_UNIT=@Double[1.1102230246251565E-16],
  BadBound=@String[bound must be positive],
  BadRange=@String[bound must be greater than origin],
  BadSize=@String[size must be non-negative],
  seedUniquifier=@AtomicLong[-3282039941672302964],
  nextNextGaussian=@Double[0.0],
  haveNextNextGaussian=@Boolean[false],
  serialPersistentFields=@ObjectStreamField[][isEmpty=false;size=3],
  unsafe=@Unsafe[sun.misc.Unsafe@28ea5898],
  seedOffset=@Long[24],
]
# 执行多行表达式，赋值给临时变量，返回一个List：
$ ognl '#value1=@System@getProperty("java.home"), #value2=@System@getProperty("java.runtime.name"), {#value1, #value2}'
@ArrayList[
  @String[/opt/java/8.0.181-zulu/jre],
  @String[OpenJDK Runtime Environment],
]
```
* [OGNL特殊用法请参考](https://github.com/alibaba/arthas/issues/71)
* [OGNL表达式官方指南](https://commons.apache.org/proper/commons-ognl/language-guide.html)
---
### getstatic
下面的内容都是官方文档里面的。
通过 `getstatic` 命令可以方便的查看类的静态属性。使用方法为 `getstatic class_name field_name`
简单用法
```sh
$ getstatic demo.MathGame random
field: random
@Random[
  serialVersionUID=@Long[3905348978240129619],
  seed=@AtomicLong[120955813885284],
  multiplier=@Long[25214903917],
  addend=@Long[11],
  mask=@Long[281474976710655],
  DOUBLE_UNIT=@Double[1.1102230246251565E-16],
  BadBound=@String[bound must be positive],
  BadRange=@String[bound must be greater than origin],
  BadSize=@String[size must be non-negative],
  seedUniquifier=@AtomicLong[-3282039941672302964],
  nextNextGaussian=@Double[0.0],
  haveNextNextGaussian=@Boolean[false],
  serialPersistentFields=@ObjectStreamField[][isEmpty=false;size=3],
  unsafe=@Unsafe[sun.misc.Unsafe@2eaa1027],
  seedOffset=@Long[24],
]
```
如果是复杂对象，可以使用 `ognl` 语法
例如，假设n是一个Map，Map的Key是一个Enum，我们想过滤出Map中Key为某个Enum的值，可以写如下命令
```sh
$ getstatic com.alibaba.arthas.Test n 'entrySet().iterator.{? #this.key.name()=="STOP"}'
field: n
@ArrayList[
  @Node[STOP=bbb],
]
Affect(row-cnt:1) cost in 68 ms.
$ getstatic com.alibaba.arthas.Test m 'entrySet().iterator.{? #this.key=="a"}'
field: m
@ArrayList[
  @Node[a=aaa],
]
```
---
### watch
方法执行数据观测
让你能方便的观察到指定方法的调用情况。能观察到的范围为：返回值、抛出异常、入参，通过编写 `OGNL` 表达式进行对应变量的查看。
格式为 `watch class-pattern method-pattern express condition-express [-x] x -[b|e|s|f|E] [-n] n`
* -x x 代表输出结果的遍历深度，默认是1，就是说如果返回对象层次很深，大部分时候是展示 `hashcode` ,设置了 这个参数，就可以展示内部信息了
* -b 在方法调用之前观察，这个就不会有异常和返回值了
* -e 在方法异常之后观察
* -s 在方法返回之后观察
* -f 在方法结束之后(正常返回和异常返回)观察,这个是默认的，不到任何观察点，那就是这个了
* -E 开启正则表达式匹配，默认为通配符匹配
* -n n 代表输出次数，默认是一直输出，如果观察的对象调用过于频繁，会刷屏，控制输出次数
其中 `express` 和 `condition-express` 可以使用 `ognl` 表达式
通常匹配尽量少的类和方法，减少命令影响的类的数量，最好直接定位
官方示例：
```sh
# 观察方法出参和返回值
$ watch demo.MathGame primeFactors "{params,returnObj}" -x 2
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 44 ms.
ts=2018-12-03 19:16:51; [cost=1.280502ms] result=@ArrayList[
@Object[][
  @Integer[535629513],
],
@ArrayList[
  @Integer[3],
  @Integer[19],
  @Integer[191],
  @Integer[49199],
],
]
# 观察方法入参
$ watch demo.MathGame primeFactors "{params,returnObj}" -x 2 -b
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 50 ms.
ts=2018-12-03 19:23:23; [cost=0.0353ms] result=@ArrayList[
@Object[][
  @Integer[-1077465243],
],
null,
]
# 对比前一个例子，返回值为空（事件点为方法执行前，因此获取不到返回值）
# 同时观察方法调用前和方法返回后
$ watch demo.MathGame primeFactors "{params,target,returnObj}" -x 2 -b -s -n 2
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 46 ms.
ts=2018-12-03 19:29:54; [cost=0.01696ms] result=@ArrayList[
@Object[][
  @Integer[1544665400],
],
@MathGame[
  random=@Random[java.util.Random@522b408a],
  illegalArgumentCount=@Integer[13038],
],
null,
]
ts=2018-12-03 19:29:54; [cost=4.277392ms] result=@ArrayList[
@Object[][
  @Integer[1544665400],
],
@MathGame[
random=@Random[java.util.Random@522b408a],
illegalArgumentCount=@Integer[13038],
],
@ArrayList[
  @Integer[2],
  @Integer[2],
  @Integer[2],
  @Integer[5],
  @Integer[5],
  @Integer[73],
  @Integer[241],
  @Integer[439],
],
]
# 参数里-n 2，表示只执行两次
# 这里输出结果中，第一次输出的是方法调用前的观察表达式的结果，第二次输出的是方法返回后的表达式的结果
# 结果的顺序和命令中 -s -b 的顺序没有关系，只与事件本身的先后顺序有关
# 调整-x的值，观察具体的方法参数值
$ watch demo.MathGame primeFactors "{params,target}" -x 3
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 58 ms.
ts=2018-12-03 19:34:19; [cost=0.587833ms] result=@ArrayList[
@Object[][
  @Integer[47816758],
],
@MathGame[
  random=@Random[
  serialVersionUID=@Long[3905348978240129619],
  seed=@AtomicLong[3133719055989],
  multiplier=@Long[25214903917],
  addend=@Long[11],
  mask=@Long[281474976710655],
  DOUBLE_UNIT=@Double[1.1102230246251565E-16],
  BadBound=@String[bound must be positive],
  BadRange=@String[bound must be greater than origin],
  BadSize=@String[size must be non-negative],
  seedUniquifier=@AtomicLong[-3282039941672302964],
  nextNextGaussian=@Double[0.0],
  haveNextNextGaussian=@Boolean[false],
  serialPersistentFields=@ObjectStreamField[][isEmpty=false;size=3],
  unsafe=@Unsafe[sun.misc.Unsafe@2eaa1027],
  seedOffset=@Long[24],
],
illegalArgumentCount=@Integer[13159],
],
]
# -x表示遍历深度，可以调整来打印具体的参数和结果内容，默认值是1
# 条件表达式的例子
$ watch demo.MathGame primeFactors "{params[0],target}" "params[0]<0"
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 68 ms.
ts=2018-12-03 19:36:04; [cost=0.530255ms] result=@ArrayList[
  @Integer[-18178089],
  @MathGame[demo.MathGame@41cf53f9],
]
# 只有满足条件的调用，才会有响应。
# 观察异常信息的例子
$ watch demo.MathGame primeFactors "{params[0],throwExp}" -e -x 2
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 62 ms.
ts=2018-12-03 19:38:00; [cost=1.414993ms] result=@ArrayList[
@Integer[-1120397038],
java.lang.IllegalArgumentException: number is: -1120397038, need >= 2
at demo.MathGame.primeFactors(MathGame.java:46)
at demo.MathGame.run(MathGame.java:24)
at demo.MathGame.main(MathGame.java:16)
,
]
# -e表示抛出异常时才触发
# express中，表示异常信息的变量是throwExp
# 按照耗时进行过滤
$ watch demo.MathGame primeFactors '{params, returnObj}' '#cost>200' -x 2
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 66 ms.
ts=2018-12-03 19:40:28; [cost=2112.168897ms] result=@ArrayList[
@Object[][
  @Integer[2141897465],
],
@ArrayList[
  @Integer[5],
  @Integer[428379493],
],
]
#cost>200(单位是ms)表示只有当耗时大于200ms时才会输出，过滤掉执行时间小于200ms的调用
# 观察当前对象中的属性
$ watch demo.MathGame primeFactors 'target'
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 52 ms.
ts=2018-12-03 19:41:52; [cost=0.477882ms] result=@MathGame[
  random=@Random[java.util.Random@522b408a],
  illegalArgumentCount=@Integer[13355],
]
# 然后使用 `target.field_name` 访问当前对象的某个属性
$ watch demo.MathGame primeFactors 'target.illegalArgumentCount'
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 67 ms.
ts=2018-12-03 20:04:34; [cost=131.303498ms] result=@Integer[8]
ts=2018-12-03 20:04:35; [cost=0.961441ms] result=@Integer[8]
```
---
### monitor
方法执行监控
用来统计一段时间内方法的调用情况(耗时，是否成功)
格式为 `monitor class-pattern method-pattern [-E] [-c] c`
* -E 开启正则表达式匹配，默认为通配符匹配
* -c c 统计周期，默认值为120秒,就是多长时间统计输出一次
统计表格的内容有：
| 监控项 | 说明 |
| --- | ---- |
| timestamp | 时间戳 |
| class | Java类 |
| method | 方法（构造方法、普通方法） |
| total | 调用次数 |
| success | 成功次数 |
| fail | 失败次数 |
| rt | 平均RT |
| fail-rate | 失败率 |
示例:
```sh
$ monitor -c 5 demo.MathGame primeFactors
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 94 ms.
timestamp class method total success fail avg-rt(ms) fail-rate
-----------------------------------------------------------------------------------------------
2018-12-03 19:06:38 demo.MathGame primeFactors 5 1 4 1.15 80.00%
timestamp class method total success fail avg-rt(ms) fail-rate
-----------------------------------------------------------------------------------------------
2018-12-03 19:06:43 demo.MathGame primeFactors 5 3 2 42.29 40.00%
timestamp class method total success fail avg-rt(ms) fail-rate
-----------------------------------------------------------------------------------------------
2018-12-03 19:06:48 demo.MathGame primeFactors 5 3 2 67.92 40.00%
timestamp class method total success fail avg-rt(ms) fail-rate
-----------------------------------------------------------------------------------------------
2018-12-03 19:06:53 demo.MathGame primeFactors 5 2 3 0.25 60.00%
timestamp class method total success fail avg-rt(ms) fail-rate
-----------------------------------------------------------------------------------------------
2018-12-03 19:06:58 demo.MathGame primeFactors 1 1 0 0.45 0.00%
timestamp class method total success fail avg-rt(ms) fail-rate
-----------------------------------------------------------------------------------------------
2018-12-03 19:07:03 demo.MathGame primeFactors 2 2 0 3182.72 0.00%
```
---
### trace
方法内部调用路径，并输出方法路径上的每个节点上耗时
`trace` 能方便的帮助你定位和发现因 RT 高而导致的性能问题缺陷，但其每次只能跟踪一级方法的调用链路。
场景就是当一个方法很耗时的时候，通过 `trace` 命令，查看这个方法内部其他方法调用的耗时，看到底是哪个方法耗时严重
格式为 `trace class-pattern method-pattern condition-express [-E] [-n] n [-j]`
* -E 开启正则表达式匹配，默认为通配符匹配
* -n n 命令执行次数,也就是输出次数
* -j 过滤掉 `jdk` 的方法
如果内部方法有多次调用，会一起统计
统计的时候没有减去自身的开销，统计结果不太精确
示例:
```sh
# trace函数
$ trace demo.MathGame run
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 42 ms.
`---ts=2018-12-04 00:44:17;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
`---[10.611029ms] demo.MathGame:run()
+---[0.05638ms] java.util.Random:nextInt()
+---[10.036885ms] demo.MathGame:primeFactors()
`---[0.170316ms] demo.MathGame:print()
# 过滤掉jdk的函数
$ trace -j demo.MathGame run
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 31 ms.
`---ts=2018-12-04 01:09:14;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
`---[5.190646ms] demo.MathGame:run()
+---[4.465779ms] demo.MathGame:primeFactors()
`---[0.375324ms] demo.MathGame:print()
# 据调用耗时过滤
$ trace demo.MathGame run '#cost > 10'
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 41 ms.
`---ts=2018-12-04 01:12:02;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
`---[12.033735ms] demo.MathGame:run()
+---[0.006783ms] java.util.Random:nextInt()
+---[11.852594ms] demo.MathGame:primeFactors()
`---[0.05447ms] demo.MathGame:print()
# 只会展示耗时大于 10ms 的调用路径，有助于在排查问题的时候，只关注异常情况
```
---
### tt
方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测
举个例子，有一个 `hello(String name)` 方法，很多人都需要调用，但是每个人的参数都不一样，我只关心我自己的调用
虽然 `watch` 命令可以看到参数，但是需要提前想好过滤条件，也没法重试
`tt` 的作用就是记录指定次数的方法调用，记录下来，后面可以查看这些记录，重试调用等
**记录调用**
格式 `tt -t class-pattern method-pattern [-n] n`
* -n n 记录命令执行次数
展示:
| 表格字段 | 字段解释 |
| --- | ---- |
| INDEX | 时间片段记录编号，每一个编号代表着一次调用，后续tt还有很多命令都是基于此编号指定记录操作，非常重要。 |
| TIMESTAMP | 方法执行的本机时间，记录了这个时间片段所发生的本机时间 |
| COST(ms) | 方法执行的耗时 |
| IS-RET | 方法是否以正常返回的形式结束 |
| IS-EXP | 方法是否以抛异常的形式结束 |
| OBJECT | 执行对象的hashCode()，注意，曾经有人误认为是对象在JVM中的内存地址，但很遗憾他不是。但他能帮助你简单的标记当前执行方法的类实体 |
| CLASS | 执行的类名 |
| METHOD | 执行的方法名 |
示例：
```sh
$ tt -t demo.MathGame primeFactors
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 66 ms.
INDEX TIMESTAMP COST(ms) IS-RET IS-EXP OBJECT CLASS METHOD
-------------------------------------------------------------------------------------------------------------------------------------
1000 2018-12-04 11:15:38 1.096236 false true 0x4b67cf4d MathGame primeFactors
1001 2018-12-04 11:15:39 0.191848 false true 0x4b67cf4d MathGame primeFactors
1002 2018-12-04 11:15:40 0.069523 false true 0x4b67cf4d MathGame primeFactors
1003 2018-12-04 11:15:41 0.186073 false true 0x4b67cf4d MathGame primeFactors
1004 2018-12-04 11:15:42 17.76437 true false 0x4b67cf4d MathGame primeFactors
```
**检索调用记录**
```sh
# 所有记录
$ tt -l
INDEX TIMESTAMP COST(ms) IS-RET IS-EXP OBJECT CLASS METHOD
-------------------------------------------------------------------------------------------------------------------------------------
1000 2018-12-04 11:15:38 1.096236 false true 0x4b67cf4d MathGame primeFactors
1001 2018-12-04 11:15:39 0.191848 false true 0x4b67cf4d MathGame primeFactors
1002 2018-12-04 11:15:40 0.069523 false true 0x4b67cf4d MathGame primeFactors
1003 2018-12-04 11:15:41 0.186073 false true 0x4b67cf4d MathGame primeFactors
1004 2018-12-04 11:15:42 17.76437 true false 0x4b67cf4d MathGame primeFactors
9
1005 2018-12-04 11:15:43 0.4776 false true 0x4b67cf4d MathGame primeFactors
Affect(row-cnt:6) cost in 4 ms.
# 筛选
$ tt -s 'method.name=="primeFactors"'
INDEX TIMESTAMP COST(ms) IS-RET IS-EXP OBJECT CLASS METHOD
-------------------------------------------------------------------------------------------------------------------------------------
1000 2018-12-04 11:15:38 1.096236 false true 0x4b67cf4d MathGame primeFactors
1001 2018-12-04 11:15:39 0.191848 false true 0x4b67cf4d MathGame primeFactors
1002 2018-12-04 11:15:40 0.069523 false true 0x4b67cf4d MathGame primeFactors
1003 2018-12-04 11:15:41 0.186073 false true 0x4b67cf4d MathGame primeFactors
1004 2018-12-04 11:15:42 17.76437 true false 0x4b67cf4d MathGame primeFactors
9
1005 2018-12-04 11:15:43 0.4776 false true 0x4b67cf4d MathGame primeFactors
Affect(row-cnt:6) cost in 607 ms.
```
**查看调用信息**
`-i` 指定 `INDEX`
```sh
$ tt -i 1003
INDEX 1003
GMT-CREATE 2018-12-04 11:15:41
COST(ms) 0.186073
OBJECT 0x4b67cf4d
CLASS demo.MathGame
METHOD primeFactors
IS-RETURN false
IS-EXCEPTION true
PARAMETERS[0] @Integer[-564322413]
THROW-EXCEPTION java.lang.IllegalArgumentException: number is: -564322413, need >= 2
at demo.MathGame.primeFactors(MathGame.java:46)
at demo.MathGame.run(MathGame.java:24)
at demo.MathGame.main(MathGame.java:16)
Affect(row-cnt:1) cost in 11 ms.
```
**重做一次调用**
>当你稍稍做了一些调整之后，你可能需要前端系统重新触发一次你的调用，此时得求爷爷告奶奶的需要前端配合联调的同学再次发起一次调用。而有些场景下，这个调用不是这么好触发的。
>tt 命令由于保存了当时调用的所有现场信息，所以我们可以自己主动对一个 INDEX 编号的时间片自主发起一次调用，从而解放你的沟通成本。此时你需要 -p 参数。
```sh
$ tt -i 1004 -p
RE-INDEX 1004
GMT-REPLAY 2018-12-04 11:26:00
OBJECT 0x4b67cf4d
CLASS demo.MathGame
METHOD primeFactors
PARAMETERS[0] @Integer[946738738]
IS-RETURN true
IS-EXCEPTION false
RETURN-OBJ @ArrayList[
@Integer[2],
@Integer[11],
@Integer[17],
@Integer[2531387],
]
Time fragment[1004] successfully replayed.
Affect(row-cnt:1) cost in 14 ms.
```
重做一次调用需要注意的地方：
1. `ThreadLocal` 信息丢失
很多框架偷偷的将一些环境变量信息塞到了发起调用线程的 `ThreadLocal` 中，由于调用线程发生了变化，这些 `ThreadLocal` 线程信息无法通过 `Arthas` 保存，所以这些信息将会丢失。
一些常见的例子 比如：鹰眼的 `TraceId` 等。
2. 引用的对象
需要强调的是，`tt` 命令是将当前环境的对象引用保存起来，但仅仅也只能保存一个引用而已。如果方法内部对入参进行了变更，或者返回的对象经过了后续的处理，那么在 `tt` 查看的时候将无法看到当时最准确的值。这也是为什么 `watch` 命令存在的意义。
---
## 其他可能用到的功能
详细的可以看官方文档
### 后台任务
1. 使用&在后台执行任务
比如希望执行后台执行trace命令，那么调用下面命令
```sh
trace Test t &
```
这时命令在后台执行，可以在console中继续执行其他命令。
2. 通过 `jobs` 查看任务
如果希望查看当前有哪些 `arthas` 任务在执行，可以执行 `jobs` 命令，执行结果如下
```sh
$ jobs
[10]*
Stopped watch com.taobao.container.Test test "params[0].{? #this.name == null }" -x 2
execution count : 19
start time : Fri Sep 22 09:59:55 CST 2017
timeout date : Sat Sep 23 09:59:55 CST 2017
session : 3648e874-5e69-473f-9eed-7f89660b079b (current)
```
可以看到目前有一个后台任务在执行。
* job id是10, `*` 表示此job是当前session创建
* 状态是Stopped
* execution count是执行次数，从启动开始已经执行了19次
* timeout date是超时的时间，到这个时间，任务将会自动超时退出
3. 任务暂停和取消
当任务正在前台执行，比如直接调用命令`trace Test t`或者调用后台执行命令`trace Test t &`后又通过`fg`命令将任务转到前台。这时`console`中无法继续执行命令，但是可以接收并处理以下事件：
* `‘ctrl + z’`：将任务暂停。通过`jbos`查看任务状态将会变为Stopped，通过`bg` 或者`fg` 可让任务重新开始执行
* `‘ctrl + c’`：停止任务
* `‘ctrl + d’`：按照`linux`语义应当是退出终端，目前`arthas`中是空实现，不处理
4. fg、bg命令，将命令转到前台、后台继续执行
* 任务在后台执行或者暂停状态（`ctrl + z`暂停任务）时，执行`fg` 将可以把对应的任务转到前台继续执行。在前台执行时，无法在console中执行其他命令
* 当任务处于暂停状态时（`ctrl + z`暂停任务），执行`bg` 将可以把对应的任务在后台继续执行
* 非当前session创建的job，只能由当前session `fg`到前台执行
5. 任务输出重定向
可通过`>`或者`>>`将任务输出结果输出到指定的文件中，可以和&一起使用，实现`arthas`命令的异步调用。比如：
```sh
$ trace Test t >> test.out &
```
这时`trace`命令会在后台执行，并且把结果输出到`~/logs/arthas-cache/test.out`。可继续执行其他命令。并可查看文件中的命令执行结果。
当连接到远程的`arthas server`时，可能无法查看远程机器的文件，`arthas`同时支持了自动重定向到本地缓存路径。使用方法如下：
```sh
$ trace Test t >> &
job id : 2
cache location : /Users/gehui/logs/arthas-cache/28198/2
```
可以看到并没有指定重定向文件位置，`arthas`自动重定向到缓存中了，执行命令后会输出job id和cache location。cache location就是重定向文件的路径，在系统logs目录下，路径包括pid和job id，避免和其他任务冲突。命令输出结果到`/Users/gehui/logs/arthas-cache/28198/2`中，job id为2。
6. 停止命令
异步执行的命令，如果希望停止，可执行kill <job-id>
7. 其他
最多同时支持8个命令使用重定向将结果写日志
请勿同时开启过多的后台异步命令，以免对目标JVM性能造成影响
---
### 结果存日志
将命令的结果完整保存在日志文件中，便于后续进行分析
默认情况下，该功能是关闭的，如果需要开启，请执行以下命令：
```sh
$ options save-result true
NAME BEFORE-VALUE AFTER-VALUE
----------------------------------------
save-result false true
Affect(row-cnt:1) cost in 3 ms.
```
看到上面的输出，即表示成功开启该功能；
结果会异步保存在：`{user.home}/logs/arthas-cache/result.log`，请定期进行清理，以免占据磁盘空间
**使用新版本Arthas的异步后台任务将结果存日志文件**
```sh
$ trace Test t >> &
job id : 2
cache location : /Users/admin/logs/arthas-cache/28198/2
```
此时命令会在后台异步执行，并将结果异步保存在文件（`~/logs/arthas-cache/${PID}/${JobId}`）中；
* 此时任务的执行不受session断开的影响；任务默认超时时间是1天，可以通过全局 `options` 命令修改默认超时时间；
* 此命令的结果将异步输出到文件中；此时不管 `save-result 是否为true，都不会再往`~/logs/arthas-cache/result.log` 中异步写结果

