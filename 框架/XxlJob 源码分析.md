#### 执行器注册

执行器注册入口方法为`com.xxl.job.core.executor.impl.XxlJobSpringExecutor`

XxlJobSpringExecutor 继承了 SmartInitializingSingleton 的 afterSingletonsInstantiated() 方法，Spring 启动之后会调用 `com.xxl.job.core.executor.impl.XxlJobSpringExecutor#afterSingletonsInstantiated` 方法，这就是 xxljob 注册的入口

```java
public void afterSingletonsInstantiated() {

    // init JobHandler Repository
    /*initJobHandlerRepository(applicationContext);*/

    // init JobHandler Repository (for method)
    // 初始化 jobHandler， 扫描加了 @XxlJob 注解的方法，放入 map 缓存中
    initJobHandlerMethodRepository(applicationContext);

    // refresh GlueFactory
    GlueFactory.refreshInstance(1);

    // super start
    try {
        // 启动注册线程 com.xxl.job.core.executor.XxlJobExecutor#start
        super.start();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

接下来调用 XxlJobExecutor 的 start() 方法，开始注册执行器到调度中心

```java
public void start() throws Exception {

    // init logpath
    // 初始化日志存放路径
    XxlJobFileAppender.initLogPath(logPath);

    // init invoker, admin-client
    // 初始化 adminClient
    initAdminBizList(adminAddresses, accessToken);

    // init JobLogFileCleanThread
    // 日志清理线程
    JobLogFileCleanThread.getInstance().start(logRetentionDays);

    // init TriggerCallbackThread
    // 初始化执行结果回调通知调度中心线程
    TriggerCallbackThread.getInstance().start();

    // init executor-server
    // 使用 netty 创建服务，监听调度中心调度，触发任务，同时注册执行器到调度中心
    initEmbedServer(address, ip, port, appname, accessToken);
}
```

调用 `com.xxl.job.core.server.EmbedServer#start`方法创建服务接收调度中心调度，并注册执行器到调度中心

```java
public void start(final String address, final int port, final String appname, final String accessToken) {
    executorBiz = new ExecutorBizImpl();
    thread = new Thread(new Runnable() {
        @Override
        public void run() {
            // param
            EventLoopGroup bossGroup = new NioEventLoopGroup();
            EventLoopGroup workerGroup = new NioEventLoopGroup();
            ThreadPoolExecutor bizThreadPool = new ThreadPoolExecutor(
                    0,
                    200,
                    60L,
                    TimeUnit.SECONDS,
                    new LinkedBlockingQueue<Runnable>(2000),
                    new ThreadFactory() {
                        @Override
                        public Thread newThread(Runnable r) {
                            return new Thread(r, "xxl-job, EmbedServer bizThreadPool-" + r.hashCode());
                        }
                    },
                    new RejectedExecutionHandler() {
                        @Override
                        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                            throw new RuntimeException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!");
                        }
                    });
            try {
                // start server
                ServerBootstrap bootstrap = new ServerBootstrap();
                bootstrap.group(bossGroup, workerGroup)
                        .channel(NioServerSocketChannel.class)
                        .childHandler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            public void initChannel(SocketChannel channel) throws Exception {
                                channel.pipeline()
                                        .addLast(new IdleStateHandler(0, 0, 30 * 3, TimeUnit.SECONDS))  // beat 3N, close if idle
                                        .addLast(new HttpServerCodec())
                                        .addLast(new HttpObjectAggregator(5 * 1024 * 1024))  // merge request & reponse to FULL
                                        .addLast(new EmbedHttpServerHandler(executorBiz, accessToken, bizThreadPool));
                            }
                        })
                        .childOption(ChannelOption.SO_KEEPALIVE, true);

                // bind
                ChannelFuture future = bootstrap.bind(port).sync();

                logger.info(">>>>>>>>>>> xxl-job remoting server start success, nettype = {}, port = {}", EmbedServer.class, port);

                // start registry
                // 发起注册，将执行器信息注册到调度中心
                startRegistry(appname, address);

                // wait util stop
                future.channel().closeFuture().sync();

            } catch (InterruptedException e) {
                logger.info(">>>>>>>>>>> xxl-job remoting server stop.");
            } catch (Exception e) {
                logger.error(">>>>>>>>>>> xxl-job remoting server error.", e);
            } finally {
                // stop
                try {
                    workerGroup.shutdownGracefully();
                    bossGroup.shutdownGracefully();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
    thread.setDaemon(true);    // daemon, service jvm, user thread leave >>> daemon leave >>> jvm leave
    thread.start();
}
```

通过 `com.xxl.job.core.thread.ExecutorRegistryThread#start` 方法注册执行器

```java
public void start(final String appname, final String address){

    // valid
    if (appname==null || appname.trim().length()==0) {
        logger.warn(">>>>>>>>>>> xxl-job, executor registry config fail, appname is null.");
        return;
    }
    if (XxlJobExecutor.getAdminBizList() == null) {
        logger.warn(">>>>>>>>>>> xxl-job, executor registry config fail, adminAddresses is null.");
        return;
    }

    registryThread = new Thread(new Runnable() {
        @Override
        public void run() {

            // registry
            while (!toStop) {
                try {
                    RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
                    for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
                        try {
                            // 注册方法
                            ReturnT<String> registryResult = adminBiz.registry(registryParam);
                            if (registryResult!=null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
                                registryResult = ReturnT.SUCCESS;
                                logger.debug(">>>>>>>>>>> xxl-job registry success, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                                break;
                            } else {
                                logger.info(">>>>>>>>>>> xxl-job registry fail, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                            }
                        } catch (Exception e) {
                            logger.info(">>>>>>>>>>> xxl-job registry error, registryParam:{}", registryParam, e);
                        }

                    }
                } catch (Exception e) {
                    if (!toStop) {
                        logger.error(e.getMessage(), e);
                    }

                }

                try {
                    if (!toStop) {
                        // public static final int BEAT_TIMEOUT = 30;
                        // 30秒重试一次，保证执行器在调度中心存活
                        TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
                    }
                } catch (InterruptedException e) {
                    if (!toStop) {
                        logger.warn(">>>>>>>>>>> xxl-job, executor registry thread interrupted, error msg:{}", e.getMessage());
                    }
                }
            }

            // registry remove
            try {
                RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
                for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
                    try {
                        ReturnT<String> registryResult = adminBiz.registryRemove(registryParam);
                        if (registryResult!=null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
                            registryResult = ReturnT.SUCCESS;
                            logger.info(">>>>>>>>>>> xxl-job registry-remove success, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                            break;
                        } else {
                            logger.info(">>>>>>>>>>> xxl-job registry-remove fail, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                        }
                    } catch (Exception e) {
                        if (!toStop) {
                            logger.info(">>>>>>>>>>> xxl-job registry-remove error, registryParam:{}", registryParam, e);
                        }

                    }

                }
            } catch (Exception e) {
                if (!toStop) {
                    logger.error(e.getMessage(), e);
                }
            }
            logger.info(">>>>>>>>>>> xxl-job, executor registry thread destroy.");

        }
    });
    registryThread.setDaemon(true);
    registryThread.setName("xxl-job, executor ExecutorRegistryThread");
    registryThread.start();
}
```

 `com.xxl.job.core.biz.client.AdminBizClient#registry`方法发送 http 请求请求调度中心注册接口

```java
public ReturnT<String> registry(RegistryParam registryParam) {
    return XxlJobRemotingUtil.postBody(addressUrl + "api/registry", accessToken, timeout, registryParam, String.class);
}
```

至此完成了客户端侧执行器注册与重试线程的启动。

接下来就是调度器接收执行器注册信息了

#### 调度器注入执行器信息

`com.xxl.job.admin.core.conf.XxlJobAdminConfig` 配置类实现了 InitializingBean 类的 afterPropertiesSet() 方法

```java
public void afterPropertiesSet() throws Exception {
    adminConfig = this;

    xxlJobScheduler = new XxlJobScheduler();
    // 调用 xxlJobeSchedule 的初始化方法
    xxlJobScheduler.init();
}
```

`com.xxl.job.admin.core.scheduler.XxlJobScheduler` 调度器主要的初始化方法都在这里

```java
public void init() throws Exception {
    // init i18n
    // 初始化国际化
    initI18n();

    // admin trigger pool start
    // 触发器线程池启动
    JobTriggerPoolHelper.toStart();

    // admin registry monitor run
    JobRegistryHelper.getInstance().start();

    // admin fail-monitor run
    JobFailMonitorHelper.getInstance().start();

    // admin lose-monitor run ( depend on JobTriggerPoolHelper )
    JobCompleteHelper.getInstance().start();

    // admin log report start
    JobLogReportHelper.getInstance().start();

    // start-schedule  ( depend on JobTriggerPoolHelper )
    // 启动定时任务调度线程
    JobScheduleHelper.getInstance().start();

    logger.info(">>>>>>>>> init xxl-job admin success.");
}
```

`com.xxl.job.admin.core.thread.JobScheduleHelper#start`方法使用了时间轮算法来进行任务的调度

```java
public void start(){

    // schedule thread
    scheduleThread = new Thread(new Runnable() {
        @Override
        public void run() {

            try {
                TimeUnit.MILLISECONDS.sleep(5000 - System.currentTimeMillis()%1000 );
            } catch (InterruptedException e) {
                if (!scheduleThreadToStop) {
                    logger.error(e.getMessage(), e);
                }
            }
            logger.info(">>>>>>>>> init xxl-job admin scheduler success.");

            // pre-read count: treadpool-size * trigger-qps (each trigger cost 50ms, qps = 1000/50 = 20)
            int preReadCount = (XxlJobAdminConfig.getAdminConfig().getTriggerPoolFastMax() + XxlJobAdminConfig.getAdminConfig().getTriggerPoolSlowMax()) * 20;

            while (!scheduleThreadToStop) {

                // Scan Job
                long start = System.currentTimeMillis();

                Connection conn = null;
                Boolean connAutoCommit = null;
                PreparedStatement preparedStatement = null;

                boolean preReadSuc = true;
                try {

                    conn = XxlJobAdminConfig.getAdminConfig().getDataSource().getConnection();
                    connAutoCommit = conn.getAutoCommit();
                    conn.setAutoCommit(false);

                    preparedStatement = conn.prepareStatement(  "select * from xxl_job_lock where lock_name = 'schedule_lock' for update" );
                    preparedStatement.execute();

                    // tx start

                    // 1、pre read
                    long nowTime = System.currentTimeMillis();
                    List<XxlJobInfo> scheduleList = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleJobQuery(nowTime + PRE_READ_MS, preReadCount);
                    if (scheduleList!=null && scheduleList.size()>0) {
                        // 2、push time-ring
                        for (XxlJobInfo jobInfo: scheduleList) {

                            // time-ring jump
                            if (nowTime > jobInfo.getTriggerNextTime() + PRE_READ_MS) {
                                // 2.1、trigger-expire > 5s：pass && make next-trigger-time
                                logger.warn(">>>>>>>>>>> xxl-job, schedule misfire, jobId = " + jobInfo.getId());

                                // 1、misfire match
                                MisfireStrategyEnum misfireStrategyEnum = MisfireStrategyEnum.match(jobInfo.getMisfireStrategy(), MisfireStrategyEnum.DO_NOTHING);
                                if (MisfireStrategyEnum.FIRE_ONCE_NOW == misfireStrategyEnum) {
                                    // FIRE_ONCE_NOW 》 trigger
                                    JobTriggerPoolHelper.trigger(jobInfo.getId(), TriggerTypeEnum.MISFIRE, -1, null, null, null);
                                    logger.debug(">>>>>>>>>>> xxl-job, schedule push trigger : jobId = " + jobInfo.getId() );
                                }

                                // 2、fresh next
                                refreshNextValidTime(jobInfo, new Date());

                            } else if (nowTime > jobInfo.getTriggerNextTime()) {
                                // 2.2、trigger-expire < 5s：direct-trigger && make next-trigger-time

                                // 1、trigger
                                JobTriggerPoolHelper.trigger(jobInfo.getId(), TriggerTypeEnum.CRON, -1, null, null, null);
                                logger.debug(">>>>>>>>>>> xxl-job, schedule push trigger : jobId = " + jobInfo.getId() );

                                // 2、fresh next
                                refreshNextValidTime(jobInfo, new Date());

                                // next-trigger-time in 5s, pre-read again
                                if (jobInfo.getTriggerStatus()==1 && nowTime + PRE_READ_MS > jobInfo.getTriggerNextTime()) {

                                    // 1、make ring second
                                    int ringSecond = (int)((jobInfo.getTriggerNextTime()/1000)%60);

                                    // 2、push time ring
                                    pushTimeRing(ringSecond, jobInfo.getId());

                                    // 3、fresh next
                                    refreshNextValidTime(jobInfo, new Date(jobInfo.getTriggerNextTime()));

                                }

                            } else {
                                // 2.3、trigger-pre-read：time-ring trigger && make next-trigger-time

                                // 1、make ring second
                                int ringSecond = (int)((jobInfo.getTriggerNextTime()/1000)%60);

                                // 2、push time ring
                                pushTimeRing(ringSecond, jobInfo.getId());

                                // 3、fresh next
                                refreshNextValidTime(jobInfo, new Date(jobInfo.getTriggerNextTime()));

                            }

                        }

                        // 3、update trigger info
                        for (XxlJobInfo jobInfo: scheduleList) {
                            XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleUpdate(jobInfo);
                        }

                    } else {
                        preReadSuc = false;
                    }

                    // tx stop


                } catch (Exception e) {
                    if (!scheduleThreadToStop) {
                        logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#scheduleThread error:{}", e);
                    }
                } finally {

                    // commit
                    if (conn != null) {
                        try {
                            conn.commit();
                        } catch (SQLException e) {
                            if (!scheduleThreadToStop) {
                                logger.error(e.getMessage(), e);
                            }
                        }
                        try {
                            conn.setAutoCommit(connAutoCommit);
                        } catch (SQLException e) {
                            if (!scheduleThreadToStop) {
                                logger.error(e.getMessage(), e);
                            }
                        }
                        try {
                            conn.close();
                        } catch (SQLException e) {
                            if (!scheduleThreadToStop) {
                                logger.error(e.getMessage(), e);
                            }
                        }
                    }

                    // close PreparedStatement
                    if (null != preparedStatement) {
                        try {
                            preparedStatement.close();
                        } catch (SQLException e) {
                            if (!scheduleThreadToStop) {
                                logger.error(e.getMessage(), e);
                            }
                        }
                    }
                }
                long cost = System.currentTimeMillis()-start;


                // Wait seconds, align second
                if (cost < 1000) {  // scan-overtime, not wait
                    try {
                        // pre-read period: success > scan each second; fail > skip this period;
                        TimeUnit.MILLISECONDS.sleep((preReadSuc?1000:PRE_READ_MS) - System.currentTimeMillis()%1000);
                    } catch (InterruptedException e) {
                        if (!scheduleThreadToStop) {
                            logger.error(e.getMessage(), e);
                        }
                    }
                }

            }

            logger.info(">>>>>>>>>>> xxl-job, JobScheduleHelper#scheduleThread stop");
        }
    });
    scheduleThread.setDaemon(true);
    scheduleThread.setName("xxl-job, admin JobScheduleHelper#scheduleThread");
    scheduleThread.start();


    // ring thread
    ringThread = new Thread(new Runnable() {
        @Override
        public void run() {

            while (!ringThreadToStop) {

                // align second
                try {
                    TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis() % 1000);
                } catch (InterruptedException e) {
                    if (!ringThreadToStop) {
                        logger.error(e.getMessage(), e);
                    }
                }

                try {
                    // second data
                    List<Integer> ringItemData = new ArrayList<>();
                    int nowSecond = Calendar.getInstance().get(Calendar.SECOND);   // 避免处理耗时太长，跨过刻度，向前校验一个刻度；
                    for (int i = 0; i < 2; i++) {
                        List<Integer> tmpData = ringData.remove( (nowSecond+60-i)%60 );
                        if (tmpData != null) {
                            ringItemData.addAll(tmpData);
                        }
                    }

                    // ring trigger
                    logger.debug(">>>>>>>>>>> xxl-job, time-ring beat : " + nowSecond + " = " + Arrays.asList(ringItemData) );
                    if (ringItemData.size() > 0) {
                        // do trigger
                        for (int jobId: ringItemData) {
                            // do trigger
                            // 真正触发任务调度的方法
                            JobTriggerPoolHelper.trigger(jobId, TriggerTypeEnum.CRON, -1, null, null, null);
                        }
                        // clear
                        ringItemData.clear();
                    }
                } catch (Exception e) {
                    if (!ringThreadToStop) {
                        logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#ringThread error:{}", e);
                    }
                }
            }
            logger.info(">>>>>>>>>>> xxl-job, JobScheduleHelper#ringThread stop");
        }
    });
    ringThread.setDaemon(true);
    ringThread.setName("xxl-job, admin JobScheduleHelper#ringThread");
    ringThread.start();
}
```

`com.xxl.job.admin.core.thread.JobTriggerPoolHelper#addTrigger` 调用这个方法

```java
public void addTrigger(final int jobId,
                           final TriggerTypeEnum triggerType,
                           final int failRetryCount,
                           final String executorShardingParam,
                           final String executorParam,
                           final String addressList) {

    // choose thread pool
    ThreadPoolExecutor triggerPool_ = fastTriggerPool;
    AtomicInteger jobTimeoutCount = jobTimeoutCountMap.get(jobId);
    if (jobTimeoutCount!=null && jobTimeoutCount.get() > 10) {      // job-timeout 10 times in 1 min
        triggerPool_ = slowTriggerPool;
    }

    // trigger
    // 使用触发器线程池执行任务
    triggerPool_.execute(new Runnable() {
        @Override
        public void run() {

            long start = System.currentTimeMillis();

            try {
                // do trigger
                // 最终是调用 com.xxl.job.admin.core.trigger.XxlJobTrigger#processTrigger 方法触发任务
                XxlJobTrigger.trigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam, addressList);
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            } finally {

                // check timeout-count-map
                long minTim_now = System.currentTimeMillis()/60000;
                if (minTim != minTim_now) {
                    minTim = minTim_now;
                    jobTimeoutCountMap.clear();
                }

                // incr timeout-count-map
                long cost = System.currentTimeMillis()-start;
                if (cost > 500) {       // ob-timeout threshold 500ms
                    AtomicInteger timeoutCount = jobTimeoutCountMap.putIfAbsent(jobId, new AtomicInteger(1));
                    if (timeoutCount != null) {
                        timeoutCount.incrementAndGet();
                    }
                }

            }

        }
    });
}
```

`com.xxl.job.admin.core.trigger.XxlJobTrigger#processTrigger`方法调用 `com.xxl.job.admin.core.trigger.XxlJobTrigger#runExecutor`

```java
public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
    ReturnT<String> runResult = null;
    try {
        // 创建ExecutorBizClient
        ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
        // 调用 com.xxl.job.core.biz.client.ExecutorBizClient#run或com.xxl.job.core.biz.impl.ExecutorBizImpl#run 方法
        runResult = executorBiz.run(triggerParam);
    } catch (Exception e) {
        logger.error(">>>>>>>>>>> xxl-job trigger error, please check if the executor[{}] is running.", address, e);
        runResult = new ReturnT<String>(ReturnT.FAIL_CODE, ThrowableUtil.toString(e));
    }

    StringBuffer runResultSB = new StringBuffer(I18nUtil.getString("jobconf_trigger_run") + "：");
    runResultSB.append("<br>address：").append(address);
    runResultSB.append("<br>code：").append(runResult.getCode());
    runResultSB.append("<br>msg：").append(runResult.getMsg());

    runResult.setMsg(runResultSB.toString());
    return runResult;
}
```

`com.xxl.job.core.biz.client.ExecutorBizClient#run`方法直接调用执行器接口执行任务

```java
public ReturnT<String> run(TriggerParam triggerParam) {
    return XxlJobRemotingUtil.postBody(addressUrl + "run", accessToken, timeout, triggerParam, String.class);
}
```

#### 执行器执行任务

`com.xxl.job.core.server.EmbedServer#start`方法创建 netty 服务时添加了 EmbedHttpServerHandler 处理器，因此调度器触发任务最终会进入`com.xxl.job.core.server.EmbedServer.EmbedHttpServerHandler#channelRead0`中

最终会进入 `com.xxl.job.core.server.EmbedServer.EmbedHttpServerHandler#process`方法

```java
private Object process(HttpMethod httpMethod, String uri, String requestData, String accessTokenReq) {
    // valid
    if (HttpMethod.POST != httpMethod) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, HttpMethod not support.");
    }
    if (uri == null || uri.trim().length() == 0) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping empty.");
    }
    if (accessToken != null
            && accessToken.trim().length() > 0
            && !accessToken.equals(accessTokenReq)) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "The access token is wrong.");
    }

    // services mapping
    try {
        switch (uri) {
            case "/beat":
                return executorBiz.beat();
            case "/idleBeat":
                IdleBeatParam idleBeatParam = GsonTool.fromJson(requestData, IdleBeatParam.class);
                return executorBiz.idleBeat(idleBeatParam);
            case "/run":
                // 触发任务
                TriggerParam triggerParam = GsonTool.fromJson(requestData, TriggerParam.class);
                return executorBiz.run(triggerParam);
            case "/kill":
                KillParam killParam = GsonTool.fromJson(requestData, KillParam.class);
                return executorBiz.kill(killParam);
            case "/log":
                LogParam logParam = GsonTool.fromJson(requestData, LogParam.class);
                return executorBiz.log(logParam);
            default:
                return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping(" + uri + ") not found.");
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
        return new ReturnT<String>(ReturnT.FAIL_CODE, "request error:" + ThrowableUtil.toString(e));
    }
}
```

在执行器端，就要走到 ExecutorBizImpl#run 方法，调用本地的 handler 方法执行任务

`com.xxl.job.core.biz.impl.ExecutorBizImpl#run`方法里会根据不同执行器类型获取不同的 jobHandler，然后使用 `com.xxl.job.core.executor.XxlJobExecutor#registJobThread`方法创建 jobThread 线程

```java
public static JobThread registJobThread(int jobId, IJobHandler handler, String removeOldReason){
    JobThread newJobThread = new JobThread(jobId, handler);
    // 启动job线程
    newJobThread.start();
    logger.info(">>>>>>>>>>> xxl-job regist JobThread success, jobId:{}, handler:{}", new Object[]{jobId, handler});

    JobThread oldJobThread = jobThreadRepository.put(jobId, newJobThread);  // putIfAbsent | oh my god, map's put method return the old value!!!
    if (oldJobThread != null) {
        oldJobThread.toStop(removeOldReason);
        oldJobThread.interrupt();
    }

    return newJobThread;
}
```

`com.xxl.job.core.thread.JobThread#run`

```java
public void run() {

    // init
    try {
       handler.init();
    } catch (Throwable e) {
       logger.error(e.getMessage(), e);
    }

    // execute
    while(!toStop){
       running = false;
       idleTimes++;

           TriggerParam triggerParam = null;
           try {
          // to check toStop signal, we need cycle, so wo cannot use queue.take(), instand of poll(timeout)
          triggerParam = triggerQueue.poll(3L, TimeUnit.SECONDS);
          if (triggerParam!=null) {
             running = true;
             idleTimes = 0;
             triggerLogIdSet.remove(triggerParam.getLogId());

             // log filename, like "logPath/yyyy-MM-dd/9999.log"
             String logFileName = XxlJobFileAppender.makeLogFileName(new Date(triggerParam.getLogDateTime()), triggerParam.getLogId());
             XxlJobContext xxlJobContext = new XxlJobContext(
                   triggerParam.getJobId(),
                   triggerParam.getExecutorParams(),
                   logFileName,
                   triggerParam.getBroadcastIndex(),
                   triggerParam.getBroadcastTotal());

             // init job context
             XxlJobContext.setXxlJobContext(xxlJobContext);

             // execute
             XxlJobHelper.log("<br>----------- xxl-job job execute start -----------<br>----------- Param:" + xxlJobContext.getJobParam());

             if (triggerParam.getExecutorTimeout() > 0) {
                // limit timeout
                Thread futureThread = null;
                try {
                   FutureTask<Boolean> futureTask = new FutureTask<Boolean>(new Callable<Boolean>() {
                      @Override
                      public Boolean call() throws Exception {

                         // init job context
                         XxlJobContext.setXxlJobContext(xxlJobContext);
						// 反射调用实际 handler 执行
                         handler.execute();
                         return true;
                      }
                   });
                   futureThread = new Thread(futureTask);
                   futureThread.start();

                   Boolean tempResult = futureTask.get(triggerParam.getExecutorTimeout(), TimeUnit.SECONDS);
                } catch (TimeoutException e) {

                   XxlJobHelper.log("<br>----------- xxl-job job execute timeout");
                   XxlJobHelper.log(e);

                   // handle result
                   XxlJobHelper.handleTimeout("job execute timeout ");
                } finally {
                   futureThread.interrupt();
                }
             } else {
                // just execute
                handler.execute();
             }

             ......

    logger.info(">>>>>>>>>>> xxl-job JobThread stoped, hashCode:{}", Thread.currentThread());
}
```