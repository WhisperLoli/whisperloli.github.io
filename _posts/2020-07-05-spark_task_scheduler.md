---
layout: post
title:  "Spark源码学习笔记（十五）"
date:   2020-07-05 12:29:13 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_03.webp'
subtitle: 'TaskScheduler'
---

> 一开始以为任务调度挺简单的，后来看了源码才发现这块涉及的东西非常多，比想象的复杂的多
> 
> 先说下Spark的基本概念吧，用户提交的任务在spark中被称为Application，一个Application会被划分成多个Job执行，每个Action蒜子会提交一个Job，Job里面又会划分Stage，这块根据宽依赖划分，而一个Stage里面会产生一个TaskSet，TaskSet包含了所有的Task，每个task的建议大小是100kb内
> 
> Task分为ShuffleMapTask和ResultTask两种，二者均继承自Task特质，重写了runTask方法。ShuffleMapTask的作用是划分task output到多个bucket中，基于task的partitioner，底层最终使用的是ShuffleDependency的partitioner，task的partitioner会使用上一个task的partitioner，如果没有，则使用默认的HashPartitioner；ResultTask的作用是发送task output给driver
> 
> TaskInfo存储task详细信息，例如运行的executor,host，启动时间、结束时间等信息等
> 
> TaskContextImpl：实现了TaskContext，内部包含了Task需要的所有信息，stageId、partitionId、taskAttemptId等等，还可以判断task是否完成，添加完成监听器/失败监听器，可以用markTaskCompleted强行将task置于完成状态并触发监听事件，同样也可以置于失败，会在Task的runTask中被调用
> 
> TaskSetManager：当TaskSet被提交运行时，TaskSetManager负责跟踪TaskSet中的每一次Task，如果有失败的，则重试，resourceOffer返回将要传递给executor执行的task描述
> 
> TaskDescription：传递给executor执行的task的描述，提供了encode/decode方法序列化/反序列化TaskDescription，序列化的时间和序列化后的大小都很重要，所以使用自身的方法，可以避免序列化一些不必要的字段
> 
> SchedulableBuilder: 构建调度池，添加TaskSetManager，现有的调度策略是FIFO和FAIR策略，TaskScheduler使用的默认策略是FIFO
> 
> Pool: 可执行的实体，TaskSetManagers的集合，该类中会创建相应的调度算法，每个TaskSetManager也会被添加到待调度的队列中，提供getSortedTaskSetQueue方法使用调度算法给TaskSet队列排序
> 
> TaskSchedulerImpl: 继承自TaskScheduler, SparkContext中创建完taskScheduler后，会调用initialize方法，初始化backend和判断调度策略并创建相应的SchedulableBuilder对象, 最重要的方法submitTasks，提交一系列的task（也就是一个TaskSet）到SchedulableBuilder调度池中, resourceOffer会排除黑名单中的executor，获取调度池中根据调度策略排序后的任务队列，随机为每个executor分配任务，避免任务总在同一个worker上。start方法也很重要，很启动线程判断推测执行，当有一个task运行时间过长，就会在启动一个相同的task执行这个任务

```scala
  override def submitTasks(taskSet: TaskSet) {
    val tasks = taskSet.tasks
    logInfo("Adding task set " + taskSet.id + " with " + tasks.length + " tasks")
    this.synchronized {
      val manager = createTaskSetManager(taskSet, maxTaskFailures)
      val stage = taskSet.stageId
      val stageTaskSets =
        taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])
      stageTaskSets(taskSet.stageAttemptId) = manager
      val conflictingTaskSet = stageTaskSets.exists { case (_, ts) =>
        ts.taskSet != taskSet && !ts.isZombie
      }
      if (conflictingTaskSet) {
        throw new IllegalStateException(s"more than one active taskSet for stage $stage:" +
          s" ${stageTaskSets.toSeq.map{_._2.taskSet.id}.mkString(",")}")
      }
      schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)

      if (!isLocal && !hasReceivedTask) {
        starvationTimer.scheduleAtFixedRate(new TimerTask() {
          override def run() {
            if (!hasLaunchedTask) {
              logWarning("Initial job has not accepted any resources; " +
                "check your cluster UI to ensure that workers are registered " +
                "and have sufficient resources")
            } else {
              this.cancel()
            }
          }
        }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
      }
      hasReceivedTask = true
    }
    backend.reviveOffers()
  }
  
  
  def resourceOffers(offers: IndexedSeq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {
    // Mark each slave as alive and remember its hostname
    // Also track if new executor is added
    var newExecAvail = false
    for (o <- offers) {
      if (!hostToExecutors.contains(o.host)) {
        hostToExecutors(o.host) = new HashSet[String]()
      }
      if (!executorIdToRunningTaskIds.contains(o.executorId)) {
        hostToExecutors(o.host) += o.executorId
        executorAdded(o.executorId, o.host)
        executorIdToHost(o.executorId) = o.host
        executorIdToRunningTaskIds(o.executorId) = HashSet[Long]()
        newExecAvail = true
      }
      for (rack <- getRackForHost(o.host)) {
        hostsByRack.getOrElseUpdate(rack, new HashSet[String]()) += o.host
      }
    }

    // Before making any offers, remove any nodes from the blacklist whose blacklist has expired. Do
    // this here to avoid a separate thread and added synchronization overhead, and also because
    // updating the blacklist is only relevant when task offers are being made.
    blacklistTrackerOpt.foreach(_.applyBlacklistTimeout())

    val filteredOffers = blacklistTrackerOpt.map { blacklistTracker =>
      offers.filter { offer =>
        !blacklistTracker.isNodeBlacklisted(offer.host) &&
          !blacklistTracker.isExecutorBlacklisted(offer.executorId)
      }
    }.getOrElse(offers)

    val shuffledOffers = shuffleOffers(filteredOffers)
    // Build a list of tasks to assign to each worker.
    val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores / CPUS_PER_TASK))
    val availableCpus = shuffledOffers.map(o => o.cores).toArray
    val sortedTaskSets = rootPool.getSortedTaskSetQueue
    for (taskSet <- sortedTaskSets) {
      logDebug("parentName: %s, name: %s, runningTasks: %s".format(
        taskSet.parent.name, taskSet.name, taskSet.runningTasks))
      if (newExecAvail) {
        taskSet.executorAdded()
      }
    }

    // Take each TaskSet in our scheduling order, and then offer it each node in increasing order
    // of locality levels so that it gets a chance to launch local tasks on all of them.
    // NOTE: the preferredLocality order: PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY
    for (taskSet <- sortedTaskSets) {
      var launchedAnyTask = false
      var launchedTaskAtCurrentMaxLocality = false
      for (currentMaxLocality <- taskSet.myLocalityLevels) {
        do {
          launchedTaskAtCurrentMaxLocality = resourceOfferSingleTaskSet(
            taskSet, currentMaxLocality, shuffledOffers, availableCpus, tasks)
          launchedAnyTask |= launchedTaskAtCurrentMaxLocality
        } while (launchedTaskAtCurrentMaxLocality)
      }
      if (!launchedAnyTask) {
        taskSet.abortIfCompletelyBlacklisted(hostToExecutors)
      }
    }

    if (tasks.size > 0) {
      hasLaunchedTask = true
    }
    return tasks
  }
```

> 说了这么多，TaskScheduler只是做了明面上的工作，而真正启动task的是SchedulerBackend对象
> 
> SchedulerBackend的实现有LocalSchedulerBackend，CoarseGrainedSchedulerBackend，前者用于本地模式，后者应用于集群模式，StandaloneSchedulerBackend继承CoarseGrainedSchedulerBackend
> 
> 先讲讲LocalSchedulerBackend，TaskScheduler的start方法会调用SchedulerBackend的start方法，创建endpoint和endpointRef，killTask/statusUpdate/reviveOffers都会发送RPC消息给endpointRef，真正的执行task交由executor操作

```scala
  def reviveOffers() {
    val offers = IndexedSeq(new WorkerOffer(localExecutorId, localExecutorHostname, freeCores))
    for (task <- scheduler.resourceOffers(offers).flatten) {
      freeCores -= scheduler.CPUS_PER_TASK
      executor.launchTask(executorBackend, task)
    }
  }
```
  
> CoarseGrainedSchedulerBackend会比local的实现复杂很多，会涉及到网络间的通信，同local方式类似，在start方法中会创建endpoint和endpointRef，endpoint的onStart方法会创建定时线程池，每秒钟去获取一次task，其他的也都类似，集群中Task交由特定的executor执行，CoarseGrainedExecutorBackend该类负责接收executor发送的RPC消息，执行Task，注意ExecutorData中的executorEndpoint是他自身的executorEndpoint，不是driver的，所以如下代码，会发送task到executor上执行

```scala
private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {
      for (task <- tasks.flatten) {
        val serializedTask = TaskDescription.encode(task)
        if (serializedTask.limit() >= maxRpcMessageSize) {
          Option(scheduler.taskIdToTaskSetManager.get(task.taskId)).foreach { taskSetMgr =>
            try {
              var msg = "Serialized task %s:%d was %d bytes, which exceeds max allowed: " +
                "spark.rpc.message.maxSize (%d bytes). Consider increasing " +
                "spark.rpc.message.maxSize or using broadcast variables for large values."
              msg = msg.format(task.taskId, task.index, serializedTask.limit(), maxRpcMessageSize)
              taskSetMgr.abort(msg)
            } catch {
              case e: Exception => logError("Exception in error callback", e)
            }
          }
        }
        else {
          val executorData = executorDataMap(task.executorId)
          executorData.freeCores -= scheduler.CPUS_PER_TASK

          logDebug(s"Launching task ${task.taskId} on executor id: ${task.executorId} hostname: " +
            s"${executorData.executorHost}.")

          executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
        }
      }
    }
```
> 
> ExecutorData用于封装executor的相关信息

```scala
private[cluster] class ExecutorData(
   val executorEndpoint: RpcEndpointRef,
   val executorAddress: RpcAddress,
   override val executorHost: String,
   var freeCores: Int,
   override val totalCores: Int,
   override val logUrlMap: Map[String, String]
) extends ExecutorInfo(executorHost, totalCores, logUrlMap)
```

> StandaloneSchedulerBackend继承CoarseGrainedSchedulerBackend，在start方法中调用父类start方法后，会创建自己的StandaloneAppClient，做一些通信相关的实现
> 
> 基于Mesos也有一套SchedulerBackend，这里就不展开了