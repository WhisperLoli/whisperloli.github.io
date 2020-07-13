---
layout: post
title:  "Spark源码学习笔记（十七）"
date:   2020-07-08 22:14:13 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_01.webp'
subtitle: Dynamic Allocation Executor
---

> 先解释一下master，worker，driver，executor概念吧，很多人用了很久的spark也是一知半解，Master和Workder对应的是物理节点，driver和executor对应的是节点中的进程，driver既可以在master，也可以在workder上，但是executor只能存在于worker上，本章要分享的动态分配executor，会使用到workder，master与worker是对应standalone/mesos模式，yarn模式不存在该概念
> 
> 如果开启了动态分配executor，ExecutorAllocationManager会在SparkContext中被创建，并调用实例的start方法，start方法中使用线程池定时调度线程判断当前分配的executor是缺了还是多了，缺少则申请，多了不会移除，因为executor等待时间太长会自动执行移除，那如何判当前需要的executor数量呢？计算公式如下，(正在运行的task数量+pending的task数量+每个executor能并行的task-1)/每个executor能并发的task-1，计算出来的结果就是我们需要的executor数量，之后会做出判断，是否超出设置的executor最大值/最小值

```scala
private val tasksPerExecutor =
    conf.getInt("spark.executor.cores", 1) / conf.getInt("spark.task.cpus", 1)

/**
   * The maximum number of executors we would need under the current load to satisfy all running
   * and pending tasks, rounded up.
   */
  private def maxNumExecutorsNeeded(): Int = {
    val numRunningOrPendingTasks = listener.totalPendingTasks + listener.totalRunningTasks
    (numRunningOrPendingTasks + tasksPerExecutor - 1) / tasksPerExecutor
  }
```
> 之后使用client.requestTotalExecutors(numExecutorsTarget, localityAwareTasks, hostToLocalTaskCount)执行申请，该方法会进入CoarseGrainedSchedulerBackend中并调用doRequestTotalExecutors(requestedTotalExecutors)方法，doRequestTotalExecutors在StandaloneSchedulerBackend被重写。其他的实现比如yarn中也重写了，在此只讲Standalone模式，yarn的实现在YarnSchedulerBackend中，client/cluster模式都是如此，再通过ApplicationMaster申请。重写后的方法使用StandaloneAppClient的requestTotalExecutors方法，方法如下，会使用endpointRef发送RequestExecutors消息

```scala
/**
   * Request executors from the Master by specifying the total number desired,
   * including existing pending and running executors.
   *
   * @return whether the request is acknowledged.
   */
  def requestTotalExecutors(requestedTotal: Int): Future[Boolean] = {
    if (endpoint.get != null && appId.get != null) {
      endpoint.get.ask[Boolean](RequestExecutors(appId.get, requestedTotal))
    } else {
      logWarning("Attempted to request executors before driver fully initialized.")
      Future.successful(false)
    }
  }
```

> 接着endpoint会对消息做出回应，使用了master的endpointRef发送消息

```scala
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
      case StopAppClient =>
        markDead("Application has been stopped.")
        sendToMaster(UnregisterApplication(appId.get))
        context.reply(true)
        stop()

      case r: RequestExecutors =>
        master match {
          case Some(m) => askAndReplyAsync(m, context, r)
          case None =>
            logWarning("Attempted to request executors before registering with Master.")
            context.reply(false)
        }

      case k: KillExecutors =>
        master match {
          case Some(m) => askAndReplyAsync(m, context, k)
          case None =>
            logWarning("Attempted to kill executors before registering with Master.")
            context.reply(false)
        }
    }

    private def askAndReplyAsync[T](
        endpointRef: RpcEndpointRef,
        context: RpcCallContext,
        msg: T): Unit = {
      // Ask a message and create a thread to reply with the result.  Allow thread to be
      // interrupted during shutdown, otherwise context must be notified of NonFatal errors.
      endpointRef.ask[Boolean](msg).andThen {
        case Success(b) => context.reply(b)
        case Failure(ie: InterruptedException) => // Cancelled
        case Failure(NonFatal(t)) => context.sendFailure(t)
      }(ThreadUtils.sameThread)
    }
```

> 接下来该进入Master类了，Master类继承了ThreadSafeRpcEndpoint，所以他自身就是个endpoint类，Master收到RequestExecutors后，执行handleRequestExecutors方法，注意如下会用到HashMap，appId对应唯一的ApplicationInfo，为什么会这样呢？因为用户启动spark集群的start-all.sh实际就是执行Master、Worker类的主函数，spark集群中肯定会存在多个spark application，所以使用HashMap存储

```scala
  private def handleRequestExecutors(appId: String, requestedTotal: Int): Boolean = {
    idToApp.get(appId) match {
      case Some(appInfo) =>
        logInfo(s"Application $appId requested to set total executors to $requestedTotal.")
        appInfo.executorLimit = requestedTotal
        schedule()
        true
      case None =>
        logWarning(s"Unknown application $appId requested $requestedTotal total executors.")
        false
    }
  }
  
  private def schedule(): Unit = {
    if (state != RecoveryState.ALIVE) {
      return
    }
    // Drivers take strict precedence over executors
    val shuffledAliveWorkers = Random.shuffle(workers.toSeq.filter(_.state == WorkerState.ALIVE))
    val numWorkersAlive = shuffledAliveWorkers.size
    var curPos = 0
    for (driver <- waitingDrivers.toList) { // iterate over a copy of waitingDrivers
      // We assign workers to each waiting driver in a round-robin fashion. For each driver, we
      // start from the last worker that was assigned a driver, and continue onwards until we have
      // explored all alive workers.
      var launched = false
      var numWorkersVisited = 0
      while (numWorkersVisited < numWorkersAlive && !launched) {
        val worker = shuffledAliveWorkers(curPos)
        numWorkersVisited += 1
        if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
          launchDriver(worker, driver)
          waitingDrivers -= driver
          launched = true
        }
        curPos = (curPos + 1) % numWorkersAlive
      }
    }
    startExecutorsOnWorkers()
  }
```

> 在shedule方法中会判断driver是否启动，如果没有启动的话会先启动driver，再启动executor

```scala
  private def startExecutorsOnWorkers(): Unit = {
    // Right now this is a very simple FIFO scheduler. We keep trying to fit in the first app
    // in the queue, then the second app, etc.
    for (app <- waitingApps) {
      val coresPerExecutor = app.desc.coresPerExecutor.getOrElse(1)
      // If the cores left is less than the coresPerExecutor,the cores left will not be allocated
      if (app.coresLeft >= coresPerExecutor) {
        // Filter out workers that don't have enough resources to launch an executor
        val usableWorkers = workers.toArray.filter(_.state == WorkerState.ALIVE)
          .filter(worker => worker.memoryFree >= app.desc.memoryPerExecutorMB &&
            worker.coresFree >= coresPerExecutor)
          .sortBy(_.coresFree).reverse
        val assignedCores = scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)

        // Now that we've decided how many cores to allocate on each worker, let's allocate them
        for (pos <- 0 until usableWorkers.length if assignedCores(pos) > 0) {
          allocateWorkerResourceToExecutors(
            app, assignedCores(pos), app.desc.coresPerExecutor, usableWorkers(pos))
        }
      }
    }
  }
  
  private def allocateWorkerResourceToExecutors(
      app: ApplicationInfo,
      assignedCores: Int,
      coresPerExecutor: Option[Int],
      worker: WorkerInfo): Unit = {
    // If the number of cores per executor is specified, we divide the cores assigned
    // to this worker evenly among the executors with no remainder.
    // Otherwise, we launch a single executor that grabs all the assignedCores on this worker.
    val numExecutors = coresPerExecutor.map { assignedCores / _ }.getOrElse(1)
    val coresToAssign = coresPerExecutor.getOrElse(assignedCores)
    for (i <- 1 to numExecutors) {
      val exec = app.addExecutor(worker, coresToAssign)
      launchExecutor(worker, exec)
      app.state = ApplicationState.RUNNING
    }
  }
```

> 一步一步下来，最终会调用到launchExecutor方法，发送rpc请求LaunchExecutor

```scala
private def launchExecutor(worker: WorkerInfo, exec: ExecutorDesc): Unit = {
    logInfo("Launching executor " + exec.fullId + " on worker " + worker.id)
    worker.addExecutor(exec)
    worker.endpoint.send(LaunchExecutor(masterUrl,
      exec.application.id, exec.id, exec.application.desc, exec.cores, exec.memory))
    exec.application.driver.send(
      ExecutorAdded(exec.id, worker.id, worker.hostPort, exec.cores, exec.memory))
  }
```

> Worker收到RPC请求后，会执行如下代码，会创建ExecutorRunner对象并执行start方法

```scala
case LaunchExecutor(masterUrl, appId, execId, appDesc, cores_, memory_) =>
      if (masterUrl != activeMasterUrl) {
        logWarning("Invalid Master (" + masterUrl + ") attempted to launch executor.")
      } else {
        try {
          logInfo("Asked to launch executor %s/%d for %s".format(appId, execId, appDesc.name))

          // Create the executor's working directory
          val executorDir = new File(workDir, appId + "/" + execId)
          if (!executorDir.mkdirs()) {
            throw new IOException("Failed to create directory " + executorDir)
          }

          // Create local dirs for the executor. These are passed to the executor via the
          // SPARK_EXECUTOR_DIRS environment variable, and deleted by the Worker when the
          // application finishes.
          val appLocalDirs = appDirectories.getOrElse(appId, {
            val localRootDirs = Utils.getOrCreateLocalRootDirs(conf)
            val dirs = localRootDirs.flatMap { dir =>
              try {
                val appDir = Utils.createDirectory(dir, namePrefix = "executor")
                Utils.chmod700(appDir)
                Some(appDir.getAbsolutePath())
              } catch {
                case e: IOException =>
                  logWarning(s"${e.getMessage}. Ignoring this directory.")
                  None
              }
            }.toSeq
            if (dirs.isEmpty) {
              throw new IOException("No subfolder can be created in " +
                s"${localRootDirs.mkString(",")}.")
            }
            dirs
          })
          appDirectories(appId) = appLocalDirs
          val manager = new ExecutorRunner(
            appId,
            execId,
            appDesc.copy(command = Worker.maybeUpdateSSLSettings(appDesc.command, conf)),
            cores_,
            memory_,
            self,
            workerId,
            host,
            webUi.boundPort,
            publicAddress,
            sparkHome,
            executorDir,
            workerUri,
            conf,
            appLocalDirs, ExecutorState.RUNNING)
          executors(appId + "/" + execId) = manager
          manager.start()
          coresUsed += cores_
          memoryUsed += memory_
          sendToMaster(ExecutorStateChanged(appId, execId, manager.state, None, None))
        } catch {
          case e: Exception =>
            logError(s"Failed to launch executor $appId/$execId for ${appDesc.name}.", e)
            if (executors.contains(appId + "/" + execId)) {
              executors(appId + "/" + execId).kill()
              executors -= appId + "/" + execId
            }
            sendToMaster(ExecutorStateChanged(appId, execId, ExecutorState.FAILED,
              Some(e.toString), None))
        }
      }
```

> ExecutorRunner的start方法内容如下，会调用fetchAndRunExecutor方法，该方法会拼接生成启动executor进程的命令，并使用一些底层的技术启动进程，就不细看进去了

```scala
 private[worker] def start() {
    workerThread = new Thread("ExecutorRunner for " + fullId) {
      override def run() { fetchAndRunExecutor() }
    }
    workerThread.start()
    // Shutdown hook that kills actors on shutdown.
    shutdownHook = ShutdownHookManager.addShutdownHook { () =>
      // It's possible that we arrive here before calling `fetchAndRunExecutor`, then `state` will
      // be `ExecutorState.RUNNING`. In this case, we should set `state` to `FAILED`.
      if (state == ExecutorState.RUNNING) {
        state = ExecutorState.FAILED
      }
      killProcess(Some("Worker shutting down")) }
  }
  
  private def fetchAndRunExecutor() {
    try {
      // Launch the process
      val builder = CommandUtils.buildProcessBuilder(appDesc.command, new SecurityManager(conf),
        memory, sparkHome.getAbsolutePath, substituteVariables)
      val command = builder.command()
      val formattedCommand = command.asScala.mkString("\"", "\" \"", "\"")
      logInfo(s"Launch command: $formattedCommand")

      builder.directory(executorDir)
      builder.environment.put("SPARK_EXECUTOR_DIRS", appLocalDirs.mkString(File.pathSeparator))
      // In case we are running this from within the Spark Shell, avoid creating a "scala"
      // parent process for the executor command
      builder.environment.put("SPARK_LAUNCH_WITH_SCALA", "0")

      // Add webUI log urls
      val baseUrl =
        if (conf.getBoolean("spark.ui.reverseProxy", false)) {
          s"/proxy/$workerId/logPage/?appId=$appId&executorId=$execId&logType="
        } else {
          s"http://$publicAddress:$webUiPort/logPage/?appId=$appId&executorId=$execId&logType="
        }
      builder.environment.put("SPARK_LOG_URL_STDERR", s"${baseUrl}stderr")
      builder.environment.put("SPARK_LOG_URL_STDOUT", s"${baseUrl}stdout")

      process = builder.start()
      val header = "Spark Executor Command: %s\n%s\n\n".format(
        formattedCommand, "=" * 40)

      // Redirect its stdout and stderr to files
      val stdout = new File(executorDir, "stdout")
      stdoutAppender = FileAppender(process.getInputStream, stdout, conf)

      val stderr = new File(executorDir, "stderr")
      Files.write(header, stderr, StandardCharsets.UTF_8)
      stderrAppender = FileAppender(process.getErrorStream, stderr, conf)

      // Wait for it to exit; executor may exit with code 0 (when driver instructs it to shutdown)
      // or with nonzero exit code
      val exitCode = process.waitFor()
      state = ExecutorState.EXITED
      val message = "Command exited with code " + exitCode
      worker.send(ExecutorStateChanged(appId, execId, state, Some(message), Some(exitCode)))
    } catch {
      case interrupted: InterruptedException =>
        logInfo("Runner thread for executor " + fullId + " interrupted")
        state = ExecutorState.KILLED
        killProcess(None)
      case e: Exception =>
        logError("Error running executor", e)
        state = ExecutorState.FAILED
        killProcess(Some(e.toString))
    }
  }
```

> driver的话也是类似，worker收到LaunchDriver的消息启动Driver，只不过使用的是DriverRunner，start方法中会执行prepareAndRunDriver方法

```scala
case LaunchDriver(driverId, driverDesc) =>
      logInfo(s"Asked to launch driver $driverId")
      val driver = new DriverRunner(
        conf,
        driverId,
        workDir,
        sparkHome,
        driverDesc.copy(command = Worker.maybeUpdateSSLSettings(driverDesc.command, conf)),
        self,
        workerUri,
        securityMgr)
      drivers(driverId) = driver
      driver.start()

      coresUsed += driverDesc.cores
      memoryUsed += driverDesc.mem
```

> prepareAndRunDriver方法如下，driver需要下载user jar，这点executor不需要，接着也是拼接启动进程参数，执行到最后也是使用ProcessImpl启动进程

```scala
private[worker] def prepareAndRunDriver(): Int = {
    val driverDir = createWorkingDirectory()
    val localJarFilename = downloadUserJar(driverDir)

    def substituteVariables(argument: String): String = argument match {
      case "{{WORKER_URL}}" => workerUrl
      case "{{USER_JAR}}" => localJarFilename
      case other => other
    }

    // TODO: If we add ability to submit multiple jars they should also be added here
    val builder = CommandUtils.buildProcessBuilder(driverDesc.command, securityManager,
      driverDesc.mem, sparkHome.getAbsolutePath, substituteVariables)

    runDriver(builder, driverDir, driverDesc.supervise)
  }
  
  private def runDriver(builder: ProcessBuilder, baseDir: File, supervise: Boolean): Int = {
    builder.directory(baseDir)
    def initialize(process: Process): Unit = {
      // Redirect stdout and stderr to files
      val stdout = new File(baseDir, "stdout")
      CommandUtils.redirectStream(process.getInputStream, stdout)

      val stderr = new File(baseDir, "stderr")
      val formattedCommand = builder.command.asScala.mkString("\"", "\" \"", "\"")
      val header = "Launch Command: %s\n%s\n\n".format(formattedCommand, "=" * 40)
      Files.append(header, stderr, StandardCharsets.UTF_8)
      CommandUtils.redirectStream(process.getErrorStream, stderr)
    }
    runCommandWithRetry(ProcessBuilderLike(builder), initialize, supervise)
  }
```

> 其实看源码方法中很多东西不用看的，比如参数校验，spark代码那么多，如果细细的看，恐怕有得看，学会了这一招，就快了，主要是熟悉整个的流程以及学习思想