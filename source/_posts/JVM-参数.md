---
title: JVM 参数
date: 2019-07-28 21:15:57
tags: [java,jvm]
categories: 技术笔记
---

## Trace 跟踪参数
### -XX:+PrintGC
打印GC 的简要信息
```java
[GC (Allocation Failure) [PSYoungGen: 975K->480K(1536K)] 14287K->14016K(15360K), 0.0067338 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
```
### -XX:+PrintGCTimeStamps
打印 GC 发生的时间戳
### -XX:+PrintGCDetails
程序运行结束后，打印GC的日志
PSYongGen 新生代 包含下面三个
eden 对象出生的地方
from 幸存 from to 两个对等
to
ParOldGen 老年代
元空间
Metaspace
JDK 好像没有永久代了
JDK 8开始把类的元数据放到本地化的堆内存(native heap)中，这一块区域就叫Metaspace，中文名叫元空间。
新增参数
-XX:MetaspaceSize是分配给类元数据空间（以字节计）的初始大小(Oracle逻辑存储上的初始高水位，the initial high-water-mark )，此值为估计值。MetaspaceSize的值设置的过大会延长垃圾回收时间。垃圾回收过后，引起下一次垃圾回收的类元数据空间的大小可能会变大。
-XX:MaxMetaspaceSize是分配给类元数据空间的最大值，超过此值就会触发Full GC，此值默认没有限制，但应取决于系统内存的大小。JVM会动态地改变此值。
-XX:MinMetaspaceFreeRatio表示一次GC以后，为了避免增加元数据空间的大小，空闲的类元数据的容量的最小比例，不够就会导致垃圾回收。
-XX:MaxMetaspaceFreeRatio表示一次GC以后，为了避免增加元数据空间的大小，空闲的类元数据的容量的最大比例，不够就会导致垃圾回收。
后面括号里面
分别为 低边界， 当前边界， 最高边界
```java
Heap
 PSYoungGen total 1536K, used 31K [0x00000000ff980000, 0x00000000ffb80000, 0x0000000100000000)
  eden space 1024K, 3% used [0x00000000ff980000,0x00000000ff987c68,0x00000000ffa80000)
  from space 512K, 0% used [0x00000000ffa80000,0x00000000ffa80000,0x00000000ffb00000)
  to space 512K, 0% used [0x00000000ffb00000,0x00000000ffb00000,0x00000000ffb80000)
 ParOldGen total 10240K, used 8863K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff980000) 
  object space 10240K, 86% used [0x00000000fec00000,0x00000000ff4a7e68,0x00000000ff600000)
 Metaspace used 2690K, capacity 4486K, committed 4864K, reserved 1056768K
  class space used 297K, capacity 386K, committed 512K, reserved 1048576K
```
### -Xloggc:log/gc.log
指定 GC log 的位置，以文件输出
注意一点，上级目录要存在
### -XX:+PrintHeapAtGC
每一次GC后，打印堆信息（GC 前后）
开发调试的时候可以用一下
```java
{Heap before GC invocations=1 (full 0):
 PSYoungGen total 1536K, used 975K [0x00000000ff980000, 0x00000000ffb80000, 0x0000000100000000)
  eden space 1024K, 95% used [0x00000000ff980000,0x00000000ffa73c20,0x00000000ffa80000)
  from space 512K, 0% used [0x00000000ffb00000,0x00000000ffb00000,0x00000000ffb80000)
  to space 512K, 0% used [0x00000000ffa80000,0x00000000ffa80000,0x00000000ffb00000)
 ParOldGen total 13824K, used 13312K [0x00000000fec00000, 0x00000000ff980000, 0x00000000ff980000)
  object space 13824K, 96% used [0x00000000fec00000,0x00000000ff9000d0,0x00000000ff980000)
 Metaspace used 2682K, capacity 4486K, committed 4864K, reserved 1056768K
  class space used 296K, capacity 386K, committed 512K, reserved 1048576K
0.116: [GC (Allocation Failure) 14287K->14008K(15360K), 0.0011068 secs]
Heap after GC invocations=1 (full 0):
 PSYoungGen total 1536K, used 496K [0x00000000ff980000, 0x00000000ffb80000, 0x0000000100000000)
  eden space 1024K, 0% used [0x00000000ff980000,0x00000000ff980000,0x00000000ffa80000)
  from space 512K, 96% used [0x00000000ffa80000,0x00000000ffafc040,0x00000000ffb00000)
  to space 512K, 0% used [0x00000000ffb00000,0x00000000ffb00000,0x00000000ffb80000)
 ParOldGen total 13824K, used 13512K [0x00000000fec00000, 0x00000000ff980000, 0x00000000ff980000)
  object space 13824K, 97% used [0x00000000fec00000,0x00000000ff9320d0,0x00000000ff980000)
 Metaspace used 2682K, capacity 4486K, committed 4864K, reserved 1056768K
  class space used 296K, capacity 386K, committed 512K, reserved 1048576K
}
```
### -XX:+TraceClassLoading
监控加载的类，每一个类加载都记录
同样，在跟踪调试的时候可以用一下
```java
[Opened C:\Program Files\Java\jre1.8.0_161\lib\rt.jar]
[Loaded java.lang.Object from C:\Program Files\Java\jre1.8.0_161\lib\rt.jar]
[Loaded java.io.Serializable from C:\Program Files\Java\jre1.8.0_161\lib\rt.jar]
[Loaded java.lang.Comparable from C:\Program Files\Java\jre1.8.0_161\lib\rt.jar]
[Loaded java.lang.CharSequence from C:\Program Files\Java\jre1.8.0_161\lib\rt.jar]
[Loaded java.lang.String from C:\Program Files\Java\jre1.8.0_161\lib\rt.jar]
[Loaded java.lang.reflect.AnnotatedElement from C:\Program Files\Java\jre1.8.0_161\lib\rt.jar]
[Loaded java.lang.reflect.GenericDeclaration from C:\Program Files\Java\jre1.8.0_161\
...
```
### -XX:+PrintClassHistogram
控制台在运行的时候，按 Ctrl + Break 打印类的信息
windows 上好像按不出来
就是打印每个类实例的数量，使用空间
## 堆分配参数
### -Xmx -Xms
指定最大堆和最小堆
-Xmx20m -Xms5m
### -Xmn
设置新生代大小
-Xmn2m
一个 eden 加上两个 survivor 为 2m
设置新生代为 2m
```java
PS E:\study\java\jvm\jvm01> java -Xmx20m -Xms20m -Xmn2m -XX:+PrintGCDetails TestGC
[GC (Allocation Failure) [PSYoungGen: 974K->504K(1536K)] 18383K->18160K(19968K), 0.0016176 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
[GC (Allocation Failure) [PSYoungGen: 504K->496K(1536K)] 18160K->18160K(19968K), 0.0009107 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
[Full GC (Allocation Failure) [PSYoungGen: 496K->0K(1536K)] [ParOldGen: 17664K->1695K(18432K)] 18160K->1695K(19968K), [Metaspace: 2683K->2683K(1056768K)], 0.0056298 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
end
Heap
 PSYoungGen total 1536K, used 31K [0x00000000ffe00000, 0x0000000100000000, 0x0000000100000000) 
  eden space 1024K, 3% used [0x00000000ffe00000,0x00000000ffe07c68,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
  to space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
 ParOldGen total 18432K, used 4767K [0x00000000fec00000, 0x00000000ffe00000, 0x00000000ffe00000) 
  object space 18432K, 25% used [0x00000000fec00000,0x00000000ff0a7ef0,0x00000000ffe00000)
 Metaspace used 2690K, capacity 4486K, committed 4864K, reserved 1056768K
  class space used 297K, capacity 386K, committed 512K, reserved 1048576K
```
上面可以看到对象都到老年代去了，因为新生代空间不够
```java
PS E:\study\java\jvm\jvm01> java -Xmx20m -Xms20m -Xmn15m -XX:+PrintGCDetails TestGC
[GC (Allocation Failure) [PSYoungGen: 11565K->1504K(13824K)] 11565K->1736K(18944K), 0.0012016 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
end
Heap
 PSYoungGen total 13824K, used 12107K [0x00000000ff100000, 0x0000000100000000, 0x0000000100000000)
  eden space 12288K, 86% used [0x00000000ff100000,0x00000000ffb5afb8,0x00000000ffd00000)
  from space 1536K, 97% used [0x00000000ffd00000,0x00000000ffe78040,0x00000000ffe80000)
  to space 1536K, 0% used [0x00000000ffe80000,0x00000000ffe80000,0x0000000100000000)
 ParOldGen total 5120K, used 232K [0x00000000fec00000, 0x00000000ff100000, 0x00000000ff100000)
  object space 5120K, 4% used [0x00000000fec00000,0x00000000fec3a000,0x00000000ff100000)
 Metaspace used 2690K, capacity 4486K, committed 4864K, reserved 1056768K
  class space used 297K, capacity 386K, committed 512K, reserved 1048576K
```
调大新生代后，对象在新生代
### -XX:NewRatio
新生代(eden + 2s) 和老年代的比值
4 标识 新生代：老年代= 1：4，即年轻代占堆的 1/5
### -XX:SurvivorRatio
设置两个 Survivor 区 和 eden 的比
8 标识两个 Survivor:eden = 2：8 即，一个 Survivor 占年轻代的 1/10
### -XX:+HeapDumpOnOutOfMemoryError
OOM 的时候导出堆到文件
### -XX:HeapDumpPath
导出文件路径
-XX:HeapDumpPath=logs/a.dump
### -XX:OnOutOfMemoryError
在 OOM 时候，执行一个脚本
-XX:OnOutOfMemoryError="test.bat %p"
可以在 OOM 的时候发送邮件，甚至重启程序
外面套引号
测试了一下，可以执行脚本，但是 %p 进程没有传进去，在windows 上测试的
### 堆分配总结
官方推荐
新生代占堆的 3/8
幸存代占新生代的 1/10
OOM的时候记得 Dump 堆，确保可以排查现场问题
