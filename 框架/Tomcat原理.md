#### Tomcat 体系结构

##### Server 服务器

在 Tomcat 中，Server 代表整个容器。Tomcat 提供了 Server 接口的默认实现，用户很少自定义该接口。

##### Service 服务

服务是一个中间组件，位于服务器内部，并将一个或多个连接器绑定到一个引擎。用户很少自定义 Service 元素，因为默认实现简单且充分：Service 接口。

##### Engine 发动机

Engine 表示特定服务的请求处理管道。由于一个服务可能有多个连接器，引擎接收并处理来自这些连接器的所有请求，将响应交还给相应的连接器以传输到客户端。引擎接口可以实现以提供自定义引擎。

##### Host 主机

主机是网络名称与 Tomcat 服务器的关联。一个 Engine 可以包含多个主机，并且 Host 元素还支持网络别名，如 yourcompany.com 和 abc.yourcompany.com。用户很少创建自定义主机，因为 StaandardHost 实现提供了重要的附加功能

##### Connector 连接器

连接器处理与客户端的通信。Tomcat 提供了多个连接器。其中包括用于大多数 HTTP 流量的 HTTP 连接器，尤其是在将 Tomcat 作为独立服务器运行时，以及实现将 Tomcat 连接到 Web 服务器时使用的 AJP 协议的 AJP 连接器。创建自定义连接器时一项艰巨的工作。

##### Context 上下文

Context 表示 Web 应用程序。主机可以包含多个上下文，每个上下文都有唯一的路径。可以实现 Context 接口来创建自定义 Context，但这种情况很少发生，因为 StandardContext 提供了重要的附加功能。

##### 启动

`org.apache.catalina.startup.Bootstrap#main` 是 Tomcat 启动入口

1. 先调用 init() 方法初始化配置信息，加载启动类 `org.apache.catalina.startup.Catalina`

2. 反射调用 `org.apache.catalina.startup.Catalina#load()` 方法，初始化服务器

   - `org.apache.catalina.core.StandardServer#initInternal` 方法 for 循环初始化多个 service

   ```java
   protected void initInternal() throws LifecycleException {
   
           super.initInternal();
   
           // Initialize utility executor
           reconfigureUtilityExecutor(getUtilityThreadsInternal(utilityThreads));
           register(utilityExecutor, "type=UtilityExecutor");
   
           // Register global String cache
           // Note although the cache is global, if there are multiple Servers
           // present in the JVM (may happen when embedding) then the same cache
           // will be registered under multiple names
           onameStringCache = register(new StringCache(), "type=StringCache");
   
           // Register the MBeanFactory
           MBeanFactory factory = new MBeanFactory();
           factory.setContainer(this);
           onameMBeanFactory = register(factory, "type=MBeanFactory");
   
           // Register the naming resources
           globalNamingResources.init();
   
           // Populate the extension validator with JARs from common and shared
           // class loaders
           if (getCatalina() != null) {
               ClassLoader cl = getCatalina().getParentClassLoader();
               // Walk the class loader hierarchy. Stop at the system class loader.
               // This will add the shared (if present) and common class loaders
               while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
                   if (cl instanceof URLClassLoader) {
                       URL[] urls = ((URLClassLoader) cl).getURLs();
                       for (URL url : urls) {
                           if (url.getProtocol().equals("file")) {
                               try {
                                   File f = new File (url.toURI());
                                   if (f.isFile() &&
                                           f.getName().endsWith(".jar")) {
                                       ExtensionValidator.addSystemResource(f);
                                   }
                               } catch (URISyntaxException e) {
                                   // Ignore
                               } catch (IOException e) {
                                   // Ignore
                               }
                           }
                       }
                   }
                   cl = cl.getParent();
               }
           }
           // 循环初始化 Services
           // Initialize our defined Services
           for (int i = 0; i < services.length; i++) {
               services[i].init();
           }
       }
   ```

   - `org.apache.catalina.core.StandardService#initInternal` 方法初始化 Engine（`org.apache.catalina.core.StandardEngine`）、 Executors 和 Connectors

   ```java
   protected void initInternal() throws LifecycleException {
   
           super.initInternal();
   
           if (engine != null) {
           	// 初始化 engine
               engine.init();
           }
           
   		// 初始化 Executors
           // Initialize any Executors
           for (Executor executor : findExecutors()) {
               if (executor instanceof JmxEnabled) {
                   ((JmxEnabled) executor).setDomain(getDomain());
               }
               executor.init();
           }
   
           // Initialize mapper listener
           mapperListener.init();
   
   		// 循环初始化 Connectors
           // Initialize our defined Connectors
           synchronized (connectorsLock) {
               for (Connector connector : connectors) {
                   try {
                       connector.init();
                   } catch (Exception e) {
                       String message = sm.getString("standardService.connector.initFailed", connector);
                       log.error(message, e);
   
                       if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                           throw new LifecycleException(message);
                       }
                   }
               }
           }
       }
   ```

   - `org.apache.catalina.connector.Connector#initInternal` 方法初始化 adapter 和 protocolHandler

   ```java
   protected void initInternal() throws LifecycleException {
   
           super.initInternal();
   
       	// 初始化 adapter
           // Initialize adapter
           adapter = new CoyoteAdapter(this);
           protocolHandler.setAdapter(adapter);
   
           // Make sure parseBodyMethodsSet has a default
           if (null == parseBodyMethodsSet) {
               setParseBodyMethods(getParseBodyMethods());
           }
   
           if (protocolHandler.isAprRequired() && !AprLifecycleListener.isInstanceCreated()) {
               throw new LifecycleException(
                       sm.getString("coyoteConnector.protocolHandlerNoAprListener", getProtocolHandlerClassName()));
           }
           if (protocolHandler.isAprRequired() && !AprLifecycleListener.isAprAvailable()) {
               throw new LifecycleException(
                       sm.getString("coyoteConnector.protocolHandlerNoAprLibrary", getProtocolHandlerClassName()));
           }
           if (AprLifecycleListener.isAprAvailable() && AprLifecycleListener.getUseOpenSSL() &&
                   protocolHandler instanceof AbstractHttp11JsseProtocol) {
               AbstractHttp11JsseProtocol<?> jsseProtocolHandler = (AbstractHttp11JsseProtocol<?>) protocolHandler;
               if (jsseProtocolHandler.isSSLEnabled() && jsseProtocolHandler.getSslImplementationName() == null) {
                   // OpenSSL is compatible with the JSSE configuration, so use it if APR is available
                   jsseProtocolHandler.setSslImplementationName(OpenSSLImplementation.class.getName());
               }
           }
   
           try {
               // protocolHadler 初始化
               protocolHandler.init();
           } catch (Exception e) {
               throw new LifecycleException(sm.getString("coyoteConnector.protocolHandlerInitializationFailed"), e);
           }
       }
   ```

   - `org.apache.coyote.AbstractProtocol#init` 方法初始化 endpoint

   ```java
   public void init() throws Exception {
           if (getLog().isInfoEnabled()) {
               getLog().info(sm.getString("abstractProtocolHandler.init", getName()));
               logPortOffset();
           }
   
           if (oname == null) {
               // Component not pre-registered so register it
               oname = createObjectName();
               if (oname != null) {
                   Registry.getRegistry(null, null).registerComponent(this, oname, null);
               }
           }
   
           if (this.domain != null) {
               ObjectName rgOname = new ObjectName(domain + ":type=GlobalRequestProcessor,name=" + getName());
               this.rgOname = rgOname;
               Registry.getRegistry(null, null).registerComponent(getHandler().getGlobal(), rgOname, null);
           }
   
           String endpointName = getName();
           endpoint.setName(endpointName.substring(1, endpointName.length() - 1));
           endpoint.setDomain(domain);
   		// 初始化 endpoint
           endpoint.init();
       }
   ```

   - 最终调用 `org.apache.tomcat.util.net.NioEndpoint#initServerSocket` 方法开启 ServerSocketChannel， 用于接收新的 Socket 连接。

3. 启动服务

   - 反射调用 `org.apache.catalina.startup.Catalina#start` 方法，启动服务

   ```java
   public void start() {
   
           if (getServer() == null) {
               load();
           }
   
           if (getServer() == null) {
               log.fatal("Cannot start server. Server instance is not configured.");
               return;
           }
   
           long t1 = System.nanoTime();
   
           // Start the new server
           try {
               getServer().start();
           } catch (LifecycleException e) {
               log.fatal(sm.getString("catalina.serverStartFail"), e);
               try {
                   getServer().destroy();
               } catch (LifecycleException e1) {
                   log.debug("destroy() failed for failed Server ", e1);
               }
               return;
           }
   
           long t2 = System.nanoTime();
           if(log.isInfoEnabled()) {
               log.info("Server startup in " + ((t2 - t1) / 1000000) + " ms");
           }
   
           // Register shutdown hook
           if (useShutdownHook) {
               if (shutdownHook == null) {
                   shutdownHook = new CatalinaShutdownHook();
               }
               Runtime.getRuntime().addShutdownHook(shutdownHook);
   
               // If JULI is being used, disable JULI's shutdown hook since
               // shutdown hooks run in parallel and log messages may be lost
               // if JULI's hook completes before the CatalinaShutdownHook()
               LogManager logManager = LogManager.getLogManager();
               if (logManager instanceof ClassLoaderLogManager) {
                   ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                           false);
               }
           }
   
           if (await) {
               await();
               stop();
           }
       }
   ```

   - `org.apache.catalina.core.StandardServer#startInternal` 方法循环启动 service

   ```java
   protected void startInternal() throws LifecycleException {
   
           fireLifecycleEvent(CONFIGURE_START_EVENT, null);
           setState(LifecycleState.STARTING);
   
           globalNamingResources.start();
   
           // Start our defined Services
           synchronized (servicesLock) {
               // 循环启动 services
               for (Service service : services) {
                   service.start();
               }
           }
       }
   ```

   - `org.apache.catalina.core.StandardService#startInternal` 方法启动 engine 和executor，并创建 Connectors

   ```java
   protected void startInternal() throws LifecycleException {
   
           if (log.isInfoEnabled()) {
               log.info(sm.getString("standardService.start.name", this.name));
           }
           setState(LifecycleState.STARTING);
   
           // Start our defined Container first
           if (engine != null) {
               synchronized (engine) {
                   engine.start();
               }
           }
   
           synchronized (executors) {
               for (Executor executor : executors) {
                   executor.start();
               }
           }
   
           mapperListener.start();
   
           // Start our defined Connectors second
           synchronized (connectorsLock) {
               for (Connector connector : connectors) {
                   try {
                       // If it has already failed, don't try and start it
                       if (connector.getState() != LifecycleState.FAILED) {
                           // 启动 protorhandler 和 endpoint
                           connector.start();
                       }
                   } catch (Exception e) {
                       log.error(sm.getString("standardService.connector.startFailed", connector), e);
                   }
               }
           }
       }
   ```

   - `org.apache.tomcat.util.net.Acceptor#run` 启动 socket acceptor

   ```java
   public void run() {
   
           int errorDelay = 0;
           long pauseStart = 0;
   
           try {
               // Loop until we receive a shutdown command
               while (!stopCalled) {
   
                   // Loop if endpoint is paused.
                   // There are two likely scenarios here.
                   // The first scenario is that Tomcat is shutting down. In this
                   // case - and particularly for the unit tests - we want to exit
                   // this loop as quickly as possible. The second scenario is a
                   // genuine pause of the connector. In this case we want to avoid
                   // excessive CPU usage.
                   // Therefore, we start with a tight loop but if there isn't a
                   // rapid transition to stop then sleeps are introduced.
                   // < 1ms       - tight loop
                   // 1ms to 10ms - 1ms sleep
                   // > 10ms      - 10ms sleep
                   while (endpoint.isPaused() && !stopCalled) {
                       if (state != AcceptorState.PAUSED) {
                           pauseStart = System.nanoTime();
                           // Entered pause state
                           state = AcceptorState.PAUSED;
                       }
                       if ((System.nanoTime() - pauseStart) > 1_000_000) {
                           // Paused for more than 1ms
                           try {
                               if ((System.nanoTime() - pauseStart) > 10_000_000) {
                                   Thread.sleep(10);
                               } else {
                                   Thread.sleep(1);
                               }
                           } catch (InterruptedException e) {
                               // Ignore
                           }
                       }
                   }
   
                   if (stopCalled) {
                       break;
                   }
                   state = AcceptorState.RUNNING;
   
                   try {
                       //if we have reached max connections, wait
                       endpoint.countUpOrAwaitConnection();
   
                       // Endpoint might have been paused while waiting for latch
                       // If that is the case, don't accept new connections
                       if (endpoint.isPaused()) {
                           continue;
                       }
   
                       U socket = null;
                       try {
                           // Accept the next incoming connection from the server
                           // socket
                           socket = endpoint.serverSocketAccept();
                       } catch (Exception ioe) {
                           // We didn't get a socket
                           endpoint.countDownConnection();
                           if (endpoint.isRunning()) {
                               // Introduce delay if necessary
                               errorDelay = handleExceptionWithDelay(errorDelay);
                               // re-throw
                               throw ioe;
                           } else {
                               break;
                           }
                       }
                       // Successful accept, reset the error delay
                       errorDelay = 0;
   
                       // Configure the socket
                       if (!stopCalled && !endpoint.isPaused()) {
                           // setSocketOptions() will hand the socket off to
                           // an appropriate processor if successful
                           if (!endpoint.setSocketOptions(socket)) {
                               endpoint.closeSocket(socket);
                           }
                       } else {
                           endpoint.destroySocket(socket);
                       }
                   } catch (Throwable t) {
                       ExceptionUtils.handleThrowable(t);
                       String msg = sm.getString("endpoint.accept.fail");
                       // APR specific.
                       // Could push this down but not sure it is worth the trouble.
                       if (t instanceof org.apache.tomcat.jni.Error) {
                           org.apache.tomcat.jni.Error e = (org.apache.tomcat.jni.Error) t;
                           if (e.getError() == 233) {
                               // Not an error on HP-UX so log as a warning
                               // so it can be filtered out on that platform
                               // See bug 50273
                               log.warn(msg, t);
                           } else {
                               log.error(msg, t);
                           }
                       } else {
                               log.error(msg, t);
                       }
                   }
               }
           } finally {
               stopLatch.countDown();
           }
           state = AcceptorState.ENDED;
       }
   ```

   - `org.apache.tomcat.util.net.NioEndpoint#serverSocketAccept`

   ```java
   protected SocketChannel serverSocketAccept() throws Exception {
       	// 开启 socketChannel，接收 socket
           SocketChannel result = serverSock.accept();
   
           // Bug does not affect Windows. Skip the check on that platform.
           if (!JrePlatform.IS_WINDOWS) {
               SocketAddress currentRemoteAddress = result.getRemoteAddress();
               long currentNanoTime = System.nanoTime();
               if (currentRemoteAddress.equals(previousAcceptedSocketRemoteAddress) &&
                       currentNanoTime - previousAcceptedSocketNanoTime < 1000) {
                   throw new IOException(sm.getString("endpoint.err.duplicateAccept"));
               }
               previousAcceptedSocketRemoteAddress = currentRemoteAddress;
               previousAcceptedSocketNanoTime = currentNanoTime;
           }
   
           return result;
       }
   ```

   