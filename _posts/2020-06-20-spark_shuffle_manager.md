---
layout: post
title:  "Spark源码学习笔记（九）"
date:   2020-06-19 23:09:21 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_09.webp'
subtitle: 'ShuffleManager'
---
> 目前spark中自带ShuffleManager仅有SortShuffleManager，默认为sort，当用户指定shuffle manager为tungsen-sort，其使用的还是SortShuffleManager类

```scala
val shortShuffleMgrNames = Map(
      "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
      "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
    val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
```

> ShuffleWriter的getWriter方法会在ShuffleMapTask中被调用，SortShuffleManager根据handle类型选择对应的writer，代码如下

```scala
override def getWriter[K, V](
      handle: ShuffleHandle,
      mapId: Int,
      context: TaskContext): ShuffleWriter[K, V] = {
    numMapsForShuffle.putIfAbsent(
      handle.shuffleId, handle.asInstanceOf[BaseShuffleHandle[_, _, _]].numMaps)
    val env = SparkEnv.get
    handle match {
      case unsafeShuffleHandle: SerializedShuffleHandle[K @unchecked, V @unchecked] =>
        new UnsafeShuffleWriter(
          env.blockManager,
          shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
          context.taskMemoryManager(),
          unsafeShuffleHandle,
          mapId,
          context,
          env.conf)
      case bypassMergeSortHandle: BypassMergeSortShuffleHandle[K @unchecked, V @unchecked] =>
        new BypassMergeSortShuffleWriter(
          env.blockManager,
          shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
          bypassMergeSortHandle,
          mapId,
          context,
          env.conf)
      case other: BaseShuffleHandle[K @unchecked, V @unchecked, _] =>
        new SortShuffleWriter(shuffleBlockResolver, other, mapId, context)
    }
  }
```

> ShuffleHandle的选择策略在registerShuffle阶段就会判断抉择，
> 
> 当mapSideCombine为false（不需要map端聚合）且partitions数量不大于spark.shuffle.sort.bypassMergeThreshold值时，会采用BypassMergeSortShuffleHandle
> 
> canUseSerializedShuffle方法用于判断是否使用SerializedShuffleHandle，需要满足如下条件，不需要map端聚合，partition数量不能大于指定的阈值((2^24) -1)，Serializer支持relocation，Serializer支持relocation是指Serializer可以对已经序列化的对象进行排序，这种排序起到的效果和先对数据排序再序列化一致。支持relocation的Serializer是KryoSerializer，Spark默认使用JavaSerializer
> 
> 如果不满足BypassMergeSortShuffleHandle、SerializedShuffleHandle的要求，则使用BaseShuffleHandle

```scala
override def registerShuffle[K, V, C](
      shuffleId: Int,
      numMaps: Int,
      dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
    if (SortShuffleWriter.shouldBypassMergeSort(conf, dependency)) {
      // If there are fewer than spark.shuffle.sort.bypassMergeThreshold partitions and we don't
      // need map-side aggregation, then write numPartitions files directly and just concatenate
      // them at the end. This avoids doing serialization and deserialization twice to merge
      // together the spilled files, which would happen with the normal code path. The downside is
      // having multiple files open at a time and thus more memory allocated to buffers.
      new BypassMergeSortShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
      // Otherwise, try to buffer map outputs in a serialized form, since this is more efficient:
      new SerializedShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else {
      // Otherwise, buffer map outputs in a deserialized form:
      new BaseShuffleHandle(shuffleId, numMaps, dependency)
    }
  }
```

> 三种Writer实现了各自的write方法
 
 writer | 优点 | 缺点 | 特点 | 对应handle
---|---|---|---|---
BypassMergeSortShuffleWriter | 根据numPartitions数量生成对等的n个文件，避免了两次序列化和反序列化来合并spilled files | numPartitions数量太多会一次打开多个文件，需要更多内存分配给缓冲区 | 直接写磁盘，没有sort/spill阶段，最后会merge文件 | BypassMergeSortShuffleHandle
SortShuffleWriter | 可以map端聚合 | 无 | 需要经历sort/spill/merge，使用ExternalSorter | BaseShuffleHandle
UnsafeShuffleWriter | 操作序列化的数据，效率高 | serializer需要支持relocation才行，numPartitions大小也有限制 | 写入内存buffer中，经历sort/spill/merge，使用ShuffleExternalSorter | SerializedShuffleHandle
 

 

