---
title: Spark Launcher(v 2.3.2)
tags:
  - 源码阅读
categories：
  - spark
---
# 概述
Spark Launcher的主要有两个功能：1、通过JAVA程序启动Spark任务。2、监控Spark任务的运行情况，可以根据任务状态等的变法设置不同的执行逻辑同时提供主动停止Application的功能。  
通常发布spark任务是通过shell运行spark-submit命令。但是在实际生产活动中存在业务直接启动Spark任务的情况。对于这种情况目前的做法是Java使用Runtime.getRuntime().exec(cmd)的方式运行spark-submit。这种方式虽然可以正常启动spark任务，收集spark-submit产生的日志，但是容错率低，不容易使用程序监控spark任务的状态，主动停止任务。对于这种情况可以使用Spark官方封装的任务启动器——Spark Launcher。虽然其底层也是通过JAVA启动子进程发布Spark任务，但是程序启动了监听spark application的线程，可以在任务状态或信息发生变化时触发相应的listenter。  
本文首先用一个例子展示了如何用JAVA启动一个Spark任务，监控其运行状态以及如何主动停止任务。然后着重分析了启动任务的SparkLauncher、监控任务的ChildProcAppHandle、接收任务信息的LauncherServer、发送任务信息的LauncherBackend的代码逻辑。
# 代码例子
```java
public class TestLauncher {
    public static void main(String[] args) throws IOException {
        SparkAppHandle handler = new SparkLauncher()
                .setAppName("sparkLaunch")
                .setJavaHome("/usr/java/default/")
                .setSparkHome("/opt/spark")
                .setMaster("yarn")
                .setDeployMode("client")
                .setConf("spark.driver.memory", "1g")
                .setConf("spark.executor.memory", "1g")
                .setConf("spark.executor.instances", "3")
                .setConf(SparkLauncher.EXECUTOR_CORES, "1")
                .setConf(SparkLauncher.CHILD_PROCESS_LOGGER_NAME, "testLauncher")
                .setAppResource("/home/Phoenix/bran.jar")
                .setMainClass("Test")
                .addAppArgs(new String[]{"test"})
                .setVerbose(true)
                .startApplication(new SparkAppHandle.Listener() {
                    @Override
                    public void stateChanged(SparkAppHandle handle) {
                        System.out.println("**********  state  changed  **********");
                        System.out.println(handle.getState());
 
                        if ("Failed".equalsIgnoreCase(handle.getState().toString())) {
                            System.out.println("application error！");
                        }
                    }
                    @Override
                    public void infoChanged(SparkAppHandle handle) {
                        System.out.println("**********  info  changed  **********");
                        System.out.println(handle.getState());
                    }
                });
        System.out.println("id    " + handler.getAppId());
        System.out.println("state " + handler.getState());
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        handler.stop();
 
    }
 
}
```
# Spark Launcher源码分析
## 整体过程
SparkLauncher用startApplication方法开始一个spark任务，该方法主要干了以下2件事：1、通过createBuilder方法构造Process对象调用spark-submit脚本；2、同时启动LauncherServer服务，用于接收LauncherBackend的消息。3、将Process和LauncherServer绑定到一个ChildProcAppHandle并将其返回给用户。spark-submit负责将任务提交到spark集群，再spark-submint中会启动LauncherBackend来连接LauncherServer，然后通过setAppId，setState发送通知到 launcherServer，同时接受其发送的Stop。LauncherServer收到通知后会调用用户提供的listener做相应的处理。主要逻辑如下图。
![逻辑图](http://rfc2616.oss-cn-beijing.aliyuncs.com/blog/172611.png)
## SparkLauncher
SparkLauncher是用户直接接触的用于启动Spark任务的对象。主要通过startApplication开始Spark任务。返回的用户使用的spark任务监控对象ChildProcAppHandle，该对象绑定了启动任务的Process以及用于接受集群信息的LauncherServer。主要代码如下
```scala
/**
 * Launcher for Spark applications.
 * <p>
 * Use this class to start Spark applications programmatically. The class uses a builder pattern
 * to allow clients to configure the Spark application and launch it as a child process.
 * </p>
 */
public class SparkLauncher extends AbstractLauncher<SparkLauncher> {
    // spark任务配置相关的相关代码
    …………
     
    /**
   * 同样以子进程的方式启动spark application.
   * 但是startApplication(SparkAppHandle.Listener...)可以更好的控制子应用。
   */
  public Process launch() throws IOException {
   ………………
  }
   
   /**
   * Starts a Spark application.
   *
   * <p>
   * Applications launched by this launcher run as child processes. The child's stdout and stderr
   * are merged and written to a logger (see <code>java.util.logging</code>) only if redirection
   * has not otherwise been configured on this <code>SparkLauncher</code>. The logger's name can be
   * defined by setting {@link #CHILD_PROCESS_LOGGER_NAME} in the app's configuration. If that
   * option is not set, the code will try to derive a name from the application's name or main
   * class / script file. If those cannot be determined, an internal, unique name will be used.
   * In all cases, the logger name will start with "org.apache.spark.launcher.app", to fit more
   * easily into the configuration of commonly-used logging systems.
   *
   * @since 1.6.0
   * @see AbstractLauncher#startApplication(SparkAppHandle.Listener...)
   * @param listeners Listeners to add to the handle before the app is launched.
   * @return A handle for the launched application.
   */
  @Override
  public SparkAppHandle startApplication(SparkAppHandle.Listener... listeners) throws IOException {
    // 获取launcherServer
    LauncherServer server = LauncherServer.getOrCreateServer();
    // 创建用于监视spark应用的对象，该方法返回的对象。
    ChildProcAppHandle handle = new ChildProcAppHandle(server);
    // 绑定各种处理逻辑
    for (SparkAppHandle.Listener l : listeners) {
      handle.addListener(l);
    }
    // 在launchServer上注册本任务的监控，返回secret
    String secret = server.registerHandle(handle);
    //根据配置的CHILD_PROCESS_LOGGER_NAME 获取loggerName
    String loggerName = getLoggerName();
    //相当于编写启动spark任务的shell脚本，该脚本主要是运行spark-submit命令。
    ProcessBuilder pb = createBuilder();
 
    boolean outputToLog = outputStream == null;
    boolean errorToLog = !redirectErrorStream && errorStream == null;
 
    // Only setup stderr + stdout to logger redirection if user has not otherwise configured output
    // redirection.
    // 用户没有设置日志重定向的话，将stderr stdout 重定向到logger上
    if (loggerName == null && (outputToLog || errorToLog)) {
      String appName;
      if (builder.appName != null) {
        appName = builder.appName;
      } else if (builder.mainClass != null) {
        int dot = builder.mainClass.lastIndexOf(".");
        if (dot >= 0 && dot < builder.mainClass.length() - 1) {
          appName = builder.mainClass.substring(dot + 1, builder.mainClass.length());
        } else {
          appName = builder.mainClass;
        }
      } else if (builder.appResource != null) {
        appName = new File(builder.appResource).getName();
      } else {
        appName = String.valueOf(COUNTER.incrementAndGet());
      }
      String loggerPrefix = getClass().getPackage().getName();
      loggerName = String.format("%s.app.%s", loggerPrefix, appName);
    }
 
    if (outputToLog && errorToLog) {
      pb.redirectErrorStream(true);
    }
    // 环境变量增加LauncherServer的端口，和secret
    pb.environment().put(LauncherProtocol.ENV_LAUNCHER_PORT, String.valueOf(server.getPort()));
    pb.environment().put(LauncherProtocol.ENV_LAUNCHER_SECRET, secret);
    try {
    //shell exec spark-submit
      Process child = pb.start();
      InputStream logStream = null;
      if (loggerName != null) {
        logStream = outputToLog ? child.getInputStream() : child.getErrorStream();
      }
      // 运行的子进程和监控handler绑定
      handle.setChildProc(child, loggerName, logStream);
    } catch (IOException ioe) {
      handle.kill();
      throw ioe;
    }
 
    return handle;
  }
  //
private ProcessBuilder createBuilder() throws IOException {
    List<String> cmd = new ArrayList<>();
    //根据系统区别，选用 spark-submit or spark-submit.cmd
    cmd.add(findSparkSubmit());
    // 根据配置，构造 spark-submit 后跟的参数，如 appName,executer num等
    cmd.addAll(builder.buildSparkSubmitArgs());
 
    // Since the child process is a batch script, let's quote things so that special characters are
    // preserved, otherwise the batch interpreter will mess up the arguments. Batch scripts are
    // weird.
    if (isWindows()) {
      List<String> winCmd = new ArrayList<>();
      for (String arg : cmd) {
        winCmd.add(quoteForBatchScript(arg));
      }
      cmd = winCmd;
    }
    //ProcessBuilder 绑定构造的命令
    ProcessBuilder pb = new ProcessBuilder(cmd.toArray(new String[cmd.size()]));
    // ProcessBuilder 绑定环境变量
    for (Map.Entry<String, String> e : builder.childEnv.entrySet()) {
      pb.environment().put(e.getKey(), e.getValue());
    }
    // 设置程序运行的目录
    if (workingDir != null) {
      pb.directory(workingDir);
    }
 
    // Only one of redirectError and redirectError(...) can be specified.
    // Similarly, if redirectToLog is specified, no other redirections should be specified.
    checkState(!redirectErrorStream || errorStream == null,
      "Cannot specify both redirectError() and redirectError(...) ");
    checkState(getLoggerName() == null ||
      ((!redirectErrorStream && errorStream == null) || outputStream == null),
      "Cannot used redirectToLog() in conjunction with other redirection methods.");
    // 根据配置将spark-submit的输出、错误输出重定向
    if (redirectErrorStream) {
      pb.redirectErrorStream(true);
    }
    if (errorStream != null) {
      pb.redirectError(errorStream);
    }
    if (outputStream != null) {
      pb.redirectOutput(outputStream);
    }
 
    return pb;
  }
 
}
```

## ChildProcAppHandle
ChildProcAppHandle是sparkLauncher返回给用户，用于控制Spark application的对象。重点关注并区别其disconnect()方法（断开与sparkApplication的通信）,kill()（调用disconnect方法的同时杀死spark-submit子进程）,stop()（通知spark停止任务）；注意在调用setState()和setAppId()时（spark任务状态或AppId改变时）会遍历执行用户传入listeners。源码如下
```scala
/**
 * Handle implementation for monitoring apps started as a child process.
 */
class ChildProcAppHandle extends AbstractAppHandle {
  private static final Logger LOG = Logger.getLogger(ChildProcAppHandle.class.getName());
  // spark application 子进程
  private volatile Process childProc;
  // 打印日志
  private OutputRedirector redirector;
  ChildProcAppHandle(LauncherServer server) {
    super(server);
  }
 // 断开连接，即不关注spark任务的状态改变但spark任务还会继续执行，不再关注spark-submit子进程但进程还会继续执行
  @Override
  public synchronized void disconnect() {
    try {
      super.disconnect();
    } finally {
      if (redirector != null) {
        redirector.stop();
      }
    }
  }
  // 调用disconnect方法，并调用子进程的destroyForcibly结束spark-submit子程序，但是根据spark任务提交方式的不同（yarn-client/yarn-cluster）spark任务可以停止也可能继续执行。
  @Override
  public synchronized void kill() {
    if (!isDisposed()) {
      setState(State.KILLED);
      disconnect();
      if (childProc != null) {
        if (childProc.isAlive()) {
          childProc.destroyForcibly();
        }
        childProc = null;
      }
    }
  }
// 绑定子进程，sparkLuncher中调用
  void setChildProc(Process childProc, String loggerName, InputStream logStream) {
    this.childProc = childProc;
    if (logStream != null) {
      this.redirector = new OutputRedirector(logStream, loggerName,
        SparkLauncher.REDIRECTOR_FACTORY, this);
    } else {
      // If there is no log redirection, spawn a thread that will wait for the child process
      // to finish.
      SparkLauncher.REDIRECTOR_FACTORY.newThread(this::monitorChild).start();
    }
  }
 
  /**
   * Wait for the child process to exit and update the handle's state if necessary, according to
   * the exit code.
   * 等待子进程退出，并根据需要更改SparkHandler的状态。
   */
  void monitorChild() {
    Process proc = childProc;
    if (proc == null) {
      // Process may have already been disposed of, e.g. by calling kill().
      return;
    }
    // 等待子进程退出
    while (proc.isAlive()) {
      try {
        proc.waitFor();
      } catch (Exception e) {
        LOG.log(Level.WARNING, "Exception waiting for child process to exit.", e);
      }
    }
    //更改sparkHandler的状态
    synchronized (this) {
      if (isDisposed()) {
        return;
      }
 
      int ec;
      try {
        ec = proc.exitValue();
      } catch (Exception e) {
        LOG.log(Level.WARNING, "Exception getting child process exit code, assuming failure.", e);
        ec = 1;
      }
 
      if (ec != 0) {
        State currState = getState();
        // Override state with failure if the current state is not final, or is success.
        if (!currState.isFinal() || currState == State.FINISHED) {
          setState(State.FAILED, true);
        }
      }
 
      dispose();
    }
  }
 
}
// ChildProcAppHandle父类
abstract class AbstractAppHandle implements SparkAppHandle {
 
  private static final Logger LOG = Logger.getLogger(AbstractAppHandle.class.getName());
 
  private final LauncherServer server;
 
  private LauncherServer.ServerConnection connection;
  private List<Listener> listeners;
  private AtomicReference<State> state;
  private volatile String appId;
  private volatile boolean disposed;
 
  protected AbstractAppHandle(LauncherServer server) {
    this.server = server;
    this.state = new AtomicReference<>(State.UNKNOWN);
  }
//添加listener，当spark状态发生改变时触发
  @Override
  public synchronized void addListener(Listener l) {
    if (listeners == null) {
      listeners = new CopyOnWriteArrayList<>();
    }
    listeners.add(l);
  }
 
  @Override
  public State getState() {
    return state.get();
  }
 
  @Override
  public String getAppId() {
    return appId;
  }
 
  @Override
  public void stop() {
    CommandBuilderUtils.checkState(connection != null, "Application is still not connected.");
    try {
      connection.send(new LauncherProtocol.Stop());
    } catch (IOException ioe) {
      throw new RuntimeException(ioe);
    }
  }
 
  @Override
  public synchronized void disconnect() {
    if (connection != null && connection.isOpen()) {
      try {
        connection.close();
      } catch (IOException ioe) {
        // no-op.
      }
    }
    dispose();
  }
 
  void setConnection(LauncherServer.ServerConnection connection) {
    this.connection = connection;
  }
 
  LauncherConnection getConnection() {
    return connection;
  }
 
  boolean isDisposed() {
    return disposed;
  }
 
  /**
   * Mark the handle as disposed, and set it as LOST in case the current state is not final.
   *
   * This method should be called only when there's a reasonable expectation that the communication
   * with the child application is not needed anymore, either because the code managing the handle
   * has said so, or because the child application is finished.
   * 此方法只在不需要与spark-submit通信时调用，并将sparkhandler的状态在不是final时设为lost
   */
  synchronized void dispose() {
    if (!isDisposed()) {
      // First wait for all data from the connection to be read. Then unregister the handle.
      // Otherwise, unregistering might cause the server to be stopped and all child connections
      // to be closed.
      if (connection != null) {
        try {
          connection.waitForClose();
        } catch (IOException ioe) {
          // no-op.
        }
      }
      server.unregister(this);
 
      // Set state to LOST if not yet final.
      setState(State.LOST, false);
      this.disposed = true;
    }
  }
// 设置sparkhandler的状态
  void setState(State s) {
    setState(s, false);
  }
 
  void setState(State s, boolean force) {
    if (force) {
      state.set(s);
      fireEvent(false);
      return;
    }
 
    State current = state.get();
    while (!current.isFinal()) {
      if (state.compareAndSet(current, s)) {
        fireEvent(false);
        return;
      }
      current = state.get();
    }
 
    if (s != State.LOST) {
      LOG.log(Level.WARNING, "Backend requested transition from final state {0} to {1}.",
        new Object[] { current, s });
    }
  }
//设置AppId
  void setAppId(String appId) {
    this.appId = appId;
    fireEvent(true);
  }
// 调用用户传入的Listenter，可以开出在sparkHander state改变以及AppId改变时触发此方法
  private void fireEvent(boolean isInfoChanged) {
    if (listeners != null) {
      for (Listener l : listeners) {
        if (isInfoChanged) {
          l.infoChanged(this);
        } else {
          l.stateChanged(this);
        }
      }
    }
  }
 
}
```

## LauncherServer
LauncherServer是一个用来接收LauncherBackend发送spark app状态变化的服务。  
每个spark app都有一个用于标记其的secret，在启动spark app时会被发送，LauncherBackend回连LauncherServer时需要带上这个secret。LauncherServer等待回连是有时间限制的。   
LauncherServer是一个单例，启动了一个核心线程（acceptConnections）循环处理回连。线程为每个回连创建ServerConnection线程，专门处理一个spark app发送过来的Hello、SetAppId、SetState消息并在合适时候向spark app发送Stop Msg。
```scala
class LauncherServer implements Closeable {
 
  private static final Logger LOG = Logger.getLogger(LauncherServer.class.getName());
  private static final String THREAD_NAME_FMT = "LauncherServer-%d";
  private static final long DEFAULT_CONNECT_TIMEOUT = 10000L; //回连超时时间
 
  /** 用于创建secret，它用于连接子进程 */
  private static final SecureRandom RND = new SecureRandom();
  // 单例
  private static volatile LauncherServer serverInstance;
  // 获取LauncherServer
  static synchronized LauncherServer getOrCreateServer() throws IOException {
    LauncherServer server;
    do {
      server = serverInstance != null ? serverInstance : new LauncherServer();
    } while (!server.running);
 
    server.ref();
    serverInstance = server;
    return server;
  }
 
  // For testing.
  static synchronized LauncherServer getServer() {
    return serverInstance;
  }
 
  private final AtomicLong refCount;
  private final AtomicLong threadIds;
  // 等待回连的 secret 与 ChildProcAppHandle 的映射
  private final ConcurrentMap<String, AbstractAppHandle> secretToPendingApps;
  private final List<ServerConnection> clients;
  private final ServerSocket server;
  private final Thread serverThread;
  private final ThreadFactory factory;
  private final Timer timeoutTimer;
 
  private volatile boolean running;
// 创建LauncerServer
  private LauncherServer() throws IOException {
    this.refCount = new AtomicLong(0);
    //创建socket用于接受信息
    ServerSocket server = new ServerSocket();
    try {
      server.setReuseAddress(true);
      server.bind(new InetSocketAddress(InetAddress.getLoopbackAddress(), 0));
 
      this.clients = new ArrayList<>();
      this.threadIds = new AtomicLong();
      this.factory = new NamedThreadFactory(THREAD_NAME_FMT);
      this.secretToPendingApps = new ConcurrentHashMap<>();
      this.timeoutTimer = new Timer("LauncherServer-TimeoutTimer", true);
      this.server = server;
      this.running = true;
      //初始化LauncerServer核心线程，线程run的是acceptConnections方法
      this.serverThread = factory.newThread(this::acceptConnections);
      serverThread.start();
    } catch (IOException ioe) {
      close();
      throw ioe;
    } catch (Exception e) {
      close();
      throw new IOException(e);
    }
  }
 
  /**
   * 在server上注册一个ChildProcAppHandle，并返回一个secret用于唯一标识spark任务
   */
  synchronized String registerHandle(AbstractAppHandle handle) {
    String secret = createSecret();
    secretToPendingApps.put(secret, handle);
    return secret;
  }
//关闭LauncherServer
  @Override
  public void close() throws IOException {
    synchronized (this) {
      if (!running) {
        return;
      }
      running = false;
    }
 
    synchronized(LauncherServer.class) {
      serverInstance = null;
    }
 
    timeoutTimer.cancel();
    server.close();
    synchronized (clients) {
      List<ServerConnection> copy = new ArrayList<>(clients);
      clients.clear();
      for (ServerConnection client : copy) {
        client.close();
      }
    }
 
    if (serverThread != null) {
      try {
        serverThread.join();
      } catch (InterruptedException ie) {
        // no-op
      }
    }
  }
//实例数累加1
  void ref() {
    refCount.incrementAndGet();
  }
//实例数-1
  void unref() {
    synchronized(LauncherServer.class) {
      if (refCount.decrementAndGet() == 0) {
        try {
          close();
        } catch (IOException ioe) {
          // no-op.
        }
      }
    }
  }
 
  int getPort() {
    return server.getLocalPort();
  }
 
  /**
   * 移除ChildProcAppHandle
   */
  void unregister(AbstractAppHandle handle) {
    for (Map.Entry<String, AbstractAppHandle> e : secretToPendingApps.entrySet()) {
      if (e.getValue().equals(handle)) {
        String secret = e.getKey();
        secretToPendingApps.remove(secret);
        break;
      }
    }
 
    unref();
  }
// LauncherServer核心逻辑
  private void acceptConnections() {
    try {
    // 保持循环
      while (running) {
      // 阻塞，等待接受连接
        final Socket client = server.accept();
        //超时
        TimerTask timeout = new TimerTask() {
          @Override
          public void run() {
            LOG.warning("Timed out waiting for hello message from client.");
            try {
              client.close();
            } catch (IOException ioe) {
              // no-op.
            }
          }
        };
        //建立消息处理线程对象，处理sparkBackend返回的信息。
        ServerConnection clientConnection = new ServerConnection(client, timeout);
        // 使用factory创建的线程 名字为LauncherServer-%d
        Thread clientThread = factory.newThread(clientConnection);
        // 线程在存回到clientConnection
        clientConnection.setConnectionThread(clientThread);
         
        synchronized (clients) {
          clients.add(clientConnection);
        }
 
        long timeoutMs = getConnectionTimeout();
        // 设置超时监控，如果超过配置时间，client自动关闭，不再监控此任务
        if (timeoutMs > 0) {
          timeoutTimer.schedule(timeout, timeoutMs);
        } else {
          timeout.run();
        }
        // 启动针对某个spark任务的消息处理线程
        clientThread.start();
      }
    } catch (IOException ioe) {
      if (running) {
        LOG.log(Level.SEVERE, "Error in accept loop.", ioe);
      }
    }
  }
  // 从配置中或默认值中获取回连超时时间，默认超时时间10000L
  private long getConnectionTimeout() {
    String value = SparkLauncher.launcherConfig.get(SparkLauncher.CHILD_CONNECTION_TIMEOUT);
    return (value != null) ? Long.parseLong(value) : DEFAULT_CONNECT_TIMEOUT;
  }
// secret生成算法
  private String createSecret() {
    while (true) {
      byte[] secret = new byte[128];
      RND.nextBytes(secret);
 
      StringBuilder sb = new StringBuilder();
      for (byte b : secret) {
        int ival = b >= 0 ? b : Byte.MAX_VALUE - b;
        if (ival < 0x10) {
          sb.append("0");
        }
        sb.append(Integer.toHexString(ival));
      }
 
      String secretStr = sb.toString();
      if (!secretToPendingApps.containsKey(secretStr)) {
        return secretStr;
      }
    }
  }
// 消息处理线程
// 父类LauncherConnection实现了Runnable接口，主要就有send() 和run()方法
// run()核心逻辑从socketInputStream中读取Message对象交给ServerConnection的handle方法
// send()将Message放到socket的outPutStream中
  class ServerConnection extends LauncherConnection {
 
    private TimerTask timeout;
    private volatile Thread connectionThread;
    private volatile AbstractAppHandle handle;
 
    ServerConnection(Socket socket, TimerTask timeout) throws IOException {
      super(socket);
      this.timeout = timeout;
    }
 
    void setConnectionThread(Thread t) {
      this.connectionThread = t;
    }
 
    @Override
    protected void handle(Message msg) throws IOException {
      try {
      // 如果Message是回连消息
        if (msg instanceof Hello) {
        //清楚超时计时器
          timeout.cancel();
          timeout = null;
          Hello hello = (Hello) msg;
          AbstractAppHandle handle = secretToPendingApps.remove(hello.secret);
          if (handle != null) {
            handle.setConnection(this);
            // 设置状态为CONNECTED
            handle.setState(SparkAppHandle.State.CONNECTED);
            this.handle = handle;
          } else {
            throw new IllegalArgumentException("Received Hello for unknown client.");
          }
        } else {
        //handler == null 即在Hello Message前有其他message返回，报错
          if (handle == null) {
            throw new IllegalArgumentException("Expected hello, got: " +
              msg != null ? msg.getClass().getName() : null);
          }
          // 如果是SetAppId的msg则调用ChildProcAppHandle的setAppId，会触发用户设置的listener的infoChanged方法
          if (msg instanceof SetAppId) {
            SetAppId set = (SetAppId) msg;
            handle.setAppId(set.appId);
          // 如果是SetState的msg则调用ChildProcAppHandle的setAppId，会触发用户设置的listener的stateChanged方法
          } else if (msg instanceof SetState) {
            handle.setState(((SetState)msg).state);
          } else {
            throw new IllegalArgumentException("Invalid message: " +
              msg != null ? msg.getClass().getName() : null);
          }
        }
      } catch (Exception e) {
        LOG.log(Level.INFO, "Error handling message from client.", e);
        if (timeout != null) {
          timeout.cancel();
        }
        close();
        if (handle != null) {
          handle.dispose();
        }
      } finally {
        timeoutTimer.purge();
      }
    }
 
    @Override
    public void close() throws IOException {
      if (!isOpen()) {
        return;
      }
 
      synchronized (clients) {
        clients.remove(this);
      }
 
      super.close();
    }
 
    /**
     * 仅仅在ChildProcAppHandle.dispose()被调用
     * Wait for the remote side to close the connection so that any pending data is processed.
     * This ensures any changes reported by the child application take effect.
     *
     * This method allows a short period for the above to happen (same amount of time as the
     * connection timeout, which is configurable). This should be fine for well-behaved
     * applications, where they close the connection arond the same time the app handle detects the
     * app has finished.
     *
     * In case the connection is not closed within the grace period, this method forcefully closes
     * it and any subsequent data that may arrive will be ignored.
     */
    public void waitForClose() throws IOException {
      Thread connThread = this.connectionThread;
      //当前运行的线程不是消息处理线程ServerConnection
      if (Thread.currentThread() != connThread) {
        try {
        //等待ServerConnection ConnectionTimeout毫秒主线程才能结束
          connThread.join(getConnectionTimeout());
        } catch (InterruptedException ie) {
          // Ignore.
        }
 
        if (connThread.isAlive()) {
        //做出提示。
          LOG.log(Level.WARNING, "Timed out waiting for child connection to close.");
          close();
        }
      }
    }
 
  }
 
}
```

## LauncherBackend
LauncherBackend是跟LauncherServer通信的客户端，向LauncherServer发送状态变化的通信端点。该段代码跑在Spark-submit命令运行的程序。
```scala
/**
 * A class that can be used to talk to a launcher server. Users should extend this class to
 * provide implementation for the abstract methods.
 *
 * See `LauncherServer` for an explanation of how launcher communication works.
 */
private[spark] abstract class LauncherBackend {
 
  private var clientThread: Thread = _
  private var connection: BackendConnection = _
  private var lastState: SparkAppHandle.State = _
  @volatile private var _isConnected = false
 
  protected def conf: SparkConf
//这里进行连接LauncherServer的socket初始化动作，端口是从env中获取的，env里的端口是在SparkLauncher中通告出去的。详情参看SparkLauncher记录
  def connect(): Unit = {
    val port = conf.getOption(LauncherProtocol.CONF_LAUNCHER_PORT)
      .orElse(sys.env.get(LauncherProtocol.ENV_LAUNCHER_PORT))
      .map(_.toInt)
    val secret = conf.getOption(LauncherProtocol.CONF_LAUNCHER_SECRET)
      .orElse(sys.env.get(LauncherProtocol.ENV_LAUNCHER_SECRET))
    if (port != None && secret != None) {
    /*
        *这里建立跟LauncherServer通信的socket，ip是本地回环地址，
        *因为只有通过SparkLauncher的startApplication的方式去提交spark任务的时候LauncherServer才会在本地回环地址上建立监听
        *因为SparkLauncher 通过ProcessBuilder的方式调用spark-submit，所以在spark-submit中会继承父进程的环境变量
        *LauncherBackend才能通过环境变量确定是否存在LauncherServer服务
        */
      val s = new Socket(InetAddress.getLoopbackAddress(), port.get)
      connection = new BackendConnection(s)
      connection.send(new Hello(secret.get, SPARK_VERSION))//发送Hello msg
      clientThread = LauncherBackend.threadFactory.newThread(connection)
      clientThread.start()
      _isConnected = true
    }
  }
 
  def close(): Unit = {
    if (connection != null) {
      try {
        connection.close()
      } finally {
        if (clientThread != null) {
          clientThread.join()
        }
      }
    }
  }
//这里发送setAppId msg
  def setAppId(appId: String): Unit = {
    if (connection != null && isConnected) {
      connection.send(new SetAppId(appId))
    }
  }
// 发送setState msg
  def setState(state: SparkAppHandle.State): Unit = {
    if (connection != null && isConnected && lastState != state) {
      connection.send(new SetState(state))
      lastState = state
    }
  }
 
  /** Return whether the launcher handle is still connected to this backend. */
  def isConnected(): Boolean = _isConnected
 
  /**
   * Implementations should provide this method, which should try to stop the application
   * as gracefully as possible.
   */
  protected def onStopRequest(): Unit
 
  /**
   * Callback for when the launcher handle disconnects from this backend.
   */
  protected def onDisconnected() : Unit = { }
// 启动线程停止任务，根据情况停止任务的逻辑由子类完成。
  private def fireStopRequest(): Unit = {
    val thread = LauncherBackend.threadFactory.newThread(new Runnable() {
      override def run(): Unit = Utils.tryLogNonFatalError {
        onStopRequest()
      }
    })
    thread.start()
  }
// 从LuncherServer仅接受Stop msg
  private class BackendConnection(s: Socket) extends LauncherConnection(s) {
 
    override protected def handle(m: Message): Unit = m match {
      case _: Stop =>
        fireStopRequest()
 
      case _ =>
        throw new IllegalArgumentException(s"Unexpected message type: ${m.getClass().getName()}")
    }
 
    override def close(): Unit = {
      try {
        _isConnected = false
        super.close()
      } finally {
        onDisconnected()
      }
    }
 
  }
 
}
 
private object LauncherBackend {
 
  val threadFactory = ThreadUtils.namedThreadFactory("LauncherBackend")
 
}
```