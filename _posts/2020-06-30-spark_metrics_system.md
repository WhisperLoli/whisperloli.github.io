---
layout: post
title:  "Spark源码学习笔记（十二）"
date:   2020-06-30 21:47:13 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_06.webp'
subtitle: 'MetricsSystem'
---

> 在SparkEnv中会为driver，executor创建MetricsSystem对象，之后调用该对象的start方法

```scala
def start() {
  require(!running, "Attempting to start a MetricsSystem that is already running")
  running = true
  StaticSources.allSources.foreach(registerSource)
  registerSources()
  registerSinks()
  sinks.foreach(_.start)
}
```

> start方法会注册source和sink

```scala
private[spark] object StaticSources {
  /**
   * The set of all static sources. These sources may be reported to from any class, including
   * static classes, without requiring reference to a SparkEnv.
   */
  val allSources = Seq(CodegenMetrics, HiveCatalogMetrics)
}
```

> StaticSource有CodegenMetrics、HiveCatalogMetrics，前者用于编译时间、字节码大小、生成的方法大小等统计指标，后者用于Hive相关的指标统计
> 
> 该统计指标会被加入到MetricsSystem中的统计指标中，相当于metrics里面嵌套着一层metrics，使用buildRegistryName生成metrics的名称

```scala
private[spark] def buildRegistryName(source: Source): String = {
    val metricsNamespace = conf.get(METRICS_NAMESPACE).orElse(conf.getOption("spark.app.id"))

    val executorId = conf.getOption("spark.executor.id")
    val defaultName = MetricRegistry.name(source.sourceName)

    if (instance == "driver" || instance == "executor") {
      if (metricsNamespace.isDefined && executorId.isDefined) {
        MetricRegistry.name(metricsNamespace.get, executorId.get, source.sourceName)
      } else {
        // Only Driver and Executor set spark.app.id and spark.executor.id.
        // Other instance types, e.g. Master and Worker, are not related to a specific application.
        if (metricsNamespace.isEmpty) {
          logWarning(s"Using default name $defaultName for source because neither " +
            s"${METRICS_NAMESPACE.key} nor spark.app.id is set.")
        }
        if (executorId.isEmpty) {
          logWarning(s"Using default name $defaultName for source because spark.executor.id is " +
            s"not set.")
        }
        defaultName
      }
    } else { defaultName }
  }
```
> 
> registerSources会从metricsConfig配置中获取相关的metrics配置，并使用registerSource添加到metrics中，registerSinks稍微有点区别，会提供一个MetricsServlet，这个东西用于web ui，而如果是sink的话，添加到ArrayBuffer中，随后遍历ArrayBuffer，调用每个sink的start方法

```scala
private def registerSources() {
    val instConfig = metricsConfig.getInstance(instance)
    val sourceConfigs = metricsConfig.subProperties(instConfig, MetricsSystem.SOURCE_REGEX)

    // Register all the sources related to instance
    sourceConfigs.foreach { kv =>
      val classPath = kv._2.getProperty("class")
      try {
        val source = Utils.classForName(classPath).newInstance()
        registerSource(source.asInstanceOf[Source])
      } catch {
        case e: Exception => logError("Source class " + classPath + " cannot be instantiated", e)
      }
    }
  }

  private def registerSinks() {
    val instConfig = metricsConfig.getInstance(instance)
    val sinkConfigs = metricsConfig.subProperties(instConfig, MetricsSystem.SINK_REGEX)

    sinkConfigs.foreach { kv =>
      val classPath = kv._2.getProperty("class")
      if (null != classPath) {
        try {
          val sink = Utils.classForName(classPath)
            .getConstructor(classOf[Properties], classOf[MetricRegistry], classOf[SecurityManager])
            .newInstance(kv._2, registry, securityMgr)
          if (kv._1 == "servlet") {
            metricsServlet = Some(sink.asInstanceOf[MetricsServlet])
          } else {
            sinks += sink.asInstanceOf[Sink]
          }
        } catch {
          case e: Exception =>
            logError("Sink class " + classPath + " cannot be instantiated")
            throw e
        }
      }
    }
  }
```
