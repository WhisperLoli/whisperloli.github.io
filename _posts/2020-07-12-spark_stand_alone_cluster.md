---
layout: post
title:  "Spark源码学习笔记(十九)"
date:   2020-07-12 21:14:13 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070803.webp'
subtitle: SparkSubmit之StandAlone模式启动流程
---
> StandAlone中cluster和client模式启动类是不一样的，在SparkSubmit类中可以看到，

```scala
private[deploy] val REST_CLUSTER_SUBMIT_CLASS = classOf[RestSubmissionClientApp].getName()

if (args.isStandaloneCluster) {
  if (args.useRest) {
    childMainClass = REST_CLUSTER_SUBMIT_CLASS
    childArgs += (args.primaryResource, args.mainClass)
  } else {
    // In legacy standalone cluster mode, use Client as a wrapper around the user class
    childMainClass = STANDALONE_CLUSTER_SUBMIT_CLASS
    if (args.supervise) { childArgs += "--supervise" }
    Option(args.driverMemory).foreach { m => childArgs += ("--memory", m) }
    Option(args.driverCores).foreach { c => childArgs += ("--cores", c) }
    childArgs += "launch"
    childArgs += (args.master, args.primaryResource, args.mainClass)
  }
  if (args.childArgs != null) {
    childArgs ++= args.childArgs
  }
}
```

> 提交后执行start方法，我看到这段代码时突然懵了，这创建完endpoint和endpointRef后就没了，amazing，那怎么启动呢，想了一会，就明白了，创建endpointRef后会自动存入onstart消息，触发onStart方法，Master的onStart可以做一些初始化工作

```scala
private[spark] class ClientApp extends SparkApplication {

  override def start(args: Array[String], conf: SparkConf): Unit = {
    val driverArgs = new ClientArguments(args)

    if (!conf.contains("spark.rpc.askTimeout")) {
      conf.set("spark.rpc.askTimeout", "10s")
    }
    Logger.getRootLogger.setLevel(driverArgs.logLevel)

    val rpcEnv =
      RpcEnv.create("driverClient", Utils.localHostName(), 0, conf, new SecurityManager(conf))

    val masterEndpoints = driverArgs.masters.map(RpcAddress.fromSparkURL).
      map(rpcEnv.setupEndpointRef(_, Master.ENDPOINT_NAME))
    rpcEnv.setupEndpoint("client", new ClientEndpoint(rpcEnv, driverArgs, masterEndpoints, conf))

    rpcEnv.awaitTermination()
  }

}
```

> ClientEndpoint的onStart方法会调用MasterEndpointRef发送RequestSubmitDriver消息给Master的Endpoint

```scala
override def onStart(): Unit = {
  driverArgs.cmd match {
    case "launch" =>
    // TODO: We could add an env variable here and intercept it in `sc.addJar` that would
    //       truncate filesystem paths similar to what YARN does. For now, we just require
    //       people call `addJar` assuming the jar is in the same directory.
    val mainClass = "org.apache.spark.deploy.worker.DriverWrapper"

    val classPathConf = "spark.driver.extraClassPath"
    val classPathEntries = sys.props.get(classPathConf).toSeq.flatMap { cp =>
      cp.split(java.io.File.pathSeparator)
    }

    val libraryPathConf = "spark.driver.extraLibraryPath"
    val libraryPathEntries = sys.props.get(libraryPathConf).toSeq.flatMap { cp =>
      cp.split(java.io.File.pathSeparator)
    }

    val extraJavaOptsConf = "spark.driver.extraJavaOptions"
    val extraJavaOpts = sys.props.get(extraJavaOptsConf)
      .map(Utils.splitCommandString).getOrElse(Seq.empty)
    val sparkJavaOpts = Utils.sparkJavaOpts(conf)
    val javaOpts = sparkJavaOpts ++ extraJavaOpts
    val command = new Command(mainClass,
      Seq("{{WORKER_URL}}", "{{USER_JAR}}", driverArgs.mainClass) ++ driverArgs.driverOptions,
      sys.env, classPathEntries, libraryPathEntries, javaOpts)

    val driverDescription = new DriverDescription(
      driverArgs.jarUrl,
      driverArgs.memory,
      driverArgs.cores,
      driverArgs.supervise,
      command)
    asyncSendToMasterAndForwardReply[SubmitDriverResponse](
      RequestSubmitDriver(driverDescription))

  case "kill" =>
    val driverId = driverArgs.driverId
    asyncSendToMasterAndForwardReply[KillDriverResponse](RequestKillDriver(driverId))
    }
}

private def asyncSendToMasterAndForwardReply[T: ClassTag](message: Any): Unit = {
  for (masterEndpoint <- masterEndpoints) {
    masterEndpoint.ask[T](message).onComplete {
      case Success(v) => self.send(v)
      case Failure(e) =>
        logWarning(s"Error sending messages to master   $masterEndpoint", e)
    }(forwardMessageExecutionContext)
  }
}  
  
```

> createDriver(description)方法并不会创建driver，只是生成DriverInfo类，里面包含了driver的一些信息，真正执行的还是shedule方法

```scala
case RequestSubmitDriver(description) =>
  if (state != RecoveryState.ALIVE) {
    val msg = s"${Utils.BACKUP_STANDALONE_MASTER_PREFIX}: $state. " +
      "Can only accept driver submissions in ALIVE state."
    context.reply(SubmitDriverResponse(self, false, None, msg))
  } else {
    logInfo("Driver submitted " + description.command.mainClass)
    val driver = createDriver(description)
    persistenceEngine.addDriver(driver)
    waitingDrivers += driver
    drivers.add(driver)
    schedule()

    // TODO: It might be good to instead have the submission client poll the master to determine
    //       the current status of the driver. For now it's simply "fire and forget".

    context.reply(SubmitDriverResponse(self, true, Some(driver.id),
      s"Driver successfully submitted as ${driver.id}"))
  }
```

> shedule种两个方法，一个是launchDriver，另一个是startExecutorsOnWorkers，看名字也就知道含义，前者启动driver，后者用于启动executors

```scala
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

> launchDriver中使用work的endpointRef发送LaunchDriver消息到WorkEndpoint中

```scala
private def launchDriver(worker: WorkerInfo, driver: DriverInfo) {
    logInfo("Launching driver " + driver.id + " on worker " + worker.id)
    worker.addDriver(driver)
    driver.worker = Some(worker)
    worker.endpoint.send(LaunchDriver(driver.id, driver.desc))
    driver.state = DriverState.RUNNING
  }
```

> LaunchDriver使用会创建DriverRunner对象，并调用start方法

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

> DriverRunner的start方法运行到最后，会调用命令创建系统process

```scala
/** Starts a thread to run and manage the driver. */
  private[worker] def start() = {
    new Thread("DriverRunner for " + driverId) {
      override def run() {
        var shutdownHook: AnyRef = null
        try {
          shutdownHook = ShutdownHookManager.addShutdownHook { () =>
            logInfo(s"Worker shutting down, killing driver $driverId")
            kill()
          }

          // prepare driver jars and run driver
          val exitCode = prepareAndRunDriver()

          // set final state depending on if forcibly killed and process exit code
          finalState = if (exitCode == 0) {
            Some(DriverState.FINISHED)
          } else if (killed) {
            Some(DriverState.KILLED)
          } else {
            Some(DriverState.FAILED)
          }
        } catch {
          case e: Exception =>
            kill()
            finalState = Some(DriverState.ERROR)
            finalException = Some(e)
        } finally {
          if (shutdownHook != null) {
            ShutdownHookManager.removeShutdownHook(shutdownHook)
          }
        }

        // notify worker of final driver state, possible exception
        worker.send(DriverStateChanged(driverId, finalState.get, finalException))
      }
    }.start()
  }
```

> 启动executor也类似呗，先分配资源，再使用命令创建executor