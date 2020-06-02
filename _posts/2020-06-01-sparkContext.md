---
layout: post
title:  "Spark源码学习笔记（四）"
date:   2020-06-01 23:17:34 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_14.webp'
subtitle: 'SparkContext研究'
---
> 开始正式进入spark源码研究阶段了，这段时间也一直在研究这块东西，主要是里面的类、特质太多，各种跳转吧，比较费时间，然后最后也没明白里面各个细节。想到个策略，还是先粗粒度的学习整个流程，之后再深入研究各个细分类。暂时先不研究SparkContext中的方法，SparkContext文件中将近3000行代码，不可能都研究一遍，先大致了解流程吧，等以后再慢慢了解。
> 
> 分享下我在研究源码中的策略
> 
> 	1. 有些方法看不懂代码，就直接写一个主函数，调用一下这个方法，看输入输出大致就能明白方法的功能。
> 	2. debug，直接new SparkContext，进入单步调试模式，巧妙设置断点，看代码是如何运行的
> 
> spark源码学习中会涉及到metrics(监控工具)，Future/Await（Scala异步编程特性）等知识点，参考[github demo](https://github.com/WhisperLoli/spark-learing)
> 
> 进入正题，差不多在SparkContext.scala文件362行附近，之后这一大块都是对变量赋值，首先克隆config给_conf对象，之后基于_conf对象操作，进行校验，较验完设置其他参数，比较重要的就是生成_listenerBus，_statusStore等监听对象。
> 
> 在这之后使用SparkEnv伴生对象的createDriverEnv方法创建SparkEnv对象，这个方法里面创建了各种Spark需要的对象，目前就不细看里面的类，讲一下各个变量吧。
> 
> rpcEnv：RpcEnv的实例，根据名字就能知道是rpc环境，其内部可以启动NettyRpc server，后续的通信都会使用到他
> 
> broadcastManager：Broadcast管理器，内部可以创建broadcast，broadcast创建底层使用的是TorrentBroadcastFactory创建TorrentBroadcast，也十分复杂
> 
> mapOutputTracker：keeps track of the location of the map output of a stage
> 
> shuffleManager：shuffle管理器，默认使用的是SortShuffleManager
> 
> memoryManager：内存管理器，默认使用的是UnifiedMemoryManager，官方不建议使用StaticMemoryManager
> 
> blockTransferService：用于拉数据操作，uses Netty to fetch a set of blocks at time

> blockManager：block管理器，用于将块缓存到各种存储（内存，磁盘和堆外）中
> 
> metricsSystem：监控指标系统，例如spark ui中看到的各种指标都是从这里获取
> 
> outputCommitCoordinator：提交任务到hdfs
> 
> 根据这些变量，创建SparkEnv对象，真的超复杂
> 
> 接着回到SparkContext中，会继续创建各种对象，例如ui，hadoopConfiguration等等，_heartbeatReceiver用于心跳检测，分布式系统中都会有存在心跳检查
> taskScheduler，dagScheduler也会在SparkContext中创建，我们在web ui中看见的application id、AttemptId也是由taskScheduler生成
> 
> 如果开启了动态生成Executor，则ExecutorAllocationManager会根据资源情况自动添加移除
> 最后还会由ShutdownHookManager添加个shutdownHook，猜都能知道shutdownHook内部肯定是关闭各种资源，清理生成的临时文件
> 至此，SparkContext初始化完成，看了好久，涉及的东西非常多，加油！