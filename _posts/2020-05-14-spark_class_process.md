---
layout: post
title:  "Spark源码学习笔记（三）"
date:   2020-05-14 23:49:08 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_16.webp'
subtitle: 'spark-class命令学习'
---
> spark-submit命令底层使用的是spark-class命令，那么spark-class又做了哪些操作呢？

```bash
# Find the java binary
if [ -n "${JAVA_HOME}" ]; then
  RUNNER="${JAVA_HOME}/bin/java"
else
  if [ "$(command -v java)" ]; then
    RUNNER="java"
  else
    echo "JAVA_HOME is not set" >&2
    exit 1
  fi
fi

build_command() {
  "$RUNNER" -Xmx128m -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"
  printf "%d\0" $?
}

# Turn off posix mode since it does not allow process substitution
set +o posix
CMD=()
while IFS= read -d '' -r ARG; do
  CMD+=("$ARG")
done < <(build_command "$@")

COUNT=${#CMD[@]}
LAST=$((COUNT - 1))
LAUNCHER_EXIT_CODE=${CMD[$LAST]}

# Certain JVM failures result in errors being printed to stdout (instead of stderr), which causes
# the code that parses the output of the launcher to get confused. In those cases, check if the
# exit code is an integer, and if it's not, handle it as a special error case.
if ! [[ $LAUNCHER_EXIT_CODE =~ ^[0-9]+$ ]]; then
  echo "${CMD[@]}" | head -n-1 1>&2
  exit 1
fi

if [ $LAUNCHER_EXIT_CODE != 0 ]; then
  exit $LAUNCHER_EXIT_CODE
fi

CMD=("${CMD[@]:0:$LAST}")
exec "${CMD[@]}"
```
> 分析脚本可以知道build_command方法会启动java程序，脚本通过循环读取build\_command的输出给CMD变量，最后通过exec命令执行CMD变量里的内容

![](/blog/spark_source_code/source_code_3/spark_launcher_main.jpg)
>进入主函数，代码中会获取第一个参数判断是不是SparkSubmit操作，如果不是SparkSubmit，则会创建SparkClassCommandBuilder对象，SparkSubmitCommandBuilder和SparkClassCommandBuilder均继承自抽象类AbstractCommandBuilder，重写父类中的buildCommand方法
>

```java
private List<String> buildSparkSubmitCommand(Map<String, String> env)
      throws IOException, IllegalArgumentException {
    // Load the properties file and check whether spark-submit will be running the app's driver
    // or just launching a cluster app. When running the driver, the JVM's argument will be
    // modified to cover the driver's configuration.
    Map<String, String> config = getEffectiveConfig();
    boolean isClientMode = isClientMode(config);
    String extraClassPath = isClientMode ? config.get(SparkLauncher.DRIVER_EXTRA_CLASSPATH) : null;
    List<String> cmd = buildJavaCommand(extraClassPath);
    // Take Thrift Server as daemon
    if (isThriftServer(mainClass)) {
      addOptionString(cmd, System.getenv("SPARK_DAEMON_JAVA_OPTS"));
    }
    addOptionString(cmd, System.getenv("SPARK_SUBMIT_OPTS"));
    // We don't want the client to specify Xmx. These have to be set by their corresponding
    // memory flag --driver-memory or configuration entry spark.driver.memory
    String driverExtraJavaOptions = config.get(SparkLauncher.DRIVER_EXTRA_JAVA_OPTIONS);
    if (!isEmpty(driverExtraJavaOptions) && driverExtraJavaOptions.contains("Xmx")) {
      String msg = String.format("Not allowed to specify max heap(Xmx) memory settings through " +
                   "java options (was %s). Use the corresponding --driver-memory or " +
                   "spark.driver.memory configuration instead.", driverExtraJavaOptions);
      throw new IllegalArgumentException(msg);
    }
    if (isClientMode) {
      // Figuring out where the memory value come from is a little tricky due to precedence.
      // Precedence is observed in the following order:
      // - explicit configuration (setConf()), which also covers --driver-memory cli argument.
      // - properties file.
      // - SPARK_DRIVER_MEMORY env variable
      // - SPARK_MEM env variable
      // - default value (1g)
      // Take Thrift Server as daemon
      String tsMemory =
        isThriftServer(mainClass) ? System.getenv("SPARK_DAEMON_MEMORY") : null;
      String memory = firstNonEmpty(tsMemory, config.get(SparkLauncher.DRIVER_MEMORY),
        System.getenv("SPARK_DRIVER_MEMORY"), System.getenv("SPARK_MEM"), DEFAULT_MEM);
      cmd.add("-Xmx" + memory);
      addOptionString(cmd, driverExtraJavaOptions);
      mergeEnvPathList(env, getLibPathEnvName(),
        config.get(SparkLauncher.DRIVER_EXTRA_LIBRARY_PATH));
    }
    cmd.add("org.apache.spark.deploy.SparkSubmit");
    cmd.addAll(buildSparkSubmitArgs());
    return cmd;
}
```

> 其中buildJavaCommand(extraClassPath)会获取JAVA_HOME路径，添加到cmd中，后面会执行cmd.add("org.apache.spark.deploy.SparkSubmit")，以及添加spark-submit参数进入cmd中，最后打印出来返回给shell，最后拼接的cmd变量完整结果类似用java命令执行jar，然后配齐参数，利用shell直接运行启动JVM
> 
> 说到这里，有人可能就会有疑问，用命令行就可以直接启动JVM运行org.apache.spark.deploy.SparkSubmit，为什么还要做一层spark-class呢，最终效果都是是java命令行启动JVM，不用急，我们还有参数mainClass不是org.apache.spark.deploy.SparkSubmit的情况呢
> 

```java
 @Override
  public List<String> buildCommand(Map<String, String> env)
      throws IOException, IllegalArgumentException {
    List<String> javaOptsKeys = new ArrayList<>();
    String memKey = null;
    String extraClassPath = null;

    // Master, Worker, HistoryServer, ExternalShuffleService, MesosClusterDispatcher use
    // SPARK_DAEMON_JAVA_OPTS (and specific opts) + SPARK_DAEMON_MEMORY.
    switch (className) {
      case "org.apache.spark.deploy.master.Master":
        javaOptsKeys.add("SPARK_DAEMON_JAVA_OPTS");
        javaOptsKeys.add("SPARK_MASTER_OPTS");
        extraClassPath = getenv("SPARK_DAEMON_CLASSPATH");
        memKey = "SPARK_DAEMON_MEMORY";
        break;
      case "org.apache.spark.deploy.worker.Worker":
        javaOptsKeys.add("SPARK_DAEMON_JAVA_OPTS");
        javaOptsKeys.add("SPARK_WORKER_OPTS");
        extraClassPath = getenv("SPARK_DAEMON_CLASSPATH");
        memKey = "SPARK_DAEMON_MEMORY";
        break;
      case "org.apache.spark.deploy.history.HistoryServer":
        javaOptsKeys.add("SPARK_DAEMON_JAVA_OPTS");
        javaOptsKeys.add("SPARK_HISTORY_OPTS");
        extraClassPath = getenv("SPARK_DAEMON_CLASSPATH");
        memKey = "SPARK_DAEMON_MEMORY";
        break;
      case "org.apache.spark.executor.CoarseGrainedExecutorBackend":
        javaOptsKeys.add("SPARK_EXECUTOR_OPTS");
        memKey = "SPARK_EXECUTOR_MEMORY";
        extraClassPath = getenv("SPARK_EXECUTOR_CLASSPATH");
        break;
      case "org.apache.spark.executor.MesosExecutorBackend":
        javaOptsKeys.add("SPARK_EXECUTOR_OPTS");
        memKey = "SPARK_EXECUTOR_MEMORY";
        extraClassPath = getenv("SPARK_EXECUTOR_CLASSPATH");
        break;
      case "org.apache.spark.deploy.mesos.MesosClusterDispatcher":
        javaOptsKeys.add("SPARK_DAEMON_JAVA_OPTS");
        extraClassPath = getenv("SPARK_DAEMON_CLASSPATH");
        memKey = "SPARK_DAEMON_MEMORY";
        break;
      case "org.apache.spark.deploy.ExternalShuffleService":
      case "org.apache.spark.deploy.mesos.MesosExternalShuffleService":
        javaOptsKeys.add("SPARK_DAEMON_JAVA_OPTS");
        javaOptsKeys.add("SPARK_SHUFFLE_OPTS");
        extraClassPath = getenv("SPARK_DAEMON_CLASSPATH");
        memKey = "SPARK_DAEMON_MEMORY";
        break;
      default:
        memKey = "SPARK_DRIVER_MEMORY";
        break;
    }

    List<String> cmd = buildJavaCommand(extraClassPath);

    for (String key : javaOptsKeys) {
      String envValue = System.getenv(key);
      if (!isEmpty(envValue) && envValue.contains("Xmx")) {
        String msg = String.format("%s is not allowed to specify max heap(Xmx) memory settings " +
                "(was %s). Use the corresponding configuration instead.", key, envValue);
        throw new IllegalArgumentException(msg);
      }
      addOptionString(cmd, envValue);
    }

    String mem = firstNonEmpty(memKey != null ? System.getenv(memKey) : null, DEFAULT_MEM);
    cmd.add("-Xmx" + mem);
    cmd.add(className);
    cmd.addAll(classArgs);
    return cmd;
  }
```

> 看到这段代码，大家应该就会明白，还会启动Master，Worker呢，SparkClassCommandBuilder类的解释为This class handles building the command to launch all internal Spark classes except for
SparkSubmit，中文含义是构建除了SparkSubmit外的其他spark class启动命令

> 在看spark-class时，CommandBuilderUtils类里面有一段下面的代码，我当时一看，这不就是scala中mkString方法的效果吗，偷偷说一句，scala真香

```java
/** Joins a list of strings using the given separator. */
  static String join(String sep, Iterable<String> elements) {
    StringBuilder sb = new StringBuilder();
    for (String e : elements) {
      if (e != null) {
        if (sb.length() > 0) {
          sb.append(sep);
        }
        sb.append(e);
      }
    }
    return sb.toString();
  }
```


