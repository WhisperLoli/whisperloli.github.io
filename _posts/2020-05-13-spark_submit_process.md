---
layout: post
title:  "Spark源码学习笔记（二）"
date:   2020-05-13 22:35:21 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2019_05_06_10.webp'
subtitle: 'spark-submit命令揭秘'
---
> 上篇讲完spark-shell，那么这篇就讲讲spark-submit吧，spark-shell其实在shell脚本中也是执行的spark-submit命令，指定\--class参数为org.apache.spark.repl.Main
> 
> ![](/blog/spark_source_code/source_code_2/spark_submit_shell.jpg)
> 从spark-submit脚本中，可以看到，底层执行的是spark-class命令，SparkSubmit被当作主类传进去，exec命令的意思也是执行，用被执行的命令行替换掉当前的shell进程，且exec命令后的其他命令将不再执行
> 
> 进入SparkSubmit类，查找主函数
> 

```scala
override def main(args: Array[String]): Unit = {
  // Initialize logging if it hasn't been done yet. Keep track of   whether logging needs to
  // be reset before the application starts.
  val uninitLog = initializeLogIfNecessary(true, silent = true)
  val appArgs = new SparkSubmitArguments(args)
  if (appArgs.verbose) {
    // scalastyle:off println
    printStream.println(appArgs)
    // scalastyle:on println
  }
  appArgs.action match {
    case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
    case SparkSubmitAction.KILL => kill(appArgs)
    case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
  }
}
  
```
 
> 主函数中会创建SparkSubmitArguments对象，这个类主要的作用就是较验用户使用spark-submit提交的参数，最后封装到对象中，进入该类后，会执行parse(args.asJava)方法，解析spark-sumit后面跟着的参数，正则匹配参数中如果带有=并且是--开头的话，则会解析成key-value键值对，如果参数中没有=，则会进行关键字匹配，例如--driver-memory，代码中已经枚举了所有的关键key，循环较验参数，正则没匹配上的参数再进行循环匹配，如果正则全都没匹配上的话，时间复杂度就是O(n2)，这段代码是java写的，可能是多个开发者协同开发吧，个人感觉使用scala match匹配会更好些，内层循环中如果匹配上了关键词，则会将下一个参数也取进来当成value，如果没匹配上，则会执行handleUnknown方法，判断当前参数是否为提交的jar资源，如果是jar资源则跳出参数解析循环，剩下的参数被传递进入handleExtraArgs方法，最后当成用户提交的jar的main class参数
> 
> 参数解析完后调用mergeDefaultSparkProperties合并重复的配置参数，接着执行ignoreNonSparkProperties忽略无效的配置参数，无效的参数主要就是非spark开头的参数，该类参数会被忽略掉，项目中如果使用conf配置自定义参数时可以避坑，接着加载参数给类中的属性赋值，action默认为submit，如果指定的话可以为kill，status，用了这么久的spark-submit，一直以为只有提交任务用，没想到还可以查看任务状态、kill任务。
> 
```scala
if (mainClass == null && !isPython && !isR && primaryResource != null) {
  val uri = new URI(primaryResource)
  val uriScheme = uri.getScheme()
  uriScheme match {
    case "file" =>
      try {
        Utils.tryWithResource(new JarFile(uri.getPath)) { jar =>
          // Note that this might still return null if no main-class is set; we catch that later
          mainClass = jar.getManifest.getMainAttributes.getValue("Main-Class")
        }
      } catch {
        case _: Exception =>
          SparkSubmit.printErrorAndExit(s"Cannot load main class from JAR $primaryResource")
      }
    case _ =>
      SparkSubmit.printErrorAndExit(
        s"Cannot load main class from JAR $primaryResource with URI $uriScheme. " +
        "Please specify a class through --class.")
  }
}
```
提交的jar的主类参数如果不存在，则会从打包时pom生成的文件中获取指定的main class，最后还要执行validateArguments方法验证一遍参数。

> 参数解析较验完后，接着会根据args.action匹配是submit还是kill或者status操作

```scala
@tailrec
private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
  val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)

  def doRunMain(): Unit = {
    if (args.proxyUser != null) {
      val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
        UserGroupInformation.getCurrentUser())
    try {
      proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
        override def run(): Unit = {
          runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
        }
      })
    } catch {
      case e: Exception =>
        // Hadoop's AuthorizationException suppresses the exception's stack trace, which
        // makes the message printed to the output by the JVM not very helpful. Instead,
        // detect exceptions with empty stack traces here, and treat them differently.
        if (e.getStackTrace().length == 0) {
          // scalastyle:off println
          printStream.println(s"ERROR: ${e.getClass().getName()}: ${e.getMessage()}")
          // scalastyle:on println
          exitFn(1)
        } else {
          throw e
        }
    }
  } else {
      runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
    }
  }

  // Let the main class re-initialize the logging system once it starts.
  if (uninitLog) {
    Logging.uninitialize()
  }

  // In standalone cluster mode, there are two submission gateways:
  //   (1) The traditional RPC gateway using o.a.s.deploy.Client as a wrapper
  //   (2) The new REST-based gateway introduced in Spark 1.3
  // The latter is the default behavior as of Spark 1.3, but Spark submit will fail over
  // to use the legacy gateway if the master endpoint turns out to be not a REST server.
  if (args.isStandaloneCluster && args.useRest) {
    try {
      // scalastyle:off println
      printStream.println("Running Spark using the REST application submission protocol.")
      // scalastyle:on println
      doRunMain()
    } catch {
      // Fail over to use the legacy submission gateway
      case e: SubmitRestConnectionException =>
        printWarning(s"Master endpoint ${args.master} was not a REST server. " +
          "Falling back to legacy submission gateway instead.")
        args.useRest = false
        submit(args, false)
    }
  // In all other modes, just run the main class as prepared
  } else {
  	doRunMain()
  }
}

```
> 进入submit方法后会执行prepareSubmitEnvironment，该方法执行到下面，会判断当前执行模式，基于yarn,k8s,mesos等，不同的模式会提供不同的操作，提供各自的较验，当然所有的配置会写入sparkConf对象中，再后续会执行DependencyUtils.resolveMavenDependencies方法，解决maven中的jar依赖问题，生成一些排除规则去除一些jar。如下部分jar就会被排除

```scala
val IVY_DEFAULT_EXCLUDES = Seq("catalyst_", "core_", "graphx_", "kvstore_", "launcher_", "mllib_",
    "mllib-local_", "network-common_", "network-shuffle_", "repl_", "sketch_", "sql_", "streaming_",
    "tags_", "unsafe_")
    
/** Add exclusion rules for dependencies already included in the spark-assembly */
def addExclusionRules(
  ivySettings: IvySettings,
  ivyConfName: String,
  md: DefaultModuleDescriptor): Unit = {
// Add scala exclusion rule
md.addExcludeRule(createExclusion("*:scala-library:*", ivySettings, ivyConfName))

IVY_DEFAULT_EXCLUDES.foreach { comp =>
  md.addExcludeRule(createExclusion(s"org.apache.spark:spark-$comp*:*", ivySettings,
    ivyConfName))
}
```

> 获取到返回值后，从doRunMain进入runMain方法，准备执行用户提交的main函数，创建SparkApplication对象

```scala
val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
  mainClass.newInstance().asInstanceOf[SparkApplication]
} else {
  // SPARK-4170
  if (classOf[scala.App].isAssignableFrom(mainClass)) {
    printWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
  }
  new JavaMainApplication(mainClass)
}
```
> SparkApplication是trait类型，如果SparkApplicaiton是mainClass的父类，则直接通过mainClass创建实例，否则创建类JavaMainApplication实例，将mainClass当作参数传入

```scala
@tailrec
def findCause(t: Throwable): Throwable = t match {
  case e: UndeclaredThrowableException =>
    if (e.getCause() != null) findCause(e.getCause()) else e
  case e: InvocationTargetException =>
    if (e.getCause() != null) findCause(e.getCause()) else e
  case e: Throwable =>
    e
}

try {
  app.start(childArgs.toArray, sparkConf)
} catch {
  case t: Throwable =>
    findCause(t) match {
      case SparkUserAppException(exitCode) =>
        System.exit(exitCode)

      case t: Throwable =>
        throw t
    }
}
```
> 调用start方法，这个地方用到了多态，如果调用的是JavaMainApplication类型的start方法，如下，则会通过反射执行main方法，并将sparkConf对象中的配置写入SystemProperties中
> 

```scala
private[deploy] class JavaMainApplication(klass: Class[_]) extends SparkApplication {
  override def start(args: Array[String], conf: SparkConf): Unit =   {
    val mainMethod = klass.getMethod("main", new Array[String](0).getClass)
    if (!Modifier.isStatic(mainMethod.getModifiers)) {
      throw new IllegalStateException("The main method in the given 	main class must be static")
    }
    val sysProps = conf.getAll.toMap
	   sysProps.foreach { case (k, v) =>
  	   sys.props(k) = v
    }
    mainMethod.invoke(null, args)
  }
}

```

> 源码中涉及到很多面向对象知识，有些知识点都忘了，看了源码，满脑子都是继承、封装、多态，对于面向对象很熟的同学，看源码应该会容易很多
