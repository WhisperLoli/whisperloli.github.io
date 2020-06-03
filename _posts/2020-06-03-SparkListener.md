---
layout: post
title:  "Spark源码学习笔记（五）"
date:   2020-06-03 22:40:21 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_13.webp'
subtitle: 'Spark事件监听器'
---
> 之前已经粗略的研究SparkContext启动的整个流程，那么现在细细的研究下里面使用到的监听相关类
> 
> LiveListenerBus，该类主要用于监听SparkListenerEvent事件
> 
> 先讲下相关联的类吧，本来想画类图的，偷个懒，哈哈
> 
> MetricRegistry，没啥好说的，就是做个监控，比如总共产生了多少个event的计数之类
> 
> AsyncEventQueue继承自SparkListenerBus，类中eventQueue采用了阻塞队列，使用metrics获取当前队列中有多少个事件，总共shared、appStatus、executorManagement、eventLog四种类型的异步事件队列，分别用于存放对应的监听事件
> 
> SparkListenerBus继承自ListenerBus
> 
> ListenrBuss主要用于添加监听，及向所有已注册的监听发送event
> 
> SparkListenerEvent，是所有其他事件的父类，子类诸如SparkListenerStageCompleted，SparkListenerTaskStart，SparkListenerUnpersistRDD等等，子类含括了所有的监听事件
> 
> SparkListenerInterface，监听事件接口
> 
> SparkListener，继承自SparkListenerInterface，实现当某个event发生时，listener应当做什么操作
> 
> SparkFirehoseListener，同SparkListener类似，实现了SparkListenerInterface中的方法
> 
> 在SparkContext初始化中，会执行setupAndStartListenerBus()方法，该方法中会调用LiveListenerBus中的start()方法，用于启动LiveListenerBus
> 

 ```scala
 private def setupAndStartListenerBus(): Unit = {
    try {
      conf.get(EXTRA_LISTENERS).foreach { classNames =>
        val listeners = Utils.loadExtensions(classOf[SparkListenerInterface], classNames, conf)
        listeners.foreach { listener =>
          listenerBus.addToSharedQueue(listener)
          logInfo(s"Registered listener ${listener.getClass().getName()}")
        }
      }
    } catch {
      case e: Exception =>
        try {
          stop()
        } finally {
          throw new SparkException(s"Exception when registering SparkListener", e)
        }
    }

    listenerBus.start(this, _env.metricsSystem)
    _listenerBusStarted = true
  }  
 ```
 
 > queues采用java中线程安全的并发类CopyOnWriteArrayList存储AsyncEventQueue，LiveListenerBus中的start()方法如下，会调用每个AsyncEventQueue对象的start方法，暂时喊AsyncEventQueue为异步监听事件队列吧，遍历所有的监听事件queuedEvents，并将queuedEvents存储的所有监听事件发送到异步监听事件队列AsyncEventQueue中，LiveListenerBus中addToQueue方法则是添加指定的监听器到特定的队列AsyncEventQueue中，如果特定的队列不存在则新建

```scala
 private val queues = new CopyOnWriteArrayList[AsyncEventQueue]()
 
 // LiveListenerBus中的start方法
 def start(sc: SparkContext, metricsSystem: MetricsSystem): Unit = synchronized {
    if (!started.compareAndSet(false, true)) {
      throw new IllegalStateException("LiveListenerBus already started.")
    }

    this.sparkContext = sc
    queues.asScala.foreach { q =>
      q.start(sc)
      queuedEvents.foreach(q.post)
    }
    queuedEvents = null
    metricsSystem.registerSource(metrics)
  } 
  
 /**
   * Add a listener to a specific queue, creating a new queue if needed. Queues are independent
   * of each other (each one uses a separate thread for delivering events), allowing slower
   * listeners to be somewhat isolated from others.
   */
private[spark] def addToQueue(
      listener: SparkListenerInterface,
      queue: String): Unit = synchronized {
    if (stopped.get()) {
      throw new IllegalStateException("LiveListenerBus is stopped.")
    }

    queues.asScala.find(_.name == queue) match {
      case Some(queue) =>
        queue.addListener(listener)

      case None =>
        val newQueue = new AsyncEventQueue(queue, conf, metrics, this)
        newQueue.addListener(listener)
        if (started.get()) {
          newQueue.start(sparkContext)
        }
        queues.add(newQueue)
    }
  }
  
```

> AsyncEventQueue中的start方法会启动一个后台监听线程，监听线程会不断的从监听事件队列eventQueue中获取event，然后执行ListenerBus的postToAll()操作，postToAll会把传来监听事件event发送给每一个注册的监听器SparkListenerInterface，监听器根据监听到的事件再执行相应的操作，AsyncEventQueue方法就是将传进来的监听事件event写入监听事件队列中，如果队列满了，则丢弃

```scala
 private val dispatchThread = new Thread(s"spark-listener-group-$name") {
    setDaemon(true)
    override def run(): Unit = Utils.tryOrStopSparkContext(sc) {
      dispatch()
    }
  }

  private def dispatch(): Unit = LiveListenerBus.withinListenerThread.withValue(true) {
    var next: SparkListenerEvent = eventQueue.take()
    while (next != POISON_PILL) {
      val ctx = processingTime.time()
      try {
        super.postToAll(next)
      } finally {
        ctx.stop()
      }
      eventCount.decrementAndGet()
      next = eventQueue.take()
    }
    eventCount.decrementAndGet()
  }

  // AsyncEventQueue中的start方法
  private[scheduler] def start(sc: SparkContext): Unit = {
    if (started.compareAndSet(false, true)) {
      this.sc = sc
      dispatchThread.start()
    } else {
      throw new IllegalStateException(s"$name already started!")
    }
  }
  
  // AsyncEventQueue中的post方法
  def post(event: SparkListenerEvent): Unit = {
    if (stopped.get()) {
      return
    }

    eventCount.incrementAndGet()
    if (eventQueue.offer(event)) {
      return
    }

   eventCount.decrementAndGet()
    droppedEvents.inc()
    droppedEventsCounter.incrementAndGet()
    if (logDroppedEvent.compareAndSet(false, true)) {
      // Only log the following message once to avoid duplicated annoying logs.
      logError(s"Dropping event from queue $name. " +
        "This likely means one of the listeners is too slow and cannot keep up with " +
        "the rate at which tasks are being started by the scheduler.")
    }
    logTrace(s"Dropping event $event")

    val droppedCount = droppedEventsCounter.get
    if (droppedCount > 0) {
      // Don't log too frequently
      if (System.currentTimeMillis() - lastReportTimestamp >= 60 * 1000) {
        // There may be multiple threads trying to decrease droppedEventsCounter.
        // Use "compareAndSet" to make sure only one thread can win.
        // And if another thread is increasing droppedEventsCounter, "compareAndSet" will fail and
        // then that thread will update it.
        if (droppedEventsCounter.compareAndSet(droppedCount, 0)) {
          val prevLastReportTimestamp = lastReportTimestamp
          lastReportTimestamp = System.currentTimeMillis()
          val previous = new java.util.Date(prevLastReportTimestamp)
          logWarning(s"Dropped $droppedCount events from $name since $previous.")
        }
      }
    }
  }

  
// ListenerBus中postToAll方法  
def postToAll(event: E): Unit = {
    // JavaConverters can create a JIterableWrapper if we use asScala.
    // However, this method will be called frequently. To avoid the wrapper cost, here we use
    // Java Iterator directly.
    val iter = listenersPlusTimers.iterator
    while (iter.hasNext) {
      val listenerAndMaybeTimer = iter.next()
      val listener = listenerAndMaybeTimer._1
      val maybeTimer = listenerAndMaybeTimer._2
      val maybeTimerContext = if (maybeTimer.isDefined) {
        maybeTimer.get.time()
      } else {
        null
      }
      try {
        doPostEvent(listener, event)
        if (Thread.interrupted()) {
          // We want to throw the InterruptedException right away so we can associate the interrupt
          // with this listener, as opposed to waiting for a queue.take() etc. to detect it.
          throw new InterruptedException()
        }
      } catch {
        case ie: InterruptedException =>
          logError(s"Interrupted while posting to ${Utils.getFormattedClassName(listener)}.  " +
            s"Removing that listener.", ie)
          removeListenerOnError(listener)
        case NonFatal(e) =>
          logError(s"Listener ${Utils.getFormattedClassName(listener)} threw an exception", e)
      } finally {
        if (maybeTimerContext != null) {
          maybeTimerContext.stop()
        }
      }
    }
  }
```

> 差不多就是Spark的整个的监听了，很多细节还是没有讲到，需要大家自己挖掘。用户可以自己实现自定义监听器，例如Job失败发送邮件告警之类，自定义监听器需要设置到SparkConf中，SparkContext加载时会判断是否有自定义监听器