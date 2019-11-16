---
title: 从scala-repl到spark-shell
tags:
- spark
categories:
- spark, scala, spark-repl, scala-repl, spark-shell
---
# 前言
最近在思考，如何将spark-shell的交互方式变为spark-web。即用户在页面输入scala代码，spark实时运行，并将结果展示在web页面上。这样可以让有编程能力的数据处理员更方便的使用spark。为了实现这个目的就需要了解spark-shell的实现原理，想办法控制其数据入输出流。  
本文主要通过源码介绍了spark-shell是如何时实现的，其中scala-repl其实现的基础。本文所有spark原码参照V2.3.2版本。

# spark-shell
spark-shell提供了一种交互式使用spark的方式。用户只要运行`./bin/spark-shell`就可以进入spark-shell中，在该环境中系统默认为用户创建了一个`SparkContext`，用户可以直接使用scala语言控制这个`sc`。下面我们先看下`spark-shell`。
```shell script
....
"${SPARK_HOME}"/bin/spark-submit --class org.apache.spark.repl.Main --name "Spark shell" "$@"
....

```
可以看出spark-shell其实是用`spark-submit`提交了一个任务，这个任务的主函数是`org.apache.spark.repl.Main`，它在名为`spark-repl`的module中。原码如下
```scala
object Main extends Logging {
  ………………
  val conf = new SparkConf()

  var sparkContext: SparkContext = _
  var sparkSession: SparkSession = _
  // this is a public var because tests reset it.
  var interp: SparkILoop = _

  def main(args: Array[String]) {
    doMain(args, new SparkILoop)
  }

  // Visible for testing
  private[repl] def doMain(args: Array[String], _interp: SparkILoop): Unit = {
  // 核心  
  interp = _interp
  // 设置classpath
    val jars = Utils.getLocalUserJarsForShell(conf)
      // Remove file:///, file:// or file:/ scheme if exists for each jar
      .map { x => if (x.startsWith("file:")) new File(new URI(x)).getPath else x }
      .mkString(File.pathSeparator)
  // 拼接启动参数
    val interpArguments = List(
      "-Yrepl-class-based",
      "-Yrepl-outdir", s"${outputDir.getAbsolutePath}",
      "-classpath", jars
    ) ++ args.toList
  // scala-shell通用参数 拼接用户自定义参数
    val settings = new GenericRunnerSettings(scalaOptionError)
    settings.processArguments(interpArguments, true)

    if (!hasErrors) {
      // 启动 scala-repl
      interp.process(settings) // Repl starts and goes in loop of R.E.P.L
      Option(sparkContext).foreach(_.stop)
    }
  }
  …………
}
```
从源码可以看出启动spark-shell时主要就是创建并启动了一个`SparkILoop`，启动时设置了spark所需要的`classpath`。下面我们看先这`SparkILoop`。
```scala
/**
 *  A Spark-specific interactive shell.
 */
class SparkILoop(in0: Option[BufferedReader], out: JPrintWriter)
    extends ILoop(in0, out) {
  // sparkILoop主要是继承了scala的ILoop，它需要1个输入流，输出流，使用控制台输出 输入可以
  def this(in0: BufferedReader, out: JPrintWriter) = this(Some(in0), out)
  def this() = this(None, new JPrintWriter(Console.out, true))

  val initializationCommands: Seq[String] = Seq(
    """
    @transient val spark = if (org.apache.spark.repl.Main.sparkSession != null) {
        org.apache.spark.repl.Main.sparkSession
      } else {
        org.apache.spark.repl.Main.createSparkSession()
      }
    @transient val sc = {
      val _sc = spark.sparkContext
      if (_sc.getConf.getBoolean("spark.ui.reverseProxy", false)) {
        val proxyUrl = _sc.getConf.get("spark.ui.reverseProxyUrl", null)
        if (proxyUrl != null) {
          println(
            s"Spark Context Web UI is available at ${proxyUrl}/proxy/${_sc.applicationId}")
        } else {
          println(s"Spark Context Web UI is available at Spark Master Public URL")
        }
      } else {
        _sc.uiWebUrl.foreach {
          webUrl => println(s"Spark context Web UI available at ${webUrl}")
        }
      }
      println("Spark context available as 'sc' " +
        s"(master = ${_sc.master}, app id = ${_sc.applicationId}).")
      println("Spark session available as 'spark'.")
      _sc
    }
    """,
    "import org.apache.spark.SparkContext._",
    "import spark.implicits._",
    "import spark.sql",
    "import org.apache.spark.sql.functions._"
  )

  def initializeSpark() {
    intp.beQuietDuring {
      savingReplayStack { // remove the commands from session history.
        initializationCommands.foreach(command)
      }
    }
  }

  /** Print a welcome message */
  override def printWelcome() {
    import org.apache.spark.SPARK_VERSION
    echo("""Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version %s
      /_/
         """.format(SPARK_VERSION))
    val welcomeMsg = "Using Scala %s (%s, Java %s)".format(
      versionString, javaVmName, javaVersion)
    echo(welcomeMsg)
    echo("Type in expressions to have them evaluated.")
    echo("Type :help for more information.")
  }

  /** Available commands */
  override def commands: List[LoopCommand] = standardCommands

  /**
   * We override `createInterpreter` because we need to initialize Spark *before* the REPL
   * sees any files, so that the Spark context is visible in those files. This is a bit of a
   * hack, but there isn't another hook available to us at this point.
   */
  override def createInterpreter(): Unit = {
    super.createInterpreter()
    initializeSpark()
  }

  override def resetCommand(line: String): Unit = {
    super.resetCommand(line)
    initializeSpark()
    echo("Note that after :reset, state of SparkSession and SparkContext is unchanged.")
  }

  override def replay(): Unit = {
    initializeSpark()
    super.replay()
  }

}

object SparkILoop {

  /**
   * Creates an interpreter loop with default settings and feeds
   * the given code to it as input.
   */
  def run(code: String, sets: Settings = new Settings): String = {
    import java.io.{ BufferedReader, StringReader, OutputStreamWriter }

    stringFromStream { ostream =>
      Console.withOut(ostream) {
        val input = new BufferedReader(new StringReader(code))
        val output = new JPrintWriter(new OutputStreamWriter(ostream), true)
        val repl = new SparkILoop(input, output)

        if (sets.classpath.isDefault) {
          sets.classpath.value = sys.props("java.class.path")
        }
        repl process sets
      }
    }
  }
  def run(lines: List[String]): String = run(lines.map(_ + "\n").mkString)
}

```



