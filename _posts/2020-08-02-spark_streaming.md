---
layout: post
title:  "Spark源码学习笔记(二十四)"
date:   2020-08-02 22:46:50 +0800
tags:
- Spark
- Spark Streaming
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070809.webp'
subtitle: Spark Streaming详解
---
> 这篇可能是spark源码学习系列的最后一篇了，structed streaming不打算看了，有兴趣的同学可以自己研究。从spark架构中每一个重要的组件到sql，再到现在的streaming，看完源码，真的学习了很多东西吧。
> 
> 创建完spark streaming项目后，代码中需要调用StreamingContext的start方法启动。start方法中会调用JobScheduler的start方法。

```scala
  // JobScheduler的start方法
  def start(): Unit = synchronized {
    if (eventLoop != null) return // scheduler has already been started

    logDebug("Starting JobScheduler")
    eventLoop = new EventLoop[JobSchedulerEvent]("JobScheduler") {
      override protected def onReceive(event: JobSchedulerEvent): Unit = processEvent(event)

      override protected def onError(e: Throwable): Unit = reportError("Error in job scheduler", e)
    }
    eventLoop.start()

    // attach rate controllers of input streams to receive batch completion updates
    for {
      inputDStream <- ssc.graph.getInputStreams
      // 用于获取每批Job数据量大小
      rateController <- inputDStream.rateController
    } ssc.addStreamingListener(rateController)

    listenerBus.start()
    receiverTracker = new ReceiverTracker(ssc)
    inputInfoTracker = new InputInfoTracker(ssc)

    val executorAllocClient: ExecutorAllocationClient = ssc.sparkContext.schedulerBackend match {
      case b: ExecutorAllocationClient => b.asInstanceOf[ExecutorAllocationClient]
      case _ => null
    }

    executorAllocationManager = ExecutorAllocationManager.createIfEnabled(
      executorAllocClient,
      receiverTracker,
      ssc.conf,
      ssc.graph.batchDuration.milliseconds,
      clock)
    executorAllocationManager.foreach(ssc.addStreamingListener)
    receiverTracker.start()
    // jobgenerator生成每批Job
    jobGenerator.start()
    executorAllocationManager.foreach(_.start())
    logInfo("Started JobScheduler")
  }
```

> 调用JobGenerator.start()方法

```scala
  // JobGenerator类中方法
  /** Start generation of jobs */
  def start(): Unit = synchronized {
    if (eventLoop != null) return // generator has already been started

    // Call checkpointWriter here to initialize it before eventLoop uses it to avoid a deadlock.
    // See SPARK-10125
    checkpointWriter

    eventLoop = new EventLoop[JobGeneratorEvent]("JobGenerator") {
      override protected def onReceive(event: JobGeneratorEvent): Unit = processEvent(event)

      override protected def onError(e: Throwable): Unit = {
        jobScheduler.reportError("Error in job generator", e)
      }
    }
    eventLoop.start()

    if (ssc.isCheckpointPresent) {
      restart()
    } else {
      startFirstTime()
    }
  }
  
  /** Starts the generator for the first time */
  private def startFirstTime() {
    val startTime = new Time(timer.getStartTime())
    graph.start(startTime - graph.batchDuration)
    timer.start(startTime.milliseconds)
    logInfo("Started JobGenerator at " + startTime)
  }
  
  // RecurringTimer中有线程变量，调用start方法可以启动线程
  private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
    longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
```

> RecurringTimer类中会执行回调生成GenerateJobs事件

```scala
	// RecurringTimer类中的方法
  /**
   * Start at the given start time.
   */
  def start(startTime: Long): Long = synchronized {
    nextTime = startTime
    thread.start()
    logInfo("Started timer for " + name + " at time " + nextTime)
    nextTime
  }
  
  private val thread = new Thread("RecurringTimer - " + name) {
    setDaemon(true)
    override def run() { loop }
  }
  
  /**
   * Repeatedly call the callback every interval.
   */
  private def loop() {
    try {
      while (!stopped) {
        triggerActionForNextInterval()
      }
      triggerActionForNextInterval()
    } catch {
      case e: InterruptedException =>
    }
  }
  
  // callback就是JobGenerator中传入的longTime => eventLoop.post(GenerateJobs(new Time(longTime)))，period是传入的每一批的间隔时间，回调方法会调用GenerateJobs事件
  private def triggerActionForNextInterval(): Unit = {
    // 堵塞直到下一批开始执行时间
    clock.waitTillTime(nextTime)
    callback(nextTime)
    prevTime = nextTime
    nextTime += period
    logDebug("Callback for " + name + " called at time " + prevTime)
  }
```

> EventLoop已经见到过很多次了，类中有个线程+阻塞队列，线程不停的消费阻塞队列，并调用onReceive方法，JobGenerator中onReceive方法执行的是processEvent方法

```scala
  /** Processes all events */
  private def processEvent(event: JobGeneratorEvent) {
    logDebug("Got event " + event)
    event match {
      case GenerateJobs(time) => generateJobs(time)
      case ClearMetadata(time) => clearMetadata(time)
      case DoCheckpoint(time, clearCheckpointDataLater) =>
        doCheckpoint(time, clearCheckpointDataLater)
      case ClearCheckpointData(time) => clearCheckpointData(time)
    }
  }
  
  /** Generate jobs and perform checkpointing for the given `time`.  */
  private def generateJobs(time: Time) {
    // Checkpoint all RDDs marked for checkpointing to ensure their lineages are
    // truncated periodically. Otherwise, we may run into stack overflows (SPARK-6847).
    ssc.sparkContext.setLocalProperty(RDD.CHECKPOINT_ALL_MARKED_ANCESTORS, "true")
    Try {
      jobScheduler.receiverTracker.allocateBlocksToBatch(time) // allocate received blocks to batch
      // 生成每批的job
      graph.generateJobs(time) // generate jobs using allocated block
    } match {
      case Success(jobs) =>
        val streamIdToInputInfos = jobScheduler.inputInfoTracker.getInfo(time)
        // 提交Job
        jobScheduler.submitJobSet(JobSet(time, jobs, streamIdToInputInfos))
      case Failure(e) =>
        jobScheduler.reportError("Error generating jobs for time " + time, e)
        PythonDStream.stopStreamingContextIfPythonProcessIsDead(e)
    }
    eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = false))
  }
```

> DStreamGraph会调用Dstream类中的generateJob生成，生成Seq[Job]

```scala
  def generateJobs(time: Time): Seq[Job] = {
    logDebug("Generating jobs for time " + time)
    val jobs = this.synchronized {
      outputStreams.flatMap { outputStream =>
        // 调用Dstream的generateJob方法
        val jobOption = outputStream.generateJob(time)
        jobOption.foreach(_.setCallSite(outputStream.creationSite))
        jobOption
      }
    }
    logDebug("Generated " + jobs.length + " jobs for time " + time)
    jobs
  }
```

> Dstream中会为Job定义调用sparkContext的runJob方法

```scala
  /**
   * Generate a SparkStreaming job for the given time. This is an internal method that
   * should not be called directly. This default implementation creates a job
   * that materializes the corresponding RDD. Subclasses of DStream may override this
   * to generate their own jobs.
   */
  private[streaming] def generateJob(time: Time): Option[Job] = {
    // getOrCompute会获取每一批次对应的RDD
    getOrCompute(time) match {
      case Some(rdd) =>
        val jobFunc = () => {
          val emptyFunc = { (iterator: Iterator[T]) => {} }
          context.sparkContext.runJob(rdd, emptyFunc)
        }
        Some(new Job(time, jobFunc))
      case None => None
    }
  }
  
  /**
   * Get the RDD corresponding to the given time; either retrieve it from cache
   * or compute-and-cache it.
   */
  private[streaming] final def getOrCompute(time: Time): Option[RDD[T]] = {
    // If RDD was already generated, then retrieve it from HashMap,
    // or else compute the RDD
    generatedRDDs.get(time).orElse {
      // Compute the RDD if time is valid (e.g. correct time in a sliding window)
      // of RDD generation, else generate nothing.
      if (isTimeValid(time)) {

        val rddOption = createRDDWithLocalProperties(time, displayInnerRDDOps = false) {
          // Disable checks for existing output directories in jobs launched by the streaming
          // scheduler, since we may need to write output to an existing directory during checkpoint
          // recovery; see SPARK-4835 for more details. We need to have this call here because
          // compute() might cause Spark jobs to be launched.
         
          // 获取真正当前Time对应的RDD
          SparkHadoopWriterUtils.disableOutputSpecValidation.withValue(true) {
            // 内部会执行获取当前批次应该拉取的数据限制，反压机制，由DirectKafkaInputDStream的compute实现
            compute(time)
          }
        }

        rddOption.foreach { case newRDD =>
          // Register the generated RDD for caching and checkpointing
          if (storageLevel != StorageLevel.NONE) {
            // 如果持久化策略不是None，则持久化RDD，默认策略是None，调用DStream的persist/cache方法会改变该策略为StorageLevel.MEMORY_ONLY_SER
            newRDD.persist(storageLevel)
            logDebug(s"Persisting RDD ${newRDD.id} for time $time to $storageLevel")
          }
          if (checkpointDuration != null && (time - zeroTime).isMultipleOf(checkpointDuration)) {
            // 如果开启了checkpoint，则执行checkpoint
            newRDD.checkpoint()
            logInfo(s"Marking RDD ${newRDD.id} for time $time for checkpointing")
          }
          generatedRDDs.put(time, newRDD)
        }
        rddOption
      } else {
        None
      }
    }
  }
```

> 生成job后需要提交JobSet(jobs)

```scala
  def submitJobSet(jobSet: JobSet) {
    if (jobSet.jobs.isEmpty) {
      logInfo("No jobs added for time " + jobSet.time)
    } else {
      listenerBus.post(StreamingListenerBatchSubmitted(jobSet.toBatchInfo))
      jobSets.put(jobSet.time, jobSet)
      jobSet.jobs.foreach(job => jobExecutor.execute(new JobHandler(job)))
      logInfo("Added jobs for time " + jobSet.time)
    }
  }
  
  private class JobHandler(job: Job) extends Runnable with Logging {
    import JobScheduler._

    def run() {
      val oldProps = ssc.sparkContext.getLocalProperties
      try {
        ssc.sparkContext.setLocalProperties(SerializationUtils.clone(ssc.savedProperties.get()))
        val formattedTime = UIUtils.formatBatchTime(
          job.time.milliseconds, ssc.graph.batchDuration.milliseconds, showYYYYMMSS = false)
        val batchUrl = s"/streaming/batch/?id=${job.time.milliseconds}"
        val batchLinkText = s"[output operation ${job.outputOpId}, batch time ${formattedTime}]"

        ssc.sc.setJobDescription(
          s"""Streaming job from <a href="$batchUrl">$batchLinkText</a>""")
        ssc.sc.setLocalProperty(BATCH_TIME_PROPERTY_KEY, job.time.milliseconds.toString)
        ssc.sc.setLocalProperty(OUTPUT_OP_ID_PROPERTY_KEY, job.outputOpId.toString)
        // Checkpoint all RDDs marked for checkpointing to ensure their lineages are
        // truncated periodically. Otherwise, we may run into stack overflows (SPARK-6847).
        ssc.sparkContext.setLocalProperty(RDD.CHECKPOINT_ALL_MARKED_ANCESTORS, "true")

        // We need to assign `eventLoop` to a temp variable. Otherwise, because
        // `JobScheduler.stop(false)` may set `eventLoop` to null when this method is running, then
        // it's possible that when `post` is called, `eventLoop` happens to null.
        var _eventLoop = eventLoop
        if (_eventLoop != null) {
          _eventLoop.post(JobStarted(job, clock.getTimeMillis()))
          // Disable checks for existing output directories in jobs launched by the streaming
          // scheduler, since we may need to write output to an existing directory during checkpoint
          // recovery; see SPARK-4835 for more details.
         // job.run会真正的执行方法，方法在Dstream中定义，封装到Job类中，执行run会触发sparkContext.runJob方法提交rdd执行
          SparkHadoopWriterUtils.disableOutputSpecValidation.withValue(true) {
            job.run()
          }
          _eventLoop = eventLoop
          if (_eventLoop != null) {
            // 发送JobComplete事件
            _eventLoop.post(JobCompleted(job, clock.getTimeMillis()))
          }
        } else {
          // JobScheduler has been stopped.
        }
      } finally {
        ssc.sparkContext.setLocalProperties(oldProps)
      }
    }
  }
```
> 每批数据处理过程都会如此，用户如果调用DStream的cache/persist方法，但是DStream并没有提供unpersist方法，streaming会自动清理cache的数据，由spark.streaming.unpersist配置，默认值为true
> 
> 当每一批Job完成后发送JobCompleted事件到JobScheduler的eventLoop中,该事件对应执行handleJobCompletion方法，之后执行JobGenerator的onBatchCompletion方法，该方法会发送ClearMetadata事件到eventLoop中

```scala
  /** Clear DStream metadata for the given `time`. */
  private def clearMetadata(time: Time) {
    // 调用clearMetadata方法
    ssc.graph.clearMetadata(time)

    // If checkpointing is enabled, then checkpoint,
    // else mark batch to be fully processed
    if (shouldCheckpoint) {
      eventLoop.post(DoCheckpoint(time, clearCheckpointDataLater = true))
    } else {
      // If checkpointing is not enabled, then delete metadata information about
      // received blocks (block data not saved in any case). Otherwise, wait for
      // checkpointing of this batch to complete.
      val maxRememberDuration = graph.getMaxInputStreamRememberDuration()
      jobScheduler.receiverTracker.cleanupOldBlocksAndBatches(time - maxRememberDuration)
      jobScheduler.inputInfoTracker.cleanup(time - maxRememberDuration)
      markBatchFullyProcessed(time)
    }
  }
  
  def clearMetadata(time: Time) {
    logDebug("Clearing metadata for time " + time)
    this.synchronized {
      outputStreams.foreach(_.clearMetadata(time))
    }
    logDebug("Cleared old metadata for time " + time)
  }
  
  /**
   * Clear metadata that are older than `rememberDuration` of this DStream.
   * This is an internal method that should not be called directly. This default
   * implementation clears the old generated RDDs. Subclasses of DStream may override
   * this to clear their own metadata along with the generated RDDs.
   */
   // 真正执行clear Metadata方法
  private[streaming] def clearMetadata(time: Time) {
    val unpersistData = ssc.conf.getBoolean("spark.streaming.unpersist", true)
    val oldRDDs = generatedRDDs.filter(_._1 <= (time - rememberDuration))
    logDebug("Clearing references to old RDDs: [" +
      oldRDDs.map(x => s"${x._1} -> ${x._2.id}").mkString(", ") + "]")
    generatedRDDs --= oldRDDs.keys
    if (unpersistData) {
      logDebug(s"Unpersisting old RDDs: ${oldRDDs.values.map(_.id).mkString(", ")}")
      // 清理过期的rdd缓存数据
      oldRDDs.values.foreach { rdd =>
        rdd.unpersist(false)
        // Explicitly remove blocks of BlockRDD
        rdd match {
          case b: BlockRDD[_] =>
            logInfo(s"Removing blocks of RDD $b of time $time")
            b.removeBlocks()
          case _ =>
        }
      }
    }
    logDebug(s"Cleared ${oldRDDs.size} RDDs that were older than " +
      s"${time - rememberDuration}: ${oldRDDs.keys.mkString(", ")}")
    dependencies.foreach(_.clearMetadata(time))
  }
```

> 还有一个想说的就是streaming的反压机制，需要用户开启spark.streaming.backpressure.enabled，反压机制就是streaming能够根据当前情况，自动获取当前适合处理的条目数，而不一定是用户设置的最大的当前处理数，spark.streaming.kafka.maxRatePerPartition（每秒钟每个分区拉取数目，如果topic有3个分区，streaming间隔为5s,该配置为1000,则每批拉取最大条目为15000条数据）
> 

```scala
 /**
   * Asynchronously maintains & sends new rate limits to the receiver through the receiver tracker.
   */
  override protected[streaming] val rateController: Option[RateController] = {
    // 开启反压机制后，rateController变量会使用DirectKafkaRateController
    if (RateController.isBackPressureEnabled(ssc.conf)) {
      Some(new DirectKafkaRateController(id,
        RateEstimator.create(ssc.conf, context.graph.batchDuration)))
    } else {
      None
    }
  }
  
  protected[streaming] def maxMessagesPerPartition(
    offsets: Map[TopicPartition, Long]): Option[Map[TopicPartition, Long]] = {
    // 使用getLatestRate获取最新速率大小
    val estimatedRateLimit = rateController.map(_.getLatestRate())

    // calculate a per-partition rate limit based on current lag
    val effectiveRateLimitPerPartition = estimatedRateLimit.filter(_ > 0) match {
      case Some(rate) =>
        val lagPerPartition = offsets.map { case (tp, offset) =>
          tp -> Math.max(offset - currentOffsets(tp), 0)
        }
        val totalLag = lagPerPartition.values.sum

        lagPerPartition.map { case (tp, lag) =>
          // 用户的配置的spark.streaming.kafka.maxRatePerPartition值
          val maxRateLimitPerPartition = ppc.maxRatePerPartition(tp)
          // 反压机制推测的值
          val backpressureRate = Math.round(lag / totalLag.toFloat * rate)
          tp -> (if (maxRateLimitPerPartition > 0) {
            // 二者取比较小的值
            Math.min(backpressureRate, maxRateLimitPerPartition)} else backpressureRate)
        }
      case None => offsets.map { case (tp, offset) => tp -> ppc.maxRatePerPartition(tp) }
    }

    if (effectiveRateLimitPerPartition.values.sum > 0) {
      val secsPerBatch = context.graph.batchDuration.milliseconds.toDouble / 1000
      Some(effectiveRateLimitPerPartition.map {
        // 每秒 * 每秒limit值就是该批每个分区应该获取的值
        case (tp, limit) => tp -> (secsPerBatch * limit).toLong
      })
    } else {
      None
    }
  }
```

> StreamingListenerBus中有事件StreamingListenerBatchCompleted对应批完成，相应执行onBatchCompleted方法，并执行computeAndPublish方法

```scala
  override def onBatchCompleted(batchCompleted: StreamingListenerBatchCompleted) {
    val elements = batchCompleted.batchInfo.streamIdToInputInfo

    for {
      processingEnd <- batchCompleted.batchInfo.processingEndTime
      workDelay <- batchCompleted.batchInfo.processingDelay
      waitDelay <- batchCompleted.batchInfo.schedulingDelay
      elems <- elements.get(streamUID).map(_.numRecords)
    } computeAndPublish(processingEnd, elems, workDelay, waitDelay)
  }
  
  private def computeAndPublish(time: Long, elems: Long, workDelay: Long, waitDelay: Long): Unit =
    Future[Unit] {
      // 该方法用于计算最新的限制速率，compute方法由PIDRateEstimator中实现
      val newRate = rateEstimator.compute(time, elems, workDelay, waitDelay)
      newRate.foreach { s =>
        rateLimit.set(s.toLong)
        publish(getLatestRate())
      }
    }
    
  def compute(
      time: Long, // in milliseconds
      numElements: Long,
      processingDelay: Long, // in milliseconds
      schedulingDelay: Long // in milliseconds
    ): Option[Double] = {
    logTrace(s"\ntime = $time, # records = $numElements, " +
      s"processing time = $processingDelay, scheduling delay = $schedulingDelay")
    this.synchronized {
      if (time > latestTime && numElements > 0 && processingDelay > 0) {

        // in seconds, should be close to batchDuration
        val delaySinceUpdate = (time - latestTime).toDouble / 1000

        // in elements/second
        val processingRate = numElements.toDouble / processingDelay * 1000

        // In our system `error` is the difference between the desired rate and the measured rate
        // based on the latest batch information. We consider the desired rate to be latest rate,
        // which is what this estimator calculated for the previous batch.
        // in elements/second
        val error = latestRate - processingRate

        // The error integral, based on schedulingDelay as an indicator for accumulated errors.
        // A scheduling delay s corresponds to s * processingRate overflowing elements. Those
        // are elements that couldn't be processed in previous batches, leading to this delay.
        // In the following, we assume the processingRate didn't change too much.
        // From the number of overflowing elements we can calculate the rate at which they would be
        // processed by dividing it by the batch interval. This rate is our "historical" error,
        // or integral part, since if we subtracted this rate from the previous "calculated rate",
        // there wouldn't have been any overflowing elements, and the scheduling delay would have
        // been zero.
        // (in elements/second)
        val historicalError = schedulingDelay.toDouble * processingRate / batchIntervalMillis

        // in elements/(second ^ 2)
        val dError = (error - latestError) / delaySinceUpdate

        val newRate = (latestRate - proportional * error -
                                    integral * historicalError -
                                    derivative * dError).max(minRate)
        logTrace(s"""
            | latestRate = $latestRate, error = $error
            | latestError = $latestError, historicalError = $historicalError
            | delaySinceUpdate = $delaySinceUpdate, dError = $dError
            """.stripMargin)

        latestTime = time
        if (firstRun) {
          latestRate = processingRate
          latestError = 0D
          firstRun = false
          logTrace("First run, rate estimation skipped")
          None
        } else {
          latestRate = newRate
          latestError = error
          logTrace(s"New rate = $newRate")
          Some(newRate)
        }
      } else {
        logTrace("Rate estimation skipped")
        None
      }
    }
  }
```
> 也就是说，每一批任务执行完成后，就会执行相应的方法，更新最新的拉取速率

> 在说说offset管理吧

```scala
  // 第一次启动时调用，consumer是KafkaConsumer
  override def start(): Unit = {
    // consumer会创建KafkaConsumer
    val c = consumer
    paranoidPoll(c)
    if (currentOffsets.isEmpty) {
      currentOffsets = c.assignment().asScala.map { tp =>
        // position方法会获取每个TopicPartition在kafka中的offset，consumer中会使用seek方法设置offset,将用户传入的offset设置到kafka
        tp -> c.position(tp)
      }.toMap
    }

    // don't actually want to consume any messages, so pause all partitions
    c.pause(currentOffsets.keySet.asJava)
  }
  
  // spark自身管理的offset
  protected var currentOffsets = Map[TopicPartition, Long]()

  @transient private var kc: Consumer[K, V] = null
  def consumer(): Consumer[K, V] = this.synchronized {
    if (null == kc) {
    	// 内部方法会判断currentOffsets的值，后续操作会使用spark的currentOffsets拉取kafka数据
      kc = consumerStrategy.onStart(currentOffsets.mapValues(l => new java.lang.Long(l)).asJava)
    }
    kc
  }
  
  private case class Subscribe[K, V](
    topics: ju.Collection[jl.String],
    kafkaParams: ju.Map[String, Object],
    offsets: ju.Map[TopicPartition, jl.Long]
  ) extends ConsumerStrategy[K, V] with Logging {

  def executorKafkaParams: ju.Map[String, Object] = kafkaParams

  def onStart(currentOffsets: ju.Map[TopicPartition, jl.Long]): Consumer[K, V] = {
    val consumer = new KafkaConsumer[K, V](kafkaParams)
    consumer.subscribe(topics)
    val toSeek = if (currentOffsets.isEmpty) {
    	// 第一次启动时，currentOffsets为空，使用用户创建directStream传入的offsets
      offsets
    } else {
      currentOffsets
    }
    if (!toSeek.isEmpty) {
      // work around KAFKA-3370 when reset is none
      // poll will throw if no position, i.e. auto offset reset none and no explicit position
      // but cant seek to a position before poll, because poll is what gets subscription partitions
      // So, poll, suppress the first exception, then seek
      val aor = kafkaParams.get(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG)
      val shouldSuppress =
        aor != null && aor.asInstanceOf[String].toUpperCase(Locale.ROOT) == "NONE"
      try {
        consumer.poll(0)
      } catch {
        case x: NoOffsetForPartitionException if shouldSuppress =>
          logWarning("Catching NoOffsetForPartitionException since " +
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG + " is none.  See KAFKA-3370")
      }
      toSeek.asScala.foreach { case (topicPartition, offset) =>
      	// 设置offset
          consumer.seek(topicPartition, offset)
      }
      // we've called poll, we must pause or next poll may consume messages and set position
      consumer.pause(consumer.assignment())
    }

    consumer
  }
}

  /**
   * Returns the latest (highest) available offsets, taking new partitions into account.
   */
  protected def latestOffsets(): Map[TopicPartition, Long] = {
    val c = consumer
    paranoidPoll(c)
    val parts = c.assignment().asScala

    // make sure new partitions are reflected in currentOffsets
    val newPartitions = parts.diff(currentOffsets.keySet)
    // position for new partitions determined by auto.offset.reset if no commit
    // 每次拉取数据前都会获取新增加的topic分区，即streaming可以动态识别kafka增加分区
    currentOffsets = currentOffsets ++ newPartitions.map(tp => tp -> c.position(tp)).toMap
    // don't want to consume messages, so pause
    c.pause(newPartitions.asJava)
    // find latest available offsets
    c.seekToEnd(currentOffsets.keySet.asJava)
    parts.map(tp => tp -> c.position(tp)).toMap
  }

  // limits the maximum number of messages per partition
  protected def clamp(
    offsets: Map[TopicPartition, Long]): Map[TopicPartition, Long] = {

    maxMessagesPerPartition(offsets).map { mmp =>
      mmp.map { case (tp, messages) =>
          // 最新offset
          val uo = offsets(tp)
          // 获取当前offset + 消费大小与最新offset的小值，做为该批次的until offset
          tp -> Math.min(currentOffsets(tp) + messages, uo)
      }
    }.getOrElse(offsets)
  }

  // 重写DStream的compute方法，获取RDD信息
  override def compute(validTime: Time): Option[KafkaRDD[K, V]] = {
    val untilOffsets = clamp(latestOffsets())
    val offsetRanges = untilOffsets.map { case (tp, uo) =>
      val fo = currentOffsets(tp)
      OffsetRange(tp.topic, tp.partition, fo, uo)
    }
    val useConsumerCache = context.conf.getBoolean("spark.streaming.kafka.consumer.cache.enabled",
      true)
    // 方法底层拉取kafka数据  
    val rdd = new KafkaRDD[K, V](context.sparkContext, executorKafkaParams, offsetRanges.toArray,
      getPreferredHosts, useConsumerCache)

    // Report the record number and metadata of this batch interval to InputInfoTracker.
    val description = offsetRanges.filter { offsetRange =>
      // Don't display empty ranges.
      offsetRange.fromOffset != offsetRange.untilOffset
    }.map { offsetRange =>
      s"topic: ${offsetRange.topic}\tpartition: ${offsetRange.partition}\t" +
        s"offsets: ${offsetRange.fromOffset} to ${offsetRange.untilOffset}"
    }.mkString("\n")
    // Copy offsetRanges to immutable.List to prevent from being modified by the user
    val metadata = Map(
      "offsets" -> offsetRanges.toList,
      StreamInputInfo.METADATA_KEY_DESCRIPTION -> description)
    val inputInfo = StreamInputInfo(id, rdd.count, metadata)
    ssc.scheduler.inputInfoTracker.reportInfo(validTime, inputInfo)

	// 拉取kafka数据成功后，更新spark中当前offset为最新的offset，currentOffsets是spark中管理的offsets
    currentOffsets = untilOffsets
    // 提交offset
    commitAll()
    Some(rdd)
  }
  
  // 存放OffsetRange
  protected val commitQueue = new ConcurrentLinkedQueue[OffsetRange]
  
  protected def commitAll(): Unit = {
    val m = new ju.HashMap[TopicPartition, OffsetAndMetadata]()
    // offset队列中取数据，每次读完数据后
    var osr = commitQueue.poll()
    while (null != osr) {
      val tp = osr.topicPartition
      val x = m.get(tp)
      // 获取该partition最大的offset
      val offset = if (null == x) { osr.untilOffset } else { Math.max(x.offset, osr.untilOffset) }
      m.put(tp, new OffsetAndMetadata(offset))
      osr = commitQueue.poll()
    }
    if (!m.isEmpty) {
      // 提交offset到kafka，enable.auto.commit参数用于控制consumer是否拉取数据后直接提交offset到kafka
      consumer.commitAsync(m, commitCallback.get)
    }
  }
  
  /**
   * Queue up offset ranges for commit to Kafka at a future time.  Threadsafe.
   * @param offsetRanges The maximum untilOffset for a given partition will be used at commit.
   */
  // 用户调用commit offset接口
  def commitAsync(offsetRanges: Array[OffsetRange]): Unit = {
    commitAsync(offsetRanges, null)
  }

  /**
   * Queue up offset ranges for commit to Kafka at a future time.  Threadsafe.
   * @param offsetRanges The maximum untilOffset for a given partition will be used at commit.
   * @param callback Only the most recently provided callback will be used at commit.
   */
  def commitAsync(offsetRanges: Array[OffsetRange], callback: OffsetCommitCallback): Unit = {
    commitCallback.set(callback)
    commitQueue.addAll(ju.Arrays.asList(offsetRanges: _*))
```

> 所以spark只能保证kafka到spark的exactly once语义，只拉取一次kafka数据，但是如果spark某一批次数据处理失败(比如调用第三方服务失败等等)，导致没有写入第三方没办法保证，抛开这些情况，spark RDD拥有血缘依赖，可以重新生成，也可以使用checkpoint机制存储，不存在数据从kafka拉取后丢失问题
> 
> 因为spark使用的是自身管理的offset拉取数据，所以当每批任务提交后，即使offset没有提交kafka集群/第三方管理，spark也会继续拉取下一批次的数据，不会停留在一直拉取这批数据的阶段，当程序重启时，又会从没有提交的offset拉取