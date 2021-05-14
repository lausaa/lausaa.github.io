---
layout: post
title: "HDFS RPC 之callQueue"
date: 2021-05-13
tags: [HDFS, 分布式存储]
categories: "202105"
comments: true
author: lausaa
---

此篇以前一篇[网络通信之RPC](https://lausaa.github.io/2021/05/12/HadoopRPC/) 为背景。

我们已经了解了Reader 线程构造完指令后会推到指令队列，然后Handler 线程会从指令队列里拿指令任务去执行。不难想象，这个指令队列可能会是一个性能瓶颈。如果指令队列仅仅是单个queue 的话，那么将引发严重的竞争（单个client 提交了一大批调用指令，其他client 可能饿死）。

解决问题也不难，一个不行就上多个，多个还不够好就搞优先级。对此，HADOOP RPC 祭出了FairCallQueue。如下图：  
![](/imgs/faircallqueue-overview.png)

在RPC Server 中，构造了一个CallQueueManager，其中会创建一个调度器和一个阻塞队列，调度器和阻塞队列可以有不同的实现形式，分别通过`scheduler.impl` 和`callqueue.impl` 参数指定，如下：
```
    // 默认 DecayRpcScheduler
    this.scheduler = createScheduler(schedulerClass, priorityLevels,
        namespace, conf);

    // 默认 FairCallQueue
    BlockingQueue<E> bq = createCallQueueInstance(backingClass,
        priorityLevels, maxQueueSize, namespace, capacityWeights, conf);
```

## FairCallQueue
FairCallQueue 对象中包含了若干个阻塞队列，队列数量与优先级数量相当。  
当Handler 要从队列取出call 时，会通过FairCallQueue 中维护的一个权重管理器来确定具体从那个队列**取出**。  
FairCallQueue 主要初始化如下：
```
this.queues = new ArrayList<BlockingQueue<E>>(numQueues);

this.multiplexer = new WeightedRoundRobinMultiplexer(numQueues, ns, conf);

相关参数：
faircallqueue.priority-levels: 队列优先级数量，默认与调度优先级数量相同
scheduler.priority.levels: 调度优先级数量，默认4(0,1,2,3), 0 为最高优先级

faircallqueue.multiplexer.weights: 默认为8,4,2,1，即最高优先级队列取8个请求，第二优先级4个，以此类推。
```

## DecayRpcScheduler
当Reader 收到一个call 请求后，会通过DecayRpcScheduler 获取优先级，然后**加入**到对应的阻塞队列。  
优先级的计算主要来自用户call 的计数。  
DecayRpcScheduler 维护着每个用户的请求计数。这个计数随时间逐渐减少。一个定时任务 **DecayTask** 在每个周期将每个用户的请求计数乘以衰减系数。这样就维护了每个用户请求计数的加权/滚动平均值，以使用户请求优先级控制更加平滑。  
同时，DecayTask 在每次执行时，会重新计算所有已知用户的优先级，call 请求占总数的50％ 以上的用户置于最低优先级，占25％~50％ 在第二低优先级，占12.5％~25％ 在第二高的优先级，所有其他用户被排在最高优先级。  
在扫描结束时，每个已知用户都有一个缓存的优先级，该优先级将一直使用到下一次扫描。两次扫描之间出现的新用户将即时计算其优先级。  
此外，还有一个回退机制，即队列已满或者请求指令执行时间过长的情况，已经收到的call 请求将被以向client 抛异常的方式回退，client 通常等待一段时间再重试。  
**shouldBackOff** 方法用于判断是否应该回退。  

DecayTask 执行如下：
```
decayCurrentCosts()
-> long nextValue = (long) (currentValue * decayFactor);
-> recomputeScheduleCache();
    -> computePriorityLevel(snapshot, id);
-> updateAverageResponseTime(true); 

相关参数：
decay-scheduler.period-ms: DecayTask 执行周期，默认 5s
decay-scheduler.decay-factor: 衰减系数，默认 0.5
decay-scheduler.thresholds: 优先级阈值，默认 0.125,0.25,0.5
decay-scheduler.backoff.responsetime.thresholds: 回退时间阈值，默认0优先级10s, 1-20s, 2-30s, 3-40s
```

队列拥塞控制是无解的，所有的方案都是权衡的结果，FairCallQueue 作为当前HADOOP RPC 的拥塞控制模块，已经很大程度上解决了性能瓶颈。