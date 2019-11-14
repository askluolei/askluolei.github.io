---
title: sentinel 学习-下
date: 2019-11-14 22:13:34
tags:
  - java
  - sentinel
  - 学习
categories:
  - 技术笔记
---

上篇主要学习了怎么用 `sentinel` 
这篇主要简单的进入源码瞅瞅 `sentinel` 的结构  

上篇中，可以看到，限流操作的基本代码结构为 
```java
Entry entry = null;
try {
  ContextUtil.enter(target, origin);
  entry = SphU.entry(resourceName);
} catch (BlockException e) {
   // 限流处理
} finally {
  if (entry != null) {
      entry.exit();
  }
  ContextUtil.exit();
}

```

只不过，在使用的时候，基于 `aop` 获取拦截器，屏蔽掉类似这样的模板代码 
那么，这段代码里面到底发生了什么呢？  

先看整个调用图  

{% asset_img sentinel.svg This is an example image %}

## 核心类
这里面需要关注的几个类或接口

1. Context
2. Node
3. Entry
4. ProcessorSlotChain
5. ProcessorSlot


### Context  
每次调用 `entry` 获取资源
首先获取 `Context`，这是一个 `ThreadLocal` 变量，线程绑定变量

包含以下信息
1. 上下文名称 `name`
2. 当前调用树的入口节点  `entranceNode`
3. 当前处理的 `Entry`
4. 来源信息 `origin` 通常跨越系统调用才有
5. 异步表示 `async` 

一般，可以通过 `ContextUtil` 来设置 `contextName` 和 `origin`   
如果没，则使用默认的上下文  

```java
Context context = ContextUtil.getContext();
if (context instanceof NullContext) {
    // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
    // so here init the entry only. No rule checking will be done.
    return new CtEntry(resourceWrapper, null, context);
}

if (context == null) {
    // Using default context.
    context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
}
```

同一个资源可以对应不同的上下文，以此来区分不同的场景，这样，不同场景的调用可以独立统计  
每个 `contextName` 会生成一个 `EntranceNode` 挂在 `Constants.ROOT.addChild(node);` 子节点

### Node
节点，是内部的统计单元，根据在内部的作用，有 `ClusterNode` `EntranceNode` `DefaultNode`     

1. `EntranceNode`  是入口节点，通常是外部入口，譬如上面的，每个 contextName 对应的就是入口节点
2. `ClusterNode`  集群节点，其实算资源的汇总统计节点，上面说个，每个资源，根据 contextName 会独立统计，就是对应下面的 `DefaultNode`,每个资源的汇总统计，就是这个节点
3. `DefaultNode` 默认节点，上面说了

节点的统计信息将会用于后面的限流规则    
里面的统计是基于滑动窗口的，实现类为 `LeapArray`    
这是一个抽象的实现，实现滑动窗口    
```java
public LeapArray(int sampleCount, int intervalInMs) {
    AssertUtil.isTrue(sampleCount > 0, "bucket count is invalid: " + sampleCount);
    AssertUtil.isTrue(intervalInMs > 0, "total time interval of the sliding window should be positive");
    AssertUtil.isTrue(intervalInMs % sampleCount == 0, "time span needs to be evenly divided");

    this.windowLengthInMs = intervalInMs / sampleCount;
    this.intervalInMs = intervalInMs;
    this.sampleCount = sampleCount;

    this.array = new AtomicReferenceArray<>(sampleCount);
}
```

intervalInMs 代表每个窗口时间
sampleCount  代表将窗口时间划分为多少分，越多越平滑，性能也越差

`sentinel` 内部，秒级窗口,默认是 2 1000
也就是内部将1s 划分为 2 个  500 ms  统计    

具体实现有 `BucketLeapArray` `OccupiableBucketLeapArray`    

`BucketLeapArray`  为普通实现,不支持预占资源
`OccupiableBucketLeapArray`  这个在内部维护一个 `FutureBucketLeapArray` 用来记录未来资源使用情况，来允许预占资源

## ProcessorSlotChain ProcessorSlot 
处理链 和 处理插槽可以一起看，所有规则都是对应处理插槽处理的。使用 `SPI`  
```java
public interface SlotChainBuilder {

    /**
     * Build the processor slot chain.
     *
     * @return a processor slot that chain some slots together
     */
    ProcessorSlotChain build();
}
```

通过实现上面的接口，来构建处理链    
```java
public class DefaultSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();
        chain.addLast(new NodeSelectorSlot());
        chain.addLast(new ClusterBuilderSlot());
        chain.addLast(new LogSlot());
        chain.addLast(new StatisticSlot());
        chain.addLast(new AuthoritySlot());
        chain.addLast(new SystemSlot());
        chain.addLast(new FlowSlot());
        chain.addLast(new DegradeSlot());

        return chain;
    }

}
```

默认的实现，这里的 `Slot` 是限流的规则处理  


### NodeSelectorSlot
这个插槽记录调用链，设置当前的 node
```java
// node 记录统计信息，因此需要缓存
 DefaultNode node = map.get(context.getName());
    if (node == null) {
        synchronized (this) {
            node = map.get(context.getName());
            if (node == null) {
                node = new DefaultNode(resourceWrapper, null);
                HashMap<String, DefaultNode> cacheMap = new HashMap<String, DefaultNode>(map.size());
                cacheMap.putAll(map);
                cacheMap.put(context.getName(), node);
                map = cacheMap;
                // 构建调用树
                // Build invocation tree
                ((DefaultNode) context.getLastNode()).addChild(node);
            }

        }
    }
    // 设置当前节点
    context.setCurNode(node);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
```


### ClusterBuilderSlot  
构建 `ClusterNode` 这里每个资源对应一个，可以看到这里就是一个对象   
这里是本资源的统计汇总
然后根据 `origin` 构建 `OriginNode`
```java
if (clusterNode == null) {
    synchronized (lock) {
        if (clusterNode == null) {
            // Create the cluster node.
            clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
            HashMap<ResourceWrapper, ClusterNode> newMap = new HashMap<>(Math.max(clusterNodeMap.size(), 16));
            newMap.putAll(clusterNodeMap);
            newMap.put(node.getId(), clusterNode);

            clusterNodeMap = newMap;
        }
    }
}
node.setClusterNode(clusterNode);

/*
    * if context origin is set, we should get or create a new {@link Node} of
    * the specific origin.
    */
if (!"".equals(context.getOrigin())) {
    Node originNode = node.getClusterNode().getOrCreateOriginNode(context.getOrigin());
    context.getCurEntry().setOriginNode(originNode);
}

fireEntry(context, resourceWrapper, node, count, prioritized, args);
```

### StatisticSlot   
`LogSlot` 就是记录日志的.    
`StatisticSlot` 是记录统计信息，供后面的规则做决策  
主要记录通过资源的 `succ` `threadNum` `blocked` 等信息   
这些信息是在 `node` 处理

同时会触发
```java
public interface ProcessorSlotEntryCallback<T> {

    void onPass(Context context, ResourceWrapper resourceWrapper, T param, int count, Object... args) throws Exception;

    void onBlocked(BlockException ex, Context context, ResourceWrapper resourceWrapper, T param, int count, Object... args);
}
```

回调，可以通过 `StatisticSlotCallbackRegistry` 注册 


### 限流插槽
1. AuthoritySlot
   对应 `AuthorityRule` 处理黑白名单
2. SystemSlot
   对应 `SystemRule` ,根据系统整体性能指标 cpu 线程 等限流
3. FlowSlot
   对应 `FlowRule`,很多限流规则，处理策略，后面细看
4. DegradeSlot
   对应 `DegradeRule`,这里是熔断规则，满足条件，断开一段时间，然后再次开启

上面 `FlowSlot` 是复杂点，可以集群限流  


### FlowSlot    
可以根据 `FlowRule` 里面的 `clusterMode` 来判断是本地限流还是集群限流   
```java
public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                                boolean prioritized) {
    String limitApp = rule.getLimitApp();
    if (limitApp == null) {
        return true;
    }
    // 集群限流
    if (rule.isClusterMode()) {
        return passClusterCheck(rule, context, node, acquireCount, prioritized);
    }
    // 本地限流
    return passLocalCheck(rule, context, node, acquireCount, prioritized);
}
```


这里先只看本地检查  
```java
private static boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                        boolean prioritized) {
    // 选择使用哪个 Node 里面的统计信息进行决策
    // 之前说过了，有 Cluster 资源总的统计 Origin 根据来源统计 
    // 使用其他相关资源的 Cluster ,或者 根据
    Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
    if (selectedNode == null) {
        return true;
    }
    // 然后使用限流器决策，这里的限流器又有多个选择
    return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
}
```


这个的 `getRater()` 返回的是 `TrafficShapingController` 

有以下几个实现  
1. `DefaultController` 只有超过设置的阈值，直接拒绝
2. `RateLimiterController` 均匀限流，固定每个请求之间的最小间隔
3. `WarmUpController` 预热,缓慢达到预设的最大值
4. `WarmUpRateLimiterController` 看名字，里面涉及到限流算法

