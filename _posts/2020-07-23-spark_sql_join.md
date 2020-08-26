---
layout: post
title:  "Spark源码学习笔记(二十二)"
date:   2020-07-23 22:53:49 +0800
tags:
- Spark
- Spark SQL
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070806.webp'
subtitle: Spark SQL之Join
---

> 目前转换成为物理计划的join策略实现总共有5种，
> 
> ```
> Broadcast hash join (BHJ):不支持full outer join；right outer join只能广播左表，left outer, left semi, left anti只能广播右表；inner join左右表都可以广播。用户可以使用广播函数显示指定或者设置广播阈值大小，默认10M
> Shuffle hash join: 比BHJ更严格
> Sort merge: 通常不能使用上述方案时，考虑使用，当两个表都比较大时
> BroadcastNestedLoopJoin (BNLJ)，
> CartesianProduct，
> 最后两种都是对应着没有指定join key的情况，非常耗性能
> ```

```scala
   /**
     * Matches a plan whose output should be small enough to be used in broadcast join.
     */
    // 判断是否能够广播，表大小统计小于广播阈值
    private def canBroadcast(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold
    }

    /**
     * Matches a plan whose single partition should be small enough to build a hash table.
     *
     * Note: this assume that the number of partition is fixed, requires additional work if it's
     * dynamic.
     */
    // 构建本地HashMap，需要表大小统计小于广播阈值(spark.sql.autoBroadcastJoinThreshold)*默认分区数(spark.sql.shuffle.partitions),按照默认的算，就是10*200，大小为2000M
    private def canBuildLocalHashMap(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
    }

    /**
     * Returns whether plan a is much smaller (3X) than plan b.
     *
     * The cost to build hash map is higher than sorting, we should only build hash map on a table
     * that is much smaller than other one. Since we does not have the statistic for number of rows,
     * use the size of bytes here as estimation.
     */
    // a表大小扩大3倍要比b小，也就是b远大于a表
    private def muchSmaller(a: LogicalPlan, b: LogicalPlan): Boolean = {
      a.stats.sizeInBytes * 3 <= b.stats.sizeInBytes
    }
	
	// innerlike, leftouter,leftsemi,leftanti,ExistenceJoin可以构建右表
    private def canBuildRight(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | LeftOuter | LeftSemi | LeftAnti | _: ExistenceJoin => true
      case _ => false
    }

    // innerlike, RightOuter可以构建左表
    private def canBuildLeft(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | RightOuter => true
      case _ => false
    }

    private def broadcastSide(
        canBuildLeft: Boolean,
        canBuildRight: Boolean,
        left: LogicalPlan,
        right: LogicalPlan): BuildSide = {
		// 那边小build哪边的表
      def smallerSide =
        if (right.stats.sizeInBytes <= left.stats.sizeInBytes) BuildRight else BuildLeft
		// 左右都能build时，build小的一边
      if (canBuildRight && canBuildLeft) {
        // Broadcast smaller side base on its estimated physical size
        // if both sides have broadcast hint
        smallerSide
      } else if (canBuildRight) {
        BuildRight
      } else if (canBuildLeft) {
        BuildLeft
      } else {
      	 // 左右表都不能build时，build小的表
        // for the last default broadcast nested loop join
        smallerSide
      }
    }

    private def canBroadcastByHints(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : Boolean = {
      val buildLeft = canBuildLeft(joinType) && left.stats.hints.broadcast
      val buildRight = canBuildRight(joinType) && right.stats.hints.broadcast
      buildLeft || buildRight
    }

    private def broadcastSideByHints(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : BuildSide = {
      val buildLeft = canBuildLeft(joinType) && left.stats.hints.broadcast
      val buildRight = canBuildRight(joinType) && right.stats.hints.broadcast
      broadcastSide(buildLeft, buildRight, left, right)
    }

    private def canBroadcastBySizes(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : Boolean = {
      val buildLeft = canBuildLeft(joinType) && canBroadcast(left)
      val buildRight = canBuildRight(joinType) && canBroadcast(right)
      buildLeft || buildRight
    }

    private def broadcastSideBySizes(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : BuildSide = {
      val buildLeft = canBuildLeft(joinType) && canBroadcast(left)
      val buildRight = canBuildRight(joinType) && canBroadcast(right)
      broadcastSide(buildLeft, buildRight, left, right)
    }

    def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {

      // --- BroadcastHashJoin --------------------------------------------------------------------
		// 显示指定广播时
      // broadcast hints were specified
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastByHints(joinType, left, right) =>
        val buildSide = broadcastSideByHints(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))

		// 不显示指定广播时，只要满足条件也能广播
      // broadcast hints were not specified, so need to infer it from size and configuration.
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))

      // --- ShuffledHashJoin ---------------------------------------------------------------------
		// 需要设置spark.sql.join.preferSortMergeJoin为false,默认为true，参与join的key不具有排序的特性
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildRight(joinType) && canBuildLocalHashMap(right)
           && muchSmaller(right, left) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildRight, condition, planLater(left), planLater(right)))

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildLeft(joinType) && canBuildLocalHashMap(left)
           && muchSmaller(left, right) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildLeft, condition, planLater(left), planLater(right)))

      // --- SortMergeJoin ------------------------------------------------------------

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if RowOrdering.isOrderable(leftKeys) =>
        joins.SortMergeJoinExec(
          leftKeys, rightKeys, joinType, condition, planLater(left), planLater(right)) :: Nil

      // --- Without joining keys ------------------------------------------------------------

      // Pick BroadcastNestedLoopJoin if one side could be broadcast
      case j @ logical.Join(left, right, joinType, condition)
          if canBroadcastByHints(joinType, left, right) =>
        val buildSide = broadcastSideByHints(joinType, left, right)
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      case j @ logical.Join(left, right, joinType, condition)
          if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      // Pick CartesianProduct for InnerJoin
      case logical.Join(left, right, _: InnerLike, condition) =>
        joins.CartesianProductExec(planLater(left), planLater(right), condition) :: Nil

      case logical.Join(left, right, joinType, condition) =>
        val buildSide = broadcastSide(
          left.stats.hints.broadcast, right.stats.hints.broadcast, left, right)
        // This join could be very slow or OOM
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      // --- Cases where this strategy does not apply ---------------------------------------------

      case _ => Nil
    }

```

> hash join需要满足如下条件
> 
> ```
> 1. 设置spark.sql.join.preferSortMergeJoin=false
> 2. join key不具有排序特性
> 3. 右表/左表能被build，且右表/左表能build localhashmap，也就是右表/左表大小小于广播阈值*默认分区数
> 4. 右表/左表比左表/右表小3倍以上
> ```
> 
> broadcast join条件
> 
> ```
> 表大小不超过广播阈值，且能build，比如右连接，条件时需要build左表且左表大小比阈值小。
> 右连接build左表，左连接build右表，内连接左右表都可以
> ```
> 
> spark中broadcast join, hash join的实现都是把一张表加载到内存中，称为buildIter，另一张表称之为streamIter，join时从streamIter中依次取一条记录，然后去buildIter中查，满足条件则join

```scala
protected override def doExecute(): RDD[InternalRow] = {
    val numOutputRows = longMetric("numOutputRows")
    streamedPlan.execute().zipPartitions(buildPlan.execute()) { (streamIter, buildIter) =>
      val hashed = buildHashedRelation(buildIter)
      join(streamIter, hashed, numOutputRows)
    }
}
  
private def buildHashedRelation(iter: Iterator[InternalRow]): HashedRelation = {
    val buildDataSize = longMetric("buildDataSize")
    val buildTime = longMetric("buildTime")
    val start = System.nanoTime()
    val context = TaskContext.get()
    val relation = HashedRelation(iter, buildKeys, taskMemoryManager = context.taskMemoryManager())
    buildTime += (System.nanoTime() - start) / 1000000
    buildDataSize += relation.estimatedSize
    // This relation is usually used until the end of task.
    context.addTaskCompletionListener(_ => relation.close())
    relation
}
```

> sort merge join需要先对key shuffle操作，保证key值相同的记录会被分在相应的分区，分区后对每个分区内的数据进行排序，排序后再对相应的分区内的记录进行连接，不需要将一个表加载到内存中，所以sort merge join并没有buildIter,streamIter概念。两个序列都是有序的，从头遍历，碰到key相同的就输出，如果不同，左边小就继续取左边，反之取右边