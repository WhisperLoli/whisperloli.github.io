---
layout: post
title:  "Spark源码学习笔记（十）"
date:   2020-06-21 22:55:19 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_08.webp'
subtitle: '内存模型之MemoryManager'
---
> 如果想使用1.5及之前版本的内存管理，可以通过参数spark.memory.useLegacyMode控制，默认值是false，该传统模式使用的是StaticMemoryManager，传统模式将堆空间严格划分为固定大小的区域，需要注意的是，现在很多网上博客关于参数调优都会给如下几个参数，这几个参数在新版的内存管理中是无效的，除非使用的是StaticMemoryManager
> 
	spark.shuffle.memoryFraction
	spark.storage.memoryFraction
	spark.storage.unrollFraction
 
> spark目前版本的内存管理使用的是UnifiedMemoryManager，在该内存模型中，spark需要提供RESERVED\_SYSTEM\_MEMORY\_BYTES，该块空间为非存储，非执行目的留出一定数量的内存，大小是300M
> spark默认的内存比例spark.memory.fraction是0.6，存储内存比例spark.memory.storageFraction为0.5
> 
> executory总内存就是用户通过设置spark.executor.memory的内存，加入设置1G，spark代码中会获取运行时的最大可用内存Runtime.getRuntime.maxMemory，不考虑其他损耗，那么该最大可用内存也是1G，减去reserved memory需要的300M，最后分配给memory区域的只有(1024-300)*0.6=434.4M可用，storage memory占用0.5，那么最后用于spark计算的内存只有217.2M，用于storage的也只有217.2M，当内存不够时，就会溢写到磁盘中，如果代码中没有使用到大量的存储，可以调低该值
> 
![image](/blog/spark_source_code/source_code_10/memory.svg)

```scala
/**
   * Return the total amount of memory shared between execution and storage, in bytes.
   */
  private def getMaxMemory(conf: SparkConf): Long = {
    val systemMemory = conf.getLong("spark.testing.memory", Runtime.getRuntime.maxMemory)
    val reservedMemory = conf.getLong("spark.testing.reservedMemory",
      if (conf.contains("spark.testing")) 0 else RESERVED_SYSTEM_MEMORY_BYTES)
    val minSystemMemory = (reservedMemory * 1.5).ceil.toLong
    if (systemMemory < minSystemMemory) {
      throw new IllegalArgumentException(s"System memory $systemMemory must " +
        s"be at least $minSystemMemory. Please increase heap size using the --driver-memory " +
        s"option or spark.driver.memory in Spark configuration.")
    }
    // SPARK-12759 Check executor memory to fail fast if memory is insufficient
    if (conf.contains("spark.executor.memory")) {
      val executorMemory = conf.getSizeAsBytes("spark.executor.memory")
      if (executorMemory < minSystemMemory) {
        throw new IllegalArgumentException(s"Executor memory $executorMemory must be at least " +
          s"$minSystemMemory. Please increase executor memory using the " +
          s"--executor-memory option or spark.executor.memory in Spark configuration.")
      }
    }
    val usableMemory = systemMemory - reservedMemory
    val memoryFraction = conf.getDouble("spark.memory.fraction", 0.6)
    (usableMemory * memoryFraction).toLong
  }
```

> 所谓 Unified 的意思是 Storgae 和 Execution 在适当时候可以借用彼此的 Memory，需要注意的是，当 Execution 空间不足而且 Storage 空间也不足的情况下，Storage 空间如果曾经使用了超过 Unified 默认的 50% 空间的话则超过部份会被强制 drop 掉一部份数据来解决 Execution 空间不足的问题 (注意：drop 后数据会不会丢失主要是看你在程序设置的 storage_level 来决定你是 Drop 到那里，可能 Drop 到磁盘上)，这是因为执行(Execution) 比缓存 (Storage) 是更重要的事情。
> 
> Execution 向 Storage 借空间有两种情况：
> 
	第一种情况：Storage 曾经向 Execution 借了空间，它缓存的数据可能是非常的多，然后 Execution 又不需要那么大的空间 (默认情况下各占 50%)，假设现在 Storage 占了 80%，Execution 占了 20%，然后 Execution 说自己空间不足，Execution 会向内存管理器发信号把 Storgae 曾经占用的超过 50％数据的那部份强制挤掉，
	第二种情况：Execution 可以向 Storage Memory 借空间，在 Storage Memory 不足 50% 的情况下，Storgae Memory 会很乐意地把剩余空间借给 Execution。相反当 Execution 有剩余空间的时候，Storage 也可以找 Execution 借空间。

> In this context, execution memory refers to that used for computation in shuffles, joins,
sorts and aggregations, while storage memory refers to that used for caching and propagating
internal data across the cluster. There exists one MemoryManager per JVM.

> execution memory用于shuffle等计算，storage memory用于缓存数据

> yarn模式中有一个内存叫overheadmemory，这个内存也是堆外内存，当前2.3.3版本并不属于off-heap memory，是单独开辟的一块堆外内存，executor memory也不包含用户设置的off-heap memory
 

