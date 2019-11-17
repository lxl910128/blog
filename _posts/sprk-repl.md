---
title: 从scala-repl到spark-shell
tags:
- spark
categories:
- spark, scala, spark-repl, scala-repl, spark-shell
---
# 前言
最近在思考，如何将spark-shell的交互方式变为spark-web。即用户在页面输入scala代码，spark实时运行，并将结果展示在web页面上。这样可以让有编程能力的数据处理员更方便的使用spark。为了实现这个目的就需要了解spark-shell的实现原理，想办法控制其数据入输出流。  
本文主要通过源码介绍了spark-shell是如何时实现的，其中scala-repl是实现的基础。本文spark源码的版本是`spark_2.12:v2.3.2`。
<!--more-->
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
  // 创建sparkSession sparkContext
  def createSparkSession(): SparkSession = {
    ………………
  }
  …………
}
```
从源码可以看出启动spark-shell时主要就是创建并启动了一个`SparkILoop`，启动时设置了spark所需要的`classpath`。下面我们来看下`SparkILoop`。
```scala
/**
 *  A Spark-specific interactive shell.
 */
class SparkILoop(in0: Option[BufferedReader], out: JPrintWriter)
    extends ILoop(in0, out) {
  // sparkILoop主要是继承了scala的ILoop，它需要1个输入流，输出流，使用控制台输出 输入可以
  def this(in0: BufferedReader, out: JPrintWriter) = this(Some(in0), out)
  def this() = this(None, new JPrintWriter(Console.out, true))
  // 初始化spark时运行的代码
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
  // 使用command函数逐条运行 initializationCommands中的代码
  def initializeSpark() {
    intp.beQuietDuring {
      savingReplayStack { // remove the commands from session history.
        initializationCommands.foreach(command)
      }
    }
  }

  /** Print a welcome message */
  // spark 启动时打印的欢迎语
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
  //  shell中可以使用的命令，spark直接借用了 scala中默认的command，比如hlep edit imports等
  override def commands: List[LoopCommand] = standardCommands

  /**
   * We override `createInterpreter` because we need to initialize Spark *before* the REPL
   * sees any files, so that the Spark context is visible in those files. This is a bit of a
   * hack, but there isn't another hook available to us at this point.
   */
   // 覆盖ILoop的 createInterpreter, 在调用ILoop.createInterpreter后立马调用initializeSpark()初始化spark环境
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
// 定义1个static方法 run(code,sets) 应该用于使用sparkILoop运行1次输入的代码
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
        //等价于 repl.process(sets)
        repl process sets
      }
    }
  }
  def run(lines: List[String]): String = run(lines.map(_ + "\n").mkString)
}

```
根据sparkILoop的原码我们可以看出，它主要工作就是继承`ILoop`，定义`ILoop`初始化时需要运行的代码。这些代码保存在`initializationCommands`中，主要用于创建spark运行环境。代码调用`org.apache.spark.repl.Main.createSparkSession()`创建`sparkSession`和`sparkContext`，并将`sparkContext`赋值给变量`sc`，这样用户就可以`spark-shell`中直接使用了。同时代码还引入了spark中常用的隐式转化，spark sql常用的方法。由于scala在2.12版本只有提供了默认的`Interpreter`，所在原码中`createInterpreter()`直接调用父类的对应方法，而在之前的版本，`createInterpreter()`创建的是spark自己实现的`SparkILoopInterpreter`。  

为了更好的了解sparkILoop我们有必要看下scala的ILoop到底是什么样的。

# scala repl
scala repl("Read-Evaluate-Print-Loop")是一个命令行解释器，它提供了一个测试scala代码的环境。ILoop和IMain是其核心实现。ILoop为Interpreter类（用于翻译scala代码）提供了read（读代码）-eval（运行代码）-print（打印代码）的循环环境。IMain是用于将遵守scala代码规范的字符串翻译成可运行的字节码，它的子类就是前文所说的Interpreter类，scala在2.12版本中给出了IMain的默认实现`ILoopInterpreter`。本文重点介绍ILoop，不介绍IMain及其子类的实现。下面是ILoop的核心代码片段
```scala
/** The Scala interactive shell.  It provides a read-eval-print loop
 *  around the Interpreter class.
 *  scala交互shell，它为Interpreter类提供了一个read-eval-print的循环环境
 *  
 *  After instantiation, clients should call the main() method.
 *  在scala shell通过调用main()方法开始提供服务
 *  
 *  If no in0 is specified, then input will come from the console, and
 *  the class will attempt to provide input editing feature such as
 *  input history.
 *  默认情况下input是console, scala shell 并提供了许多方便的功能,比如 历史输入等
 *
 */
class ILoop(in0: Option[BufferedReader], protected val out: JPrintWriter)
                extends AnyRef
                   with LoopCommands
{
  def this(in0: BufferedReader, out: JPrintWriter) = this(Some(in0), out)
  def this() = this(None, new JPrintWriter(Console.out, true))

  @deprecated("Use `intp` instead.", "2.9.0") def interpreter = intp
  @deprecated("Use `intp` instead.", "2.9.0") def interpreter_= (i: Interpreter): Unit = intp = i
  // 默认输入是 console
  var in: InteractiveReader = _   // the input stream from which commands come
  // 配置
  var settings: Settings = _
  // 翻译器
  var intp: IMain = _
  
  /** Standard commands **/
  // scala shell和spark shell 中可以使用的命令
  lazy val standardCommands = List(
    cmd("edit", "<id>|<line>", "edit history", editCommand),
    cmd("help", "[command]", "print this summary or command-specific help", helpCommand),
    historyCommand,
    cmd("h?", "<string>", "search the history", searchHistory),
    cmd("imports", "[name name ...]", "show import history, identifying sources of names", importsCommand),
    cmd("implicits", "[-v]", "show the implicits in scope", intp.implicitsCommand),
    cmd("javap", "<path|class>", "disassemble a file or class name", javapCommand),
    cmd("line", "<id>|<line>", "place line(s) at the end of history", lineCommand),
    cmd("load", "<path>", "interpret lines in a file", loadCommand),
    cmd("paste", "[-raw] [path]", "enter paste mode or paste a file", pasteCommand),
    nullary("power", "enable power user mode", powerCmd),
    nullary("quit", "exit the interpreter", () => Result(keepRunning = false, None)),
    cmd("replay", "[options]", "reset the repl and replay all previous commands", replayCommand),
    cmd("require", "<path>", "add a jar to the classpath", require),
    cmd("reset", "[options]", "reset the repl to its initial state, forgetting all session entries", resetCommand),
    cmd("save", "<path>", "save replayable session to a file", saveCommand),
    shCommand,
    cmd("settings", "<options>", "update compiler options, if possible; see reset", changeSettings),
    nullary("silent", "disable/enable automatic printing of results", verbosity),
    cmd("type", "[-v] <expr>", "display the type of an expression without evaluating it", typeCommand),
    cmd("kind", "[-v] <expr>", "display the kind of expression's type", kindCommand),
    nullary("warnings", "show the suppressed warnings from the most recent line which had any", warningsCommand)
  )
  
  /** Create a new interpreter. */
  def createInterpreter() {
    if (addedClasspath != "")
      settings.classpath append addedClasspath
    // 官方实现的IMain
    intp = new ILoopInterpreter
  }
  
   // start an interpreter with the given settings
   // org.apache.spark.repl.Main中最后调用的 SparkILoop.process()的具体实现
  def process(settings: Settings): Boolean = savingContextLoader {
    this.settings = settings
    // 创建Interpreter , spark 重写了这部分，加入了初始化spark环境的操作
    createInterpreter()

    // sets in to some kind of reader depending on environmental cues
    in = in0.fold(chooseReader(settings))(r => SimpleReader(r, out, interactive = true))
    globalFuture = future {
      intp.initializeSynchronous()
      loopPostInit()
      !intp.reporter.hasErrors
    }
    // 使用load 命令加载文件，主要是外部jar包
    loadFiles(settings)
    // 打印我们熟悉的欢迎界面
    printWelcome()
    // 进入 read-eval-print循环
    try loop() match {
      case LineResults.EOF => out print Properties.shellInterruptedString
      case _               =>
    }
    catch AbstractOrMissingHandler()
    finally closeInterpreter()

    true
  }
  
   /** The main read-eval-print loop for the repl.  It calls
   *  command() for each line of input, and stops when
   *  command() returns false.
   */
  @tailrec final def loop(): LineResult = {
    import LineResults._
    // 读取1行输入
    readOneLine() match {
      case null => EOF
      // 运行 processLine() , 没错 继续调用loop循环
      case line => if (try processLine(line) catch crashRecovery) loop() else ERR
    }
  }
 }

```
从核心代码我们可以明确的感受到read-eval-print这个循环过程，使用`readOneLine()`从输入中读取指令，`processLine(line)`运行指令并打印输出然后在循环。至于具体实现和细节欢迎读者自己探索并与作者交流。


# 总结
1. spark-shell实际是使用`spark-submit`运行`org.apache.spark.repl.Main`。
2. 该Main主要是就是初始化`SparkILoop`并调用其process()方法。
3. `SparkILoop`主要是继承scala的`ILoop`，并在指定初始化时运行的代码，这些代码主要是创建`sparkContext`。
4. scala ILoop为Interpreter类提供了一个read-eval-print的循环环境。
5. 后续准备将SparkILoop的输入输出流与HTTP对接，实现一个可以与spark直接交互的web界面。




