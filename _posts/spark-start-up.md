---
title: spark程序初始化
categories:
- spark
tags:
- spark启动
---
# 前言
本文主要是依据源码介绍spark启动的过程，即初始化`sparkContext`的过程。本文会从`spark-submit.sh`脚本开始，介绍到`sparkContext`初始化完成。在此之后就是借助`sparkContxt`执行各种算子得到计算结果的过程。本文使用的spark版本是1.6

# 概述
回想我们写spark程序的基本流程，首先我们会用scala写一个包含spark application的jar包。然后编写任务提交脚本，这个脚本一般是运行spark自带`spark-sumbit`脚本，并传入各种参数。这些参数主要包括运行该spark应用所需要的计算资源，应用jar包位置以及spark应用的`main()`方法。而我们写的spark应用程序一般会先创建用于配置运行环境的`sparkConf`对象，然后用该对象创建`sparkContext`对象。至此spark计算环境的初始化就算完成了。然后就是运行`sparkConext`读取数据，执行算子最后保存结果的过程了。  


下面我们从启动脚本入手，看看spark是如何初始化的。

# spark-submit脚本
当我们完成spark Application的开发后，我们可以通过`bin/spark-submit`脚本来发布我们的应用。使用该脚本主要负责明确spark应用及应用依赖的路径，spark集群管理器的模式以及spark应用的部署方式。`spark-submit`主要所需参数如下：
```ecma script level 4
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
```

`spark-submit`脚本的主要内容是寻找`SPARK_HOME`所在位置以及运行如下命令：
```ecma script level 4
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```

不难看出该明命令是执行`bin/spark-class`，并传入参数：`org.apache.spark.deploy.SparkSubmit`以及用户运行`spark-submit`传入的参数。

# `spark-class`脚本
下面我们来看看`spark-class`脚本的代码
```ecma script level 4
# 在环境变量中设置SPARK_HOME的位置
if [ -z "${SPARK_HOME}" ]; then
  source "$(dirname "$0")"/find-spark-home
fi

# 运行load-spark-env.sh
. "${SPARK_HOME}"/bin/load-spark-env.sh

# 找到JAVA_HOME的位置
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

# 设置spark依赖JAR包的位置
if [ -d "${SPARK_HOME}/jars" ]; then
  SPARK_JARS_DIR="${SPARK_HOME}/jars"
else
  SPARK_JARS_DIR="${SPARK_HOME}/assembly/target/scala-$SPARK_SCALA_VERSION/jars"
fi

if [ ! -d "$SPARK_JARS_DIR" ] && [ -z "$SPARK_TESTING$SPARK_SQL_TESTING" ]; then
  echo "Failed to find Spark jars directory ($SPARK_JARS_DIR)." 1>&2
  echo "You need to build Spark with the target \"package\" before running this program." 1>&2
  exit 1
else
  LAUNCH_CLASSPATH="$SPARK_JARS_DIR/*"
fi

# 设置$SPARK_PREPEND_CLASSES的情况下则使用如下classpath
if [ -n "$SPARK_PREPEND_CLASSES" ]; then
  LAUNCH_CLASSPATH="${SPARK_HOME}/launcher/target/scala-$SPARK_SCALA_VERSION/classes:$LAUNCH_CLASSPATH"
fi

# 判断是否是测试
if [[ -n "$SPARK_TESTING" ]]; then
  unset YARN_CONF_DIR
  unset HADOOP_CONF_DIR
fi

# 下面这段代码一大段代码的主要用处是执行`org.apache.spark.launcher.Main`并将程序的输出添加到cmd 中，cmd是启动程序最终要执行的命令。
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

# 根据org.apache.spark.launcher.Main最后输出的返回码，判断spark程序是否能正常发布
if ! [[ $LAUNCHER_EXIT_CODE =~ ^[0-9]+$ ]]; then
  echo "${CMD[@]}" | head -n-1 1>&2
  exit 1
fi

if [ $LAUNCHER_EXIT_CODE != 0 ]; then
  exit $LAUNCHER_EXIT_CODE
fi

CMD=("${CMD[@]:0:$LAST}")
# 执行CMD中的命令
exec "${CMD[@]}"

```

根据脚本我们可以知道`spark-class`主要是运行了`org.apache.spark.launcher.Main`并将其输出作为最终运行的命令。详细来说是`spark-class.sh`主要干了以下几个工作：

- 确定SPARK_HOME的位置，然后使用${SPARK_HOME}/bin/load-spark-env.sh脚本加载环境变量。这个脚本最主要加载的环境变量有`SPARK_ENV_LOADED`，`SPARK_CONF_DIR`，`SPARK_SCALA_VERSION`以及最重要的是加载spark配置文件夹`spark-env.sh`中的环境变量(默认就是{SPARK_HOME}/conf/spark-env.sh)。
- 确定JAVA_HOME的位置，因为最终spark发布程序是运行在JVM上的。
- 设置LAUNCH_CLASSPATH，通常是{SPARK_HOME}/jars下全部jar包。如果jars不是文件夹则会找`${SPARK_HOME}/assembly/target/scala-$SPARK_SCALA_VERSION/jars`,这种情况一般适用于手动编译过spark代码。
- 运行 java -Xmx128m -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"，即在jvm虚拟机上运行`org.apache.spark.launcher.Main`,参数是传入`spark-class`的参数即`org.apache.spark.deploy.SparkSubmit`以及用户运行`spark-submit`传入的参数。
- 将上程序运行的输出保存在CMD中，并根据最后的输出判断是否执行spark发布。
- 使用exec执行CMD中的内容。注意exec命令将并不启动新的shell，而是用要被执行命令替换当前的shell进程，并且将老进程的环境清理掉，而且exec命令后的其它命令将不再执行。

# org.apache.spark.launcher.Main
这个类主要的工作是构造真正执行spark发布程序的cmd，同时还会验证各种参数是否合理。根据官方JAVA doc我们可以知道，传入的第一个参数决定了发布的模式，主要分为`spark-submit`和`spark-class`两种。顺着之前的思路可知我们主要使用的`spark-submit`模式，即发布spark Application的模式。另一种是运行spark内部类，比如`start-master.sh`,`start-slave.sh`都会传入相应的内部类并运行。核心代码如下:

```java
class Main {
    public static void main(String[] argsArray) throws Exception {
         List<String> args = new ArrayList<String>(Arrays.asList(argsArray));
         // 取第一个参数
         String className = args.remove(0);
        boolean printLaunchCommand = !isEmpty(System.getenv("SPARK_PRINT_LAUNCH_COMMAND"));
        AbstractCommandBuilder builder;
        // 根据第一个参数判断
        if (className.equals("org.apache.spark.deploy.SparkSubmit")) {
          try {
           // 新建用于构建输出的builder
            builder = new SparkSubmitCommandBuilder(args);
          } catch (IllegalArgumentException e) {
            // ……………………
          }
        } else {
          builder = new SparkClassCommandBuilder(className, args);
        }
    
        Map<String, String> env = new HashMap<String, String>();
        // 是有builder构建 cmd 和 env
        List<String> cmd = builder.buildCommand(env);
        // 根据环境变量打印日志
        if (printLaunchCommand) {
          System.err.println("Spark Command: " + join(" ", cmd));
          System.err.println("========================================");
        }
        // 根据操作系统打印输出
        if (isWindows()) {
          System.out.println(prepareWindowsCommand(cmd, env));
        } else {
          // In bash, use NULL as the arg separator since it cannot be used in an argument.
          // 构建返回字符串并打印输出
          List<String> bashCmd = prepareBashCommand(cmd, env);
          for (String c : bashCmd) {
            System.out.print(c);
            System.out.print('\0');
          }
        }
}
```

通过上述代码我们可以清楚的看出，主函数使用第一个参数作为创建了不同的用于构造输出的builder。我们submit任务构建的builder是`SparkSubmitCommandBuilder`，实例化是传入了用户编写`spark-submit`脚本时自定义的参数。然后使用builder的`buildCommand`方法构建了最终运行的命令cmd以及运行命令的环境变量env。最后使用`prepareBashCommand`方法将cmd和env转化为输出list并打印，该list就是下一步要执行的命令。下面说说`new SparkSubmitCommandBuilder(args)`,`List<String> cmd = builder.buildCommand(env)`,`List<String> bashCmd = prepareBashCommand(cmd, env);`主要干了什么。  

在实例化SparkSubmitCommandBuilder时，主要是使用`OptionParser`类结构化传参（用户在使用`spark-submit`时的自定义参数）格式是否正确，并将这些参数保存在本类对应的变量或`sparkArgs`变量中(handle/handleExtraArgs方法)。实例化比较简单，重点是其`buildCommand`方法，该方法首先区分了pySpark和sparkR，而我们使用java编写spark最终调用的方式是`buildSparkSubmitCommand`,源码如下：
```java
// 加载配置文件中的配置，查看spark-submit是执行spark Application的driver（client模式）还是用于向集群发布app（cluster模式）。如果是client模式，将会修改jvm的参数用driver的配置覆盖
private List<String> buildSparkSubmitCommand(Map<String, String> env) throws IOException {
    
    // 加载配置文件
    Map<String, String> config = getEffectiveConfig();
    // 根据配置判断是client模式还是cluster模式
    boolean isClientMode = isClientMode(config);
    // client模式下可以设置的环境变量
    String extraClassPath = isClientMode ? config.get(SparkLauncher.DRIVER_EXTRA_CLASSPATH) : null;
    // 构建java命令
    List<String> cmd = buildJavaCommand(extraClassPath);
    // Take Thrift Server as daemon
    if (isThriftServer(mainClass)) {
      addOptionString(cmd, System.getenv("SPARK_DAEMON_JAVA_OPTS"));
    }
    // CMD执行命令加入环境变量`SPARK_SUBMIT_OPTS`和`SPARK_JAVA_OPTS`设置的内容
    addOptionString(cmd, System.getenv("SPARK_SUBMIT_OPTS"));
    addOptionString(cmd, System.getenv("SPARK_JAVA_OPTS"));

    if (isClientMode) {
        // 如果是client模式，根据优先级设置driver的内存
      // - explicit configuration (setConf()), which also covers --driver-memory cli argument.
      // - properties file.
      // - SPARK_DRIVER_MEMORY env variable
      // - SPARK_MEM env variable
      // - default value (1g)
      String tsMemory =
        isThriftServer(mainClass) ? System.getenv("SPARK_DAEMON_MEMORY") : null;
      String memory = firstNonEmpty(tsMemory, config.get(SparkLauncher.DRIVER_MEMORY),
        System.getenv("SPARK_DRIVER_MEMORY"), System.getenv("SPARK_MEM"), DEFAULT_MEM);
      // 修改driver的运行内存
      cmd.add("-Xms" + memory);
      cmd.add("-Xmx" + memory);
      addOptionString(cmd, config.get(SparkLauncher.DRIVER_EXTRA_JAVA_OPTIONS));
      mergeEnvPathList(env, getLibPathEnvName(),
        config.get(SparkLauncher.DRIVER_EXTRA_LIBRARY_PATH));
    }
    // 设置永久代最大值
    addPermGenSizeOpt(cmd);
    // spark启动程序的类名
    cmd.add("org.apache.spark.deploy.SparkSubmit");
    // 添加经过验证过的spark参数
    cmd.addAll(buildSparkSubmitArgs());
    return cmd;
  }
```
通过代码我们可以知道该方法首先加载配置文件，配置文件首先会使用用户在启动命中用`--properties-file`设置的配置文件如果没设置则使用{spark_home}/conf/spark-defaults.conf。注意这里是互斥的关系，2个配置文件只会加载一个。同时注意文件中配置的优先级没有用户在执行`spark-submit`时添加的配置的优先级高。然后明确spark的模式是client/cluster。接着使用`buildJavaCommand`构建CMD命令的前几行，在这个方法里主要是在CMD中添加了这样几行命令：
1. {JAVA_HOME}/bin/java
2. 将{SPARK_HOME}/conf/java-opts 文件中的配置加入cmd，该文件主要是对driver(存疑) jvm虚拟机运行环境的配置。
3. -cp {ClassPath} ，classpath主要来源有环境变量中`SPARK_CLASSPATH`的配置，client模式下`--driver-class-path`配置的环境变量，{spark_home}/conf下的文件以及{JAVA_HOME}/core/target/jars/*，这些jars是编译过spark-core源码时会出现。


在CMD中加入了JVM参数以及环境变量后会将环境变量中`SPARK_SUBMIT_OPTS`,`SPARK_JAVA_OPTS`设置的字符串直接加入CMD。然后对于client模式，后续将直接启动driver，因此需要在CMD中设置driver的内存（-Xms和-Xmx），内存来源优先级如下：
1. explicit configuration (setConf()) ,它也会覆盖用户在使用`spark-submit`配置的`--driver-memory`参数。(thriftServer会涉及到？)
2. 用户在使用`spark-submit`配置的`--driver-memory`参数
3. 配置文件中的spark.driver.memory的配置
4. 环境变量中`SPARK_DRIVER_MEMORY`的配置
5. 环境变量中`SPARK_MEM`的配置
6. 默认1G


接着会验证CMD中有没有对永久代（Permanate generation）最大值的设置（XX:MaxPermSize），若果没则添加`-XX:MaxPermSize=256m"`。然后就是最关键的在CMD中添加下一个运行的类名——`org.apache.spark.deploy.SparkSubmit`。最后是添加验证过的spark参数，这些参数主要是运行`spark-submit`时用户添加的参数。值得注意的是这里构造CMD并没有将{SPARK_CONF}/conf/spark-defaults.conf的参数将入CMD。构建完的List如下：

![cmdList示例](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/cmdList.png)

本类的最后就使用`prepareBashCommand(cmd,evn)`生成输出的List（脚本之后运行的内容）并输出。该方法会判断env是否为空，如果不为空则返回List的首先添加env的内容然后再添加cmd的内容。当模式是client时env会添加的driver所需环境变量。如果env为空输出直接就是cmd。  
 
 
为了更好的学习本类的运行流程，可以借助Spark官方的测试用例`SparkSubmitCommandBuilderSuite`类中`testDriverCmdBuilder`方法。
 
# org.apache.spark.deploy.SparkSubmit
根据`org.apache.spark.launcher.Main`构建的运行命令，我们得知接下来运行的类是`org.apache.spark.deploy.SparkSubmit`，主要传参依然是我们使用`spark-submit`时定义的参数。下面我们来看下`SparkSubmit`的源码：
```scala
  def main(args: Array[String]): Unit = {
  // 标准化传参
    val appArgs = new SparkSubmitArguments(args)
    if (appArgs.verbose) {
      // scalastyle:off println
      printStream.println(appArgs)
      // scalastyle:on println
    }
    appArgs.action match {
    // 执行相应函数
      case SparkSubmitAction.SUBMIT => submit(appArgs)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
    }
  }
```

通过源码我们可以清晰的看出，本类主要就干了2件事：结构化传参，根据传参的action调用相应函数，我们本次主要关注的操作是`submit`。`SparkSubmitArguments`类的主要作用是解析并封装`spark-submit`脚本传入的参数。部分源码如下：
```scala
private[deploy] class SparkSubmitArguments(args: Seq[String], env: Map[String, String] = sys.env)
  extends SparkSubmitArgumentsParser { //本质是继承SparkSubmitOptionParser
  var master: String = null
  var deployMode: String = null
  var executorMemory: String = null
  var executorCores: String = null
  // ………………一堆属性………………

  try {
  // 结构化spark-submit中用户的传参传参
    parse(args.asJava)
  } catch {
    case e: IllegalArgumentException =>
      SparkSubmit.printErrorAndExit(e.getMessage())
  }
  // 从属性性文件读取内容填充sparkproperties:Map
  mergeDefaultSparkProperties()
  // 删除sparkproperties中key不是以"spark."开头的属性
  ignoreNonSparkProperties()
  // 使用`sparkProperties` 或 evn来填充缺少的必须参数。
  loadEnvironmentArguments()
  // 验证参数
  validateArguments()
  
  // ………………一堆私有方法………………
}
```

首先`SparkSubmitArguments`类是继承自`SparkSubmitArgumentsParser`，其又是`SparkSubmitOptionParser`的子类，而`SparkSubmitOptionParser`又派生了实例化`SparkSubmitCommandBuilder`时主要是使用的`OptionParser`。所以`SparkSubmitArguments`中调用的`parse()`和`SparkSubmitCommandBuilder`初始化时处理参数的方式其实是同一个，都是结构化`spark-submit`中用户的传参。
接下来主要是构造`sparkproperties`这个Map对象，这个Map对象取值的**最高优先级是`spark-submit`中用户定义的`--conf`中的参数，然后读取`--properties-file`所指向文件中的配置，如果`spark-submit`中没配置`--properties-file`才会读取{SPARK_HOME}/conf/spark-defaults.conf中的配置**。这个逻辑和之前验证参数是读取配置的顺序相同。
然后，spark会将`sparkproperties`中不是`spark.`开头的属性移除。**因此可知`--conf`以及spark配置文件中的配置都必须以`spark.`开头**。
再下面就是使用`sparkProperties` 或 evn来填充缺少的重要参数。此时只有在参数为null时，才会首先从`sparkProperties`获取。如果还是为null只有极个别参数会在env中查找。该方法中的参数都比较重要必须掌握理解。
最后是验证参数，根据action（SUBMIT/KILL/REQUEST_STATUS）的不同验证方式也略有不同。其实验证内容并不是很多，以submit操作为例，主要有`mainClass`不能少，如果是`yarn`模式管理集群则必须在环境变量中配置`HADOOP_CONF_DIR`和`YARN_CONF_DIR`  


初始化参数后就是根据action调用相应的方法。下面我们着重看下`submit`方法主要干了啥。
```scala
private def submit(args: SparkSubmitArguments): Unit = {
    val (childArgs, childClasspath, sysProps, childMainClass) = prepareSubmitEnvironment(args)

    def doRunMain(): Unit = {
      if (args.proxyUser != null) {
        val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
          UserGroupInformation.getCurrentUser())
        try {
          proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
            override def run(): Unit = {
              runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
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
        runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
      }
    }

     // In standalone cluster mode, there are two submission gateways:
     //   (1) The traditional Akka gateway using o.a.s.deploy.Client as a wrapper
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
          submit(args)
      }
    // In all other modes, just run the main class as prepared
    } else {
      doRunMain()
    }
  }
```