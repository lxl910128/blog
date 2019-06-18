---
title: spark源码走读————spark初始化
categories:
- spark
tags:
- spark启动
---
# 前言
本文主要是依据源码介绍spark启动的过程，即初始化`sparkContext`的过程。本文会从`spark-submit.sh`脚本开始，介绍到`sparkContext`初始化完成。在此之后就是借助`sparkContxt`执行各种算子得到计算结果的过程。本文使用的spark版本是1.6，主要关注使用java\scala编写spark程序，忽略R语言或python编写应用时的逻辑。
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
// 用已给参数提交spark Application。
// 本方法主要有2步，首先我们需要准备启动环境，主要是根据集群资源和发布模式为下一步运行的主方法准备classpath，环境变量以及参数。
// 第二步 我们用准备的环境借助反射运行主方法。
private def submit(args: SparkSubmitArguments): Unit = {
    // 准备启动Application的参数,主要包括: 传给Application的参数(list),classpath的list,系统属性的map,运行子类
    val (childArgs, childClasspath, sysProps, childMainClass) = prepareSubmitEnvironment(args)
    // 最终都时运行runMain方法()
    // ………………
      runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
    
  }
```

根据源码，首先调用`prepareSubmitEnvironment`方法获得传给`Application`的参数，classpath，系统属性以及子方法的类名。在这个方法里首先确定了集群的管理模式（yarn/standalone/mesos/local）以及部署方式（client/cluster）。
确定全部依赖Jar包。验证目前的模式是否是系统支持的，spark不支持以下模式：
```scala
(clusterManager, deployMode) match {
      case (MESOS, CLUSTER) if args.isR =>
        printErrorAndExit("Cluster deploy mode is currently not supported for R applications on Mesos clusters.")
      case (STANDALONE, CLUSTER) if args.isPython =>
        printErrorAndExit("Cluster deploy mode is currently not supported for python applications on standalone clusters.")
      case (STANDALONE, CLUSTER) if args.isR =>
        printErrorAndExit("Cluster deploy mode is currently not supported for R applications on standalone clusters.")
      case (LOCAL, CLUSTER) =>
        printErrorAndExit("Cluster deploy mode is not compatible with master \"local\"")
      case (_, CLUSTER) if isShell(args.primaryResource) =>
        printErrorAndExit("Cluster deploy mode is not applicable to Spark shells.")
      case (_, CLUSTER) if isSqlShell(args.mainClass) =>
        printErrorAndExit("Cluster deploy mode is not applicable to Spark SQL shell.")
      case (_, CLUSTER) if isThriftServer(args.mainClass) =>
        printErrorAndExit("Cluster deploy mode is not applicable to Spark Thrift server.")
      case _ =>
    }
```
接着就是比较重要的部分了，是针对部署模式和集群管理方式设置子方法的主类传参等。在client模式下子方法主类就是用户`--class`配置的类。如果是`standalone-cluster`模式，在这种模式下又分2种提交方式：1、子方法使用`org.apache.spark.deploy.Client`借助Akka提交; 2、子方法使用`org.apache.spark.deploy.rest.RestSubmissionClient`基于REST client的提交。默认情况下是使用第二种，但是当master节点不是REST服务，则会自动改用第一种模式。如果是yarn-cluster模式则子方法主类是`org.apache.spark.deploy.yarn.Client`。如果是mesos-cluster模式，子方法主类则为`org.apache.spark.deploy.rest.RestSubmissionClient`。这里还会将`spark.yarn.keytab`和`spark.yarn.principal`指向的密钥反在系统变量中，因此如果使用的hadoop有安全验证，着两个变量需要配置。
在确认子方法主类的同时，还会配置子方法所需的传参，classpath，系统属性。最后就是将这些属性返回。  


回到`submit`方法，如果设置了代理启动用户，则使用代理用户运行`runMain`方法，否则直接运行`runMain`方法。在standalone集群模式，如果master不是rest服务，则改用原来akka的提交方式。其实核心就是运行`runMain`方法，源码入下：
```scala
private def runMain(
      childArgs: Seq[String],
      childClasspath: Seq[String],
      sysProps: Map[String, String],
      childMainClass: String,
      verbose: Boolean): Unit = {
    // scalastyle:off println
    if (verbose) {
      printStream.println(s"Main class:\n$childMainClass")
      printStream.println(s"Arguments:\n${childArgs.mkString("\n")}")
      printStream.println(s"System properties:\n${sysProps.mkString("\n")}")
      printStream.println(s"Classpath elements:\n${childClasspath.mkString("\n")}")
      printStream.println("\n")
    }
    // scalastyle:on println
    // 初始化classpath loader
    val loader =
      if (sysProps.getOrElse("spark.driver.userClassPathFirst", "false").toBoolean) {
        new ChildFirstURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      } else {
        new MutableURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      }
    Thread.currentThread.setContextClassLoader(loader)
    // 加载子方法所需classpath
    for (jar <- childClasspath) {
      addJarToClasspath(jar, loader)
    }
    // 设置系统变量
    for ((key, value) <- sysProps) {
      System.setProperty(key, value)
    }

    var mainClass: Class[_] = null
    // 找执行子类
    try {
      mainClass = Utils.classForName(childMainClass)
    } catch {
      case e: ClassNotFoundException =>
        ……
      case e: NoClassDefFoundError =>
        ……
    }

    // SPARK-4170
    if (classOf[scala.App].isAssignableFrom(mainClass)) {
      printWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
    }
    // 从子类中找到子方法，方法名必为`main`
    val mainMethod = mainClass.getMethod("main", new Array[String](0).getClass)
    if (!Modifier.isStatic(mainMethod.getModifiers)) {
      throw new IllegalStateException("The main method in the given main class must be static")
    }

    // 异常处理
    def findCause(t: Throwable): Throwable = t match {
        ………
    }

    try {
    // 借助代理执行子方法。
      mainMethod.invoke(null, childArgs.toArray)
    } catch {
      case t: Throwable =>
        …………
    }
  }
```
该方法比较间单，主要是就加载classpath，设置系统属性，利用反射机制运行子方法。这里看出如果是client模式则该进程接下来运行的就是用户编写sparkApplication。此时该进程的角色就是driver了。而如果是cluster模式，则会调用对应集群的发布方法，在集群其它机器中启动spark Application。

# Spark Application Main()
终于到运行用户自己编写的spark代码的地方了。我们知道在我们开始一个sparkApplication时都要先使用`sparkConf`实例化`sparkContext`。`sparkConf`在实例化时会加载系统属性中所有"spark.*"。实例化后还可以在代码中增加spark配置而且添会替代已有配置，因此在sparkApplication中用sparkConf增加配置优先级时最高的。  

接下来我们来重点研究下`SparkContext`。它是spark功能的主要切入点。通过它使driver与spark集群相联，并且它还负责创建RDD、累加器、广播变量。需要注意每个JVM只能实例化一个SparkContext。如果你要重新启动sparkContext，使用`stop()`停用之前的`sparkContext`。下面我们来看下sparkContext在初始化时主要干的工作。
```scala
try {
    // 获取sparkConf的参数并验证
    _conf = config.clone()
    _conf.validateSettings()
    
    if ((master == "yarn-cluster" || master == "yarn-standalone") &&
            !_conf.contains("spark.yarn.app.id")) {
          throw new SparkException("Detected yarn-cluster mode, but isn't running on a cluster. " +
            "Deployment to YARN is not supported directly by SparkContext. Please use spark-submit.")
        }
    
        if (_conf.getBoolean("spark.logConf", false)) {
          logInfo("Spark configuration:\n" + _conf.toDebugString)
        }
    
        // 设置 Spark driver host and port system properties
        _conf.setIfMissing("spark.driver.host", Utils.localHostName())
        _conf.setIfMissing("spark.driver.port", "0")   // 系统自动分配
    
        _conf.set("spark.executor.id", SparkContext.DRIVER_IDENTIFIER)

    _jars = _conf.getOption("spark.jars").map(_.split(",")).map(_.filter(_.size != 0)).toSeq.flatten
    _files = _conf.getOption("spark.files").map(_.split(",")).map(_.filter(_.size != 0))
      .toSeq.flatten
    // spark.eventLog.dir是记录Spark事件的基本目录，如果spark.eventLog.enabled为true。 在此基本目录中，Spark为每个应用程序创建一个子目录，并在此目录中记录特定于应用程序的事件。 用户可能希望将其设置为统一位置，如HDFS目录，以便历史记录服务器可以读取历史记录文件。
    _eventLogDir =
      if (isEventLogEnabled) {
        val unresolvedDir = conf.get("spark.eventLog.dir", EventLoggingListener.DEFAULT_LOG_DIR)
          .stripSuffix("/")
        Some(Utils.resolveURI(unresolvedDir))
      } else {
        None
      }

    _eventLogCodec = {
      val compress = _conf.getBoolean("spark.eventLog.compress", false)
      if (compress && isEventLogEnabled) {
        Some(CompressionCodec.getCodecName(_conf)).map(CompressionCodec.getShortName)
      } else {
        None
      }
    }

    _conf.set("spark.externalBlockStore.folderName", externalBlockStoreFolderName)

    if (master == "yarn-client") System.setProperty("SPARK_YARN_MODE", "true")

    // 在创建sparkenv之前应该设置“jobProgressListener”，因为在创建“sparkenv”时，一些消息将被发送到“listenerbus”.
    _jobProgressListener = new JobProgressListener(_conf)
    listenerBus.addListener(jobProgressListener)

    // 使用SparkEnv构建spark driver 的环境 (cache, map output tracker, etc)，并设置
    // 该环境变量主要会关注driver的 host port，spark conf和 listenerBus对象
    _env = createSparkEnv(_conf, isLocal, listenerBus)
    SparkEnv.set(_env)
    // 构建metadataCleaner，用于定期清理元数据（比如 老文件或 hashtable）
    _metadataCleaner = new MetadataCleaner(MetadataCleanerType.SPARK_CONTEXT, this.cleanup, _conf)
    // 构建SparkStatusTracker 用于监视job和stage的进度，生成状态报告
    _statusTracker = new SparkStatusTracker(this)
    // progressBar负责在console展示stage进度，它定期从'sc.statustracker'中轮询stage状态
    _progressBar =
      if (_conf.getBoolean("spark.ui.showConsoleProgress", true) && !log.isInfoEnabled) {
        Some(new ConsoleProgressBar(this))
      } else {
        None
      }
      // 用于启动SparkUI
    _ui =
      if (conf.getBoolean("spark.ui.enabled", true)) {
        Some(SparkUI.createLiveUI(this, _conf, listenerBus, _jobProgressListener,
          _env.securityManager, appName, startTime = startTime))
      } else {
        // For tests, do not enable the UI
        None
      }
      //在启动 taskscheduler钱绑定UI，以便将端口正确的连接到cluster manager
    _ui.foreach(_.bind())
    // 读hadoop配置
    _hadoopConfiguration = SparkHadoopUtil.get.newConfiguration(_conf)

    // 将依赖Jar包和file添加到SparkEnv中，以便启动executor时使用
    if (jars != null) {
      jars.foreach(addJar)
    }
    if (files != null) {
      files.foreach(addFile)
    }
    // 配置executor内存
    _executorMemory = _conf.getOption("spark.executor.memory")
      .orElse(Option(System.getenv("SPARK_EXECUTOR_MEMORY")))
      .orElse(Option(System.getenv("SPARK_MEM"))
      .map(warnSparkMem))
      .map(Utils.memoryStringToMb)
      .getOrElse(1024)

    // 构建executor的运行环境，主要是依赖
    for { (envKey, propKey) <- Seq(("SPARK_TESTING", "spark.testing"))
      value <- Option(System.getenv(envKey)).orElse(Option(System.getProperty(propKey)))} {
      executorEnvs(envKey) = value
    }
    Option(System.getenv("SPARK_PREPEND_CLASSES")).foreach { v =>
      executorEnvs("SPARK_PREPEND_CLASSES") = v
    }
    // The Mesos scheduler backend relies on this environment variable to set executor memory.
    // TODO: Set this only in the Mesos scheduler.
    executorEnvs("SPARK_EXECUTOR_MEMORY") = executorMemory + "m"
    executorEnvs ++= _conf.getExecutorEnv
    executorEnvs("SPARK_USER") = sparkUser

    // 在创建scheduler前在SparkEnv中注册HeartbeatReceiver，是负责是executor向driver发送心跳包的
    _heartbeatReceiver = env.rpcEnv.setupEndpoint(
      HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))

    // 创建 taskShceduler，schedulerBackend以及DAGScheduler，这是Sparkcontext最核心的功能
    val (sched, ts) = SparkContext.createTaskScheduler(this, master)
    _schedulerBackend = sched
    _taskScheduler = ts
    _dagScheduler = new DAGScheduler(this)
    _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)

    // 在DAGScheduler添加了taskScheduler的引用后，taskScheduler才可以启动
    _taskScheduler.start()
    // 根据taskScheduler设置各种参数
    _applicationId = _taskScheduler.applicationId()
    _applicationAttemptId = taskScheduler.applicationAttemptId()
    _conf.set("spark.app.id", _applicationId)
    _ui.foreach(_.setAppId(_applicationId))
    _env.blockManager.initialize(_applicationId)

    // The metrics system for Driver need to be set spark.app.id to app ID.
    // So it should start after we get app ID from the task scheduler and set spark.app.id.
    metricsSystem.start()
    // Attach the driver metrics servlet handler to the web ui after the metrics system is started.
    metricsSystem.getServletHandlers.foreach(handler => ui.foreach(_.attachHandler(handler)))

    _eventLogger =
      if (isEventLogEnabled) {
        val logger =
          new EventLoggingListener(_applicationId, _applicationAttemptId, _eventLogDir.get,
            _conf, _hadoopConfiguration)
        logger.start()
        listenerBus.addListener(logger)
        Some(logger)
      } else {
        None
      }

    // Optionally scale number of executors dynamically based on workload. Exposed for testing.
    val dynamicAllocationEnabled = Utils.isDynamicAllocationEnabled(_conf)
    if (!dynamicAllocationEnabled && _conf.getBoolean("spark.dynamicAllocation.enabled", false)) {
      logWarning("Dynamic Allocation and num executors both set, thus dynamic allocation disabled.")
    }

    _executorAllocationManager =
      if (dynamicAllocationEnabled) {
        Some(new ExecutorAllocationManager(this, listenerBus, _conf))
      } else {
        None
      }
    _executorAllocationManager.foreach(_.start())

    _cleaner =
      if (_conf.getBoolean("spark.cleaner.referenceTracking", true)) {
        Some(new ContextCleaner(this))
      } else {
        None
      }
    _cleaner.foreach(_.start())

    setupAndStartListenerBus()
    postEnvironmentUpdate()
    postApplicationStart()

    // Post init
    _taskScheduler.postStartHook()
    _env.metricsSystem.registerSource(_dagScheduler.metricsSource)
    _env.metricsSystem.registerSource(new BlockManagerSource(_env.blockManager))
    _executorAllocationManager.foreach { e =>
      _env.metricsSystem.registerSource(e.executorAllocationManagerSource)
    }

    // Make sure the context is stopped if the user forgets about it. This avoids leaving
    // unfinished event logs around after the JVM exits cleanly. It doesn't help if the JVM
    // is killed, though.
    _shutdownHookRef = ShutdownHookManager.addShutdownHook(
      ShutdownHookManager.SPARK_CONTEXT_SHUTDOWN_PRIORITY) { () =>
      logInfo("Invoking stop() from shutdown hook")
      stop()
    }
  } catch {
    ……………………
  }
```
在sparkContext的构造函数中，最重要的部分是就是初始化taskScheduler，taskSchedulerBackend以及DAGScheduler，以及启动taskScheduler。taskScheduler负责为sparkContext提供task调度。taskSchedulerBackend调度系统的后端接口与具体的集群交互。DAGScheduler主要负责task划分。首先来看`SparkContext.createTaskScheduler`方法是如何根据不同的运行模式创建`SchedulerBackend`和`TaskScheduler`的。这里我们着重`local`模式和`yarn-cluster`模式
```scala
private def createTaskScheduler(
      sc: SparkContext,
      master: String): (SchedulerBackend, TaskScheduler) = {
    import SparkMasterRegex._

    // When running locally, don't try to re-execute tasks on failure.
    val MAX_LOCAL_TASK_FAILURES = 1

    master match {
      case "local" =>
      // 等效于local[1]
        ……………………………………
      // local[*]
      case LOCAL_N_REGEX(threads) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*] 表示根据cpu核数创建线程; local[N] 表示根据用户指定创建N个线程.
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        if (threadCount <= 0) {
          throw new SparkException(s"Asked to run locally with $threadCount threads")
        }
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)

      case LOCAL_N_FAILURES_REGEX(threads, maxFailures) =>
        ………………………………
      // spark standalone
      case SPARK_REGEX(sparkUrl) =>
        ………………………………
      
      //local-cluster
      case LOCAL_CLUSTER_REGEX(numSlaves, coresPerSlave, memoryPerSlave) =>
       ……………………
      // yarn
      case "yarn-standalone" | "yarn-cluster" =>
        if (master == "yarn-standalone") {
          logWarning(
            "\"yarn-standalone\" is deprecated as of Spark 1.0. Use \"yarn-cluster\" instead.")
        }
        val scheduler = try {
          val clazz = Utils.classForName("org.apache.spark.scheduler.cluster.YarnClusterScheduler")
          val cons = clazz.getConstructor(classOf[SparkContext])
          cons.newInstance(sc).asInstanceOf[TaskSchedulerImpl]
        } catch {
          // TODO: Enumerate the exact reasons why it can fail
          // But irrespective of it, it means we cannot proceed !
          case e: Exception => {
            throw new SparkException("YARN mode not available ?", e)
          }
        }
        val backend = try {
          val clazz =
            Utils.classForName("org.apache.spark.scheduler.cluster.YarnClusterSchedulerBackend")
          val cons = clazz.getConstructor(classOf[TaskSchedulerImpl], classOf[SparkContext])
          cons.newInstance(scheduler, sc).asInstanceOf[CoarseGrainedSchedulerBackend]
        } catch {
          case e: Exception => {
            throw new SparkException("YARN mode not available ?", e)
          }
        }
        scheduler.initialize(backend)
        (backend, scheduler)

      case "yarn-client" =>
        ……………………

      case MESOS_REGEX(mesosUrl) =>
       ……………………

      case SIMR_REGEX(simrUrl) =>
        ……………………
      case zkUrl if zkUrl.startsWith("zk://") =>
      ……………………
      case _ =>
        throw new SparkException("Could not parse Master URL: '" + master + "'")
    }
  }
```
在使用local模式的情况下，spark`TaskSchedulerImpl`构建taskScheduler，使用`LocalBackend`创建taskBackend。其实通过源码我们可以发现除了在yarn模式下TaskScheduler使用的是`TaskSchedulerImpl`的子类，其它的都直接使用的是`TaskSchedulerImpl`，即使是mesos（不愧是亲儿子）。而taskSchedulerBackend主要是`SchedulerBackend`的子类。  

`taskSchedulerImpl`，它通过操作底层schedulerBackend，可以达到对不同集群体调度task的效果。它也可以通过使用isLocal参数使用LocalBackend在本地模式下工作(使用线程代替节点上的进程)。`TaskScheduler`最主要的方法是`submitTasks`，提交一组TaskSet让集群运行。`taskScheduler`在初始化(包括调用initialize方法)时，主要的工作是保存SparkContext、schedulerBacked对象，初始化Poor（可调度的实体）、SchedulerBuilder(分FAIR(公平调度)/FIFO(先进先出，默认)两种模式，内部会构建可调度的树)。  

接下来我们看下`LocalBackend`。如果需要executor，backend，master都运行在一个JVM上时可以使用LocalBackend。它处于`TaskSchedulerImpl`的下层，它用于在将task发布到一个本地的executor上。有机会会详细介绍Spark的通信机制。



