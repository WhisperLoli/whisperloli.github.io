---
layout: post
title:  "Spark源码学习笔记(二十三)"
date:   2020-07-28 22:12:49 +0800
tags:
- Spark
- Spark SQL
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070808.webp'
subtitle: Spark SQL之CBO优化
---

> CBO是相对应于RBO的概念，RBO就是应用了一系列的规则到逻辑计划中。CBO是基于代价计算，会考虑到数据的统计、分布等等情况。
> 
> CBO优化join reorder，也就是有多个表join，cbo优化会按特定的顺序进行join，仅支持InnerLike类型的join，目前只有inner join和cross join属于InnerLike类型

```scala
/**
 * The explicitCartesian flag indicates if the inner join was constructed with a CROSS join
 * indicating a cartesian product has been explicitly requested.
 */
sealed abstract class InnerLike extends JoinType {
  def explicitCartesian: Boolean
}

case object Inner extends InnerLike {
  override def explicitCartesian: Boolean = false
  override def sql: String = "INNER"
}

case object Cross extends InnerLike {
  override def explicitCartesian: Boolean = true
  override def sql: String = "CROSS"
}
```

> join reorder需要开启参数spark.sql.cbo.joinReorder.enabled与spark.sql.cbo.enabled
> 
> 多表连接顺序优化算法使用了动态规划寻找最优join顺序
> 优势：动态规划算法能够求得整个搜索空间中最优解
> 缺陷：当联接表数量增加时，算法需要搜索的空间增加的非常快 ，计算最优联接顺序代价很高
> 
> 计划优劣性计算公式如下面代码中，代码中已经给出了注释
> 

```scala
 /**
   * Partial join order in a specific level.
   *
   * @param itemIds Set of item ids participating in this partial plan.
   * @param plan The plan tree with the lowest cost for these items found so far.
   * @param joinConds Join conditions included in the plan.
   * @param planCost The cost of this plan tree is the sum of costs of all intermediate joins.
   */
  case class JoinPlan(
      itemIds: Set[Int],
      plan: LogicalPlan,
      joinConds: Set[Expression],
      planCost: Cost) {

    /** Get the cost of the root node of this plan tree. */
    def rootCost(conf: SQLConf): Cost = {
      if (itemIds.size > 1) {
        val rootStats = plan.stats
        Cost(rootStats.rowCount.get, rootStats.sizeInBytes)
      } else {
        // If the plan is a leaf item, it has zero cost.
        Cost(0, 0)
      }
    }

    def betterThan(other: JoinPlan, conf: SQLConf): Boolean = {
      // card为结果行数，size为结果大小
      if (other.planCost.card == 0 || other.planCost.size == 0) {
        false
      } else {
        // 当前结果集行数/其他计划结果集行数
        val relativeRows = BigDecimal(this.planCost.card) / BigDecimal(other.planCost.card)
        // 当前结果集空间大小/其他计划结果集大小
        val relativeSize = BigDecimal(this.planCost.size) / BigDecimal(other.planCost.size)
        // joinReorderCardWeight的值为参数spark.sql.cbo.joinReorder.card.weight的值，默认0.7，所以当前结果集条目越少，空间越小，会更优
        relativeRows * conf.joinReorderCardWeight +
          relativeSize * (1 - conf.joinReorderCardWeight) < 1
      }
    }
  }
```

> 上一章中讲了broadcast join会严格要求join顺序，但是inner join 左右表都可以build，reoder仅支持InnerLike join，所以同时使用join reorder和broadcast join无需注意表顺序，reorder会自动调整。
> 
> 当然，CBO优化还有其他的地方，比如两个表Join，A表过滤前1亿数据，过滤后10W数据，B表1000万数据，默认会使用B表左右build side，开启CBO优化后，会使用A表作为build side，如果A表足够小，满足broadcast join，则会采用broadcast join
> 
> 总结下CBO优化就是如下三点：
> 
> 	 1. 选择合适的build side
>   2. 优化join类型
>   3. join reoder，优化多表join顺序