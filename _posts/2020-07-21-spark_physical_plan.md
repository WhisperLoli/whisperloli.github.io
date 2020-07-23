---
layout: post
title:  "Spark源码学习笔记(二十一)"
date:   2020-07-21 23:32:49 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070807.webp'
subtitle: Spark SQL之物理计划
---
> 逻辑计划执行完，生成的Optimizer Logical Plan会被SparkPlanner调用plan方法执行，SparkPlaner会将各种物理计划策略strategies作用到Logical Plan上，生成Iterator[SparkPlan]，SparkPlan是物理计划的父类，所以一个逻辑计划可能产生多个物理计划，然后选取最佳的SparkPlan，看如下代码，spark2.3.3版本中仍然是用next方法选取第一个。之后在执行计划前做一些准备工作，调用若干规则应用，toRDD函数正式执行物理计划，转换成RDD[InternalRow]类型

```scala
lazy val withCachedData: LogicalPlan = {
  assertAnalyzed()
  assertSupported()
  sparkSession.sharedState.cacheManager.useCachedData(analyzed)
}

lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)

lazy val sparkPlan: SparkPlan = {
  SparkSession.setActiveSession(sparkSession)
  // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
  //       but we will implement to choose the best plan.
  planner.plan(ReturnAnswer(optimizedPlan)).next()
}

// executedPlan should not be used to initialize any SparkPlan. It should be
// only used for execution.
lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)

/** Internal version of the RDD. Avoids copies and has no schema */
lazy val toRdd: RDD[InternalRow] = executedPlan.execute()

/**
 * Prepares a planned [[SparkPlan]] for execution by inserting shuffle operations and internal
 * row format conversions as needed.
 */
protected def prepareForExecution(plan: SparkPlan): SparkPlan = {
  preparations.foldLeft(plan) { case (sp, rule) => rule.apply(sp) }
}

/** A sequence of rules that will be applied in order to the physical plan before execution. */
protected def preparations: Seq[Rule[SparkPlan]] = Seq(
  python.ExtractPythonUDFs,
  PlanSubqueries(sparkSession),
  EnsureRequirements(sparkSession.sessionState.conf),
  CollapseCodegenStages(sparkSession.sessionState.conf),
  ReuseExchange(sparkSession.sessionState.conf),
  ReuseSubquery(sparkSession.sessionState.conf))
```

> 在生成物理计划时，需要对子节点进行分区和排序，每个子类会有自己的实现，UnspecifiedDistribution就是未指定分布，无需确定数据之间的位置关系

```scala
/**
   * Specifies the data distribution requirements of all the children for this operator. By default
   * it's [[UnspecifiedDistribution]] for each child, which means each child can have any
   * distribution.
   *
   * If an operator overwrites this method, and specifies distribution requirements(excluding
   * [[UnspecifiedDistribution]] and [[BroadcastDistribution]]) for more than one child, Spark
   * guarantees that the outputs of these children will have same number of partitions, so that the
   * operator can safely zip partitions of these children's result RDDs. Some operators can leverage
   * this guarantee to satisfy some interesting requirement, e.g., non-broadcast joins can specify
   * HashClusteredDistribution(a,b) for its left child, and specify HashClusteredDistribution(c,d)
   * for its right child, then it's guaranteed that left and right child are co-partitioned by
   * a,b/c,d, which means tuples of same value are in the partitions of same index, e.g.,
   * (a=1,b=2) and (c=1,d=2) are both in the second partition of left and right child.
   */
  def requiredChildDistribution: Seq[Distribution] =
    Seq.fill(children.size)(UnspecifiedDistribution)
    
/** Specifies sort order for each partition requirements on the input data for this operator. */
  def requiredChildOrdering: Seq[Seq[SortOrder]] = Seq.fill(children.size)(Nil)
```

> 过滤算子FilterExec与列剪裁算子ProjectExec分区和排序的方式仍然沿用其子节点的方式，不对RDD的分区和排序进行任何的重新操作
> 
> 转换逻辑算子到物理算子的plan方法实现在QueryPlanner类中，SparkStrategies类继承自QueryPlaner，但不提供任何方法，仅提供策略的实现，如下代码为逻辑计划转换成物理计划的实现，prunePlans方法用于过滤差的物理计划，但是当前版本2.3.3中还未实现

```scala
  def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
    // Obviously a lot to do here still...

    // Collect physical plan candidates.
    val candidates = strategies.iterator.flatMap(_(plan))

    // The candidates may contain placeholders marked as [[planLater]],
    // so try to replace them by their child plans.
    val plans = candidates.flatMap { candidate =>
      val placeholders = collectPlaceholders(candidate)

      if (placeholders.isEmpty) {
        // Take the candidate as is because it does not contain placeholders.
        Iterator(candidate)
      } else {
        // Plan the logical plan marked as [[planLater]] and replace the placeholders.
        placeholders.iterator.foldLeft(Iterator(candidate)) {
          case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
            // Plan the logical plan for the placeholder.
            val childPlans = this.plan(logicalPlan)

            candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
              childPlans.map { childPlan =>
                // Replace the placeholder by the child plan
                candidateWithPlaceholders.transformUp {
                  case p if p == placeholder => childPlan
                }
              }
            }
        }
      }
    }

    val pruned = prunePlans(plans)
    assert(pruned.hasNext, s"No plan for $plan")
    pruned
  }
  
  override protected def prunePlans(plans: Iterator[SparkPlan]): Iterator[SparkPlan] = {
    // TODO: We will need to prune bad plans when we improve plan space exploration
    //       to prevent combinatorial explosion.
    plans
  }
```

> 到RDD的过程仍然需要使用如下规则，CollapseCodegenStages用于代码生成相关

```scala
/** A sequence of rules that will be applied in order to the physical plan before execution. */
  protected def preparations: Seq[Rule[SparkPlan]] = Seq(
  	 // 提取Python中的UDF函数
    python.ExtractPythonUDFs,
    // 特殊子查询物理计划处理
    PlanSubqueries(sparkSession),
    // 确保分区排序正确性
    EnsureRequirements(sparkSession.sessionState.conf),
    // 代码生成
    CollapseCodegenStages(sparkSession.sessionState.conf),
    // Exchange节点重用
    ReuseExchange(sparkSession.sessionState.conf),
    // 子查询重用
    ReuseSubquery(sparkSession.sessionState.conf))
```