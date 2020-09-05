---
layout: post
title:  "Spark源码学习笔记(十八)"
date:   2020-07-11 14:27:13 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070802.webp'
subtitle: SparkSubmit之Yarn模式启动流程
---
> 之前说 [SparkSubmit](https://whisperloli.github.io/2020/05/13/spark_submit_process.html) 的时候，只是大致讲了下整个流程，prepareSubmitEnvironment没有细讲，该方法中会判断当前是以什么模式启动，不同的启动模式，主函数和其他配置都会变化，如下所示，如果是yarn cluster模式，spark-submit最后执行的主函数就会是org.apache.spark.deploy.yarn.YarnClusterApplication，而不是我们自己打在jar包中的主函数，如果是client模式的话，依然是我们自己提供的主函数

```scala
    private[deploy] val YARN_CLUSTER_SUBMIT_CLASS =
    "org.apache.spark.deploy.yarn.YarnClusterApplication"

    // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
    if (isYarnCluster) {
      childMainClass = YARN_CLUSTER_SUBMIT_CLASS
      if (args.isPython) {
        childArgs += ("--primary-py-file", args.primaryResource)
        childArgs += ("--class", "org.apache.spark.deploy.PythonRunner")
      } else if (args.isR) {
        val mainFile = new Path(args.primaryResource).getName
        childArgs += ("--primary-r-file", mainFile)
        childArgs += ("--class", "org.apache.spark.deploy.RRunner")
      } else {
        if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
          childArgs += ("--jar", args.primaryResource)
        }
        childArgs += ("--class", args.mainClass)
      }
      if (args.childArgs != null) {
        args.childArgs.foreach { arg => childArgs += ("--arg", arg) }
      }
    }
```

> 找到执行的主函数，就很好就判断，先看yarn-cluster模式吧，启动后创建client调用run方法

```
private[spark] class YarnClusterApplication extends SparkApplication {

  override def start(args: Array[String], conf: SparkConf): Unit = {
    // SparkSubmit would use yarn cache to distribute files & jars in yarn mode,
    // so remove them from sparkConf here for yarn mode.
    conf.remove("spark.jars")
    conf.remove("spark.files")

    new Client(new ClientArguments(args), conf).run()
  }

}
```

> run方法那么多东西，其实主要就是看第一行代码，其他都是些什么校验啊之类的

```scala
def run(): Unit = {
    this.appId = submitApplication()
    if (!launcherBackend.isConnected() && fireAndForget) {
      val report = getApplicationReport(appId)
      val state = report.getYarnApplicationState
      logInfo(s"Application report for $appId (state: $state)")
      logInfo(formatReportDetails(report))
      if (state == YarnApplicationState.FAILED || state == YarnApplicationState.KILLED) {
        throw new SparkException(s"Application $appId finished with status: $state")
      }
    } else {
      val (yarnApplicationState, finalApplicationStatus) = monitorApplication(appId)
      if (yarnApplicationState == YarnApplicationState.FAILED ||
        finalApplicationStatus == FinalApplicationStatus.FAILED) {
        throw new SparkException(s"Application $appId finished with failed status")
      }
      if (yarnApplicationState == YarnApplicationState.KILLED ||
        finalApplicationStatus == FinalApplicationStatus.KILLED) {
        throw new SparkException(s"Application $appId is killed")
      }
      if (finalApplicationStatus == FinalApplicationStatus.UNDEFINED) {
        throw new SparkException(s"The final status of application $appId is undefined")
      }
    }
  }
```

> submitApplication方法主要用于提交application，创建Container启动上下文，application提交上下文，并提交到yarn中。container启动上下文就是准备一些jvm参数，用户的配置之类用于启动ApplicationMaster。application提交上下文就是设置一些sparkConf中配置的参数，比如我们设置yarn的队列之类的，spark app name等等

```
/**
   * Submit an application running our ApplicationMaster to the ResourceManager.
   *
   * The stable Yarn API provides a convenience method (YarnClient#createApplication) for
   * creating applications and setting up the application submission context. This was not
   * available in the alpha API.
   */
  def submitApplication(): ApplicationId = {
    var appId: ApplicationId = null
    try {
      launcherBackend.connect()
      // Setup the credentials before doing anything else,
      // so we have don't have issues at any point.
      setupCredentials()
      yarnClient.init(hadoopConf)
      yarnClient.start()

      logInfo("Requesting a new application from cluster with %d NodeManagers"
        .format(yarnClient.getYarnClusterMetrics.getNumNodeManagers))

      // Get a new application from our RM
      val newApp = yarnClient.createApplication()
      val newAppResponse = newApp.getNewApplicationResponse()
      appId = newAppResponse.getApplicationId()

      new CallerContext("CLIENT", sparkConf.get(APP_CALLER_CONTEXT),
        Option(appId.toString)).setCurrentContext()

      // Verify whether the cluster has enough resources for our AM
      verifyClusterResources(newAppResponse)

      // Set up the appropriate contexts to launch our AM
      val containerContext = createContainerLaunchContext(newAppResponse)
      val appContext = createApplicationSubmissionContext(newApp, containerContext)

      // Finally, submit and monitor the application
      logInfo(s"Submitting application $appId to ResourceManager")
      yarnClient.submitApplication(appContext)
      launcherBackend.setAppId(appId.toString)
      reportLauncherState(SparkAppHandle.State.SUBMITTED)

      appId
    } catch {
      case e: Throwable =>
        if (appId != null) {
          cleanupStagingDir(appId)
        }
        throw e
    }
  }
```

> 提交完后，yarn会启动ApplicationMaster，执行ApplicationMaster的run方法，主要就是runDriver方法和runExecutorLauncher方法，cluster模式要执行runDriver主要是因为spark-submit提交后并没有执行用户的代码，所以没有启动driver，而client模式提交后，是先执行的用户代码，driver已经启动了，再来将任务提交到yarn中，申请executor。这也就是为什么client模式提交代码driver就会创建在哪台机器上，而cluster模式不会。

```scala
  final def run(): Int = {
    doAsUser {
      runImpl()
    }
    exitCode
  }

  private def runImpl(): Unit = {
    try {
      val appAttemptId = client.getAttemptId()

      var attemptID: Option[String] = None

      if (isClusterMode) {
        // Set the web ui port to be ephemeral for yarn so we don't conflict with
        // other spark processes running on the same box
        System.setProperty("spark.ui.port", "0")

        // Set the master and deploy mode property to match the requested mode.
        System.setProperty("spark.master", "yarn")
        System.setProperty("spark.submit.deployMode", "cluster")

        // Set this internal configuration if it is running on cluster mode, this
        // configuration will be checked in SparkContext to avoid misuse of yarn cluster mode.
        System.setProperty("spark.yarn.app.id", appAttemptId.getApplicationId().toString())

        attemptID = Option(appAttemptId.getAttemptId.toString)
      }

      new CallerContext(
        "APPMASTER", sparkConf.get(APP_CALLER_CONTEXT),
        Option(appAttemptId.getApplicationId.toString), attemptID).setCurrentContext()

      logInfo("ApplicationAttemptId: " + appAttemptId)

      // This shutdown hook should run *after* the SparkContext is shut down.
      val priority = ShutdownHookManager.SPARK_CONTEXT_SHUTDOWN_PRIORITY - 1
      ShutdownHookManager.addShutdownHook(priority) { () =>
        val maxAppAttempts = client.getMaxRegAttempts(sparkConf, yarnConf)
        val isLastAttempt = client.getAttemptId().getAttemptId() >= maxAppAttempts

        if (!finished) {
          // The default state of ApplicationMaster is failed if it is invoked by shut down hook.
          // This behavior is different compared to 1.x version.
          // If user application is exited ahead of time by calling System.exit(N), here mark
          // this application as failed with EXIT_EARLY. For a good shutdown, user shouldn't call
          // System.exit(0) to terminate the application.
          finish(finalStatus,
            ApplicationMaster.EXIT_EARLY,
            "Shutdown hook called before final status was reported.")
        }

        if (!unregistered) {
          // we only want to unregister if we don't want the RM to retry
          if (finalStatus == FinalApplicationStatus.SUCCEEDED || isLastAttempt) {
            unregister(finalStatus, finalMsg)
            cleanupStagingDir()
          }
        }
      }

      // If the credentials file config is present, we must periodically renew tokens. So create
      // a new AMDelegationTokenRenewer
      if (sparkConf.contains(CREDENTIALS_FILE_PATH)) {
        // Start a short-lived thread for AMCredentialRenewer, the only purpose is to set the
        // classloader so that main jar and secondary jars could be used by AMCredentialRenewer.
        val credentialRenewerThread = new Thread {
          setName("AMCredentialRenewerStarter")
          setContextClassLoader(userClassLoader)

          override def run(): Unit = {
            val credentialManager = new YARNHadoopDelegationTokenManager(
              sparkConf,
              yarnConf,
              conf => YarnSparkHadoopUtil.hadoopFSsToAccess(sparkConf, conf))

            val credentialRenewer =
              new AMCredentialRenewer(sparkConf, yarnConf, credentialManager)
            credentialRenewer.scheduleLoginFromKeytab()
          }
        }

        credentialRenewerThread.start()
        credentialRenewerThread.join()
      }

      if (isClusterMode) {
        runDriver()
      } else {
        runExecutorLauncher()
      }
    } catch {
      case e: Exception =>
        // catch everything else if not specifically handled
        logError("Uncaught exception: ", e)
        finish(FinalApplicationStatus.FAILED,
          ApplicationMaster.EXIT_UNCAUGHT_EXCEPTION,
          "Uncaught exception: " + e)
    }
  }
```

> runDriver方法会先调用startUserApplication方法提交执行用户的代码，然后自己的代码会被阻塞等待用户代码创建完TaskScheduler，SparkContext创建taskScheduler时使用ClusterManager创建，yarn模式会进入YarnClusterManager，YarnClusterScheduler会调用ApplicationMaster.sparkContextInitialized(sc)方法，发送通知给runDriver，至此driver创建完成，runDriver方法会继续向下执行，会执行registerAM方法注册ApplicationMaster（也就是AM）

```scala
private def runDriver(): Unit = {
    addAmIpFilter(None)
    userClassThread = startUserApplication()

    // This a bit hacky, but we need to wait until the spark.driver.port property has
    // been set by the Thread executing the user class.
    logInfo("Waiting for spark context initialization...")
    val totalWaitTime = sparkConf.get(AM_MAX_WAIT_TIME)
    try {
      val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
        Duration(totalWaitTime, TimeUnit.MILLISECONDS))
      if (sc != null) {
        rpcEnv = sc.env.rpcEnv
        val driverRef = createSchedulerRef(
          sc.getConf.get("spark.driver.host"),
          sc.getConf.get("spark.driver.port"))
        registerAM(sc.getConf, rpcEnv, driverRef, sc.ui.map(_.webUrl))
        registered = true
      } else {
        // Sanity check; should never happen in normal operation, since sc should only be null
        // if the user app did not create a SparkContext.
        throw new IllegalStateException("User did not initialize spark context!")
      }
      resumeDriver()
      userClassThread.join()
    } catch {
      case e: SparkException if e.getCause().isInstanceOf[TimeoutException] =>
        logError(
          s"SparkContext did not initialize after waiting for $totalWaitTime ms. " +
           "Please check earlier log output for errors. Failing the application.")
        finish(FinalApplicationStatus.FAILED,
          ApplicationMaster.EXIT_SC_NOT_INITED,
          "Timed out waiting for SparkContext.")
    } finally {
      resumeDriver()
    }
  }
```

> 注册完AM后会分配资源，执行allocator.allocateResources()方法

```scala
private def registerAM(
      _sparkConf: SparkConf,
      _rpcEnv: RpcEnv,
      driverRef: RpcEndpointRef,
      uiAddress: Option[String]) = {
    val appId = client.getAttemptId().getApplicationId().toString()
    val attemptId = client.getAttemptId().getAttemptId().toString()
    val historyAddress = ApplicationMaster
      .getHistoryServerAddress(_sparkConf, yarnConf, appId, attemptId)

    val driverUrl = RpcEndpointAddress(
      _sparkConf.get("spark.driver.host"),
      _sparkConf.get("spark.driver.port").toInt,
      CoarseGrainedSchedulerBackend.ENDPOINT_NAME).toString

    // Before we initialize the allocator, let's log the information about how executors will
    // be run up front, to avoid printing this out for every single executor being launched.
    // Use placeholders for information that changes such as executor IDs.
    logInfo {
      val executorMemory = sparkConf.get(EXECUTOR_MEMORY).toInt
      val executorCores = sparkConf.get(EXECUTOR_CORES)
      val dummyRunner = new ExecutorRunnable(None, yarnConf, sparkConf, driverUrl, "<executorId>",
        "<hostname>", executorMemory, executorCores, appId, securityMgr, localResources)
      dummyRunner.launchContextDebugInfo()
    }

    allocator = client.register(driverUrl,
      driverRef,
      yarnConf,
      _sparkConf,
      uiAddress,
      historyAddress,
      securityMgr,
      localResources)

    // Initialize the AM endpoint *after* the allocator has been initialized. This ensures
    // that when the driver sends an initial executor request (e.g. after an AM restart),
    // the allocator is ready to service requests.
    rpcEnv.setupEndpoint("YarnAM", new AMEndpoint(rpcEnv, driverRef))

    allocator.allocateResources()
    reporterThread = launchReporterThread()
  }
```

> allocateResources方法获取yarn分配给我们的资源，如果所有资源都可以获取，那么会启动containers数量与max executors数量一致，也就是一个containers中一个executor，之后执行handleAllocatedContainers方法。分配container时的策略是同一个host的优先，接下来再是同一个机架，最后是同一个机房。再执行runAllocatedContainers方法

```scala
def allocateResources(): Unit = synchronized {
    updateResourceRequests()

    val progressIndicator = 0.1f
    // Poll the ResourceManager. This doubles as a heartbeat if there are no pending container
    // requests.
    val allocateResponse = amClient.allocate(progressIndicator)

    val allocatedContainers = allocateResponse.getAllocatedContainers()

    if (allocatedContainers.size > 0) {
      logDebug(("Allocated containers: %d. Current executor count: %d. " +
        "Launching executor count: %d. Cluster resources: %s.")
        .format(
          allocatedContainers.size,
          runningExecutors.size,
          numExecutorsStarting.get,
          allocateResponse.getAvailableResources))

      handleAllocatedContainers(allocatedContainers.asScala)
    }

    val completedContainers = allocateResponse.getCompletedContainersStatuses()
    if (completedContainers.size > 0) {
      logDebug("Completed %d containers".format(completedContainers.size))
      processCompletedContainers(completedContainers.asScala)
      logDebug("Finished processing %d completed containers. Current running executor count: %d."
        .format(completedContainers.size, runningExecutors.size))
    }
  }
  
  def handleAllocatedContainers(allocatedContainers: Seq[Container]): Unit = {
    val containersToUse = new ArrayBuffer[Container](allocatedContainers.size)

    // Match incoming requests by host
    val remainingAfterHostMatches = new ArrayBuffer[Container]
    for (allocatedContainer <- allocatedContainers) {
      matchContainerToRequest(allocatedContainer, allocatedContainer.getNodeId.getHost,
        containersToUse, remainingAfterHostMatches)
    }

    // Match remaining by rack
    val remainingAfterRackMatches = new ArrayBuffer[Container]
    for (allocatedContainer <- remainingAfterHostMatches) {
      val rack = resolver.resolve(conf, allocatedContainer.getNodeId.getHost)
      matchContainerToRequest(allocatedContainer, rack, containersToUse,
        remainingAfterRackMatches)
    }

    // Assign remaining that are neither node-local nor rack-local
    val remainingAfterOffRackMatches = new ArrayBuffer[Container]
    for (allocatedContainer <- remainingAfterRackMatches) {
      matchContainerToRequest(allocatedContainer, ANY_HOST, containersToUse,
        remainingAfterOffRackMatches)
    }

    if (!remainingAfterOffRackMatches.isEmpty) {
      logDebug(s"Releasing ${remainingAfterOffRackMatches.size} unneeded containers that were " +
        s"allocated to us")
      for (container <- remainingAfterOffRackMatches) {
        internalReleaseContainer(container)
      }
    }

    runAllocatedContainers(containersToUse)

    logInfo("Received %d containers from YARN, launching executors on %d of them."
      .format(allocatedContainers.size, containersToUse.size))
  }
```

> runAllocatedContainers方法用于启动executor，代码如下

```scala
private def runAllocatedContainers(containersToUse: ArrayBuffer[Container]): Unit = {
    for (container <- containersToUse) {
      executorIdCounter += 1
      val executorHostname = container.getNodeId.getHost
      val containerId = container.getId
      val executorId = executorIdCounter.toString
      assert(container.getResource.getMemory >= resource.getMemory)
      logInfo(s"Launching container $containerId on host $executorHostname " +
        s"for executor with ID $executorId")

      def updateInternalState(): Unit = synchronized {
        runningExecutors.add(executorId)
        numExecutorsStarting.decrementAndGet()
        executorIdToContainer(executorId) = container
        containerIdToExecutorId(container.getId) = executorId

        val containerSet = allocatedHostToContainersMap.getOrElseUpdate(executorHostname,
          new HashSet[ContainerId])
        containerSet += containerId
        allocatedContainerToHostMap.put(containerId, executorHostname)
      }

      if (runningExecutors.size() < targetNumExecutors) {
        numExecutorsStarting.incrementAndGet()
        if (launchContainers) {
          launcherPool.execute(new Runnable {
            override def run(): Unit = {
              try {
                new ExecutorRunnable(
                  Some(container),
                  conf,
                  sparkConf,
                  driverUrl,
                  executorId,
                  executorHostname,
                  executorMemory,
                  executorCores,
                  appAttemptId.getApplicationId.toString,
                  securityMgr,
                  localResources
                ).run()
                updateInternalState()
              } catch {
                case e: Throwable =>
                  numExecutorsStarting.decrementAndGet()
                  if (NonFatal(e)) {
                    logError(s"Failed to launch executor $executorId on container $containerId", e)
                    // Assigned container should be released immediately
                    // to avoid unnecessary resource occupation.
                    amClient.releaseAssignedContainer(containerId)
                  } else {
                    throw e
                  }
              }
            }
          })
        } else {
          // For test only
          updateInternalState()
        }
      } else {
        logInfo(("Skip launching executorRunnable as running executors count: %d " +
          "reached target executors count: %d.").format(
          runningExecutors.size, targetNumExecutors))
      }
    }
  }
```

> 再来说说client模式启动流程吧，client模式会先执行用户代码，进入YarnClusterManager中，创建SchedulerBackend会进入YarnClientSchedulerBackend中

```scala
override def createSchedulerBackend(sc: SparkContext,
      masterURL: String,
      scheduler: TaskScheduler): SchedulerBackend = {
    sc.deployMode match {
      case "cluster" =>
        new YarnClusterSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
      case "client" =>
        new YarnClientSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
      case  _ =>
        throw new SparkException(s"Unknown deploy mode '${sc.deployMode}' for Yarn")
    }
  }
```
> YarnClientSchedulerBackend的start方法也会执行client.submitApplication()，提交到yarn中，接下来就和cluster模式差不多了，创建AM，再执行runExecutorLauncher方法

```scala
/**
   * Create a Yarn client to submit an application to the ResourceManager.
   * This waits until the application is running.
   */
  override def start() {
    val driverHost = conf.get("spark.driver.host")
    val driverPort = conf.get("spark.driver.port")
    val hostport = driverHost + ":" + driverPort
    sc.ui.foreach { ui => conf.set("spark.driver.appUIAddress", ui.webUrl) }

    val argsArrayBuf = new ArrayBuffer[String]()
    argsArrayBuf += ("--arg", hostport)

    logDebug("ClientArguments called with: " + argsArrayBuf.mkString(" "))
    val args = new ClientArguments(argsArrayBuf.toArray)
    totalExpectedExecutors = SchedulerBackendUtils.getInitialTargetExecutorNumber(conf)
    client = new Client(args, conf)
    // 提交到yarn中
    bindToYarn(client.submitApplication(), None)

    // SPARK-8687: Ensure all necessary properties have already been set before
    // we initialize our driver scheduler backend, which serves these properties
    // to the executors
    super.start()
    waitForApplication()

    // SPARK-8851: In yarn-client mode, the AM still does the credentials refresh. The driver
    // reads the credentials from HDFS, just like the executors and updates its own credentials
    // cache.
    if (conf.contains("spark.yarn.credentials.file")) {
      YarnSparkHadoopUtil.startCredentialUpdater(conf)
    }
    monitorThread = asyncMonitorApplication()
    monitorThread.start()
  }
```

> runExecutorLauncher方法中也会使用到registerAM方法，注册完AM就开始分配资源，申请Containers，启动executors

```scala
private def runExecutorLauncher(): Unit = {
    val hostname = Utils.localHostName
    val amCores = sparkConf.get(AM_CORES)
    rpcEnv = RpcEnv.create("sparkYarnAM", hostname, hostname, -1, sparkConf, securityMgr,
      amCores, true)
    val driverRef = waitForSparkDriver()
    addAmIpFilter(Some(driverRef))
    registerAM(sparkConf, rpcEnv, driverRef, sparkConf.getOption("spark.driver.appUIAddress"))
    registered = true

    // In client mode the actor will stop the reporter thread.
    reporterThread.join()
  }
```

> 讲到这就差不多了，自己看代码的话yarn这块确实花了好长时间，因为一开始漏了spark-submit yarn-cluster模式启动函数会变，后来找到这块，就好理解很多