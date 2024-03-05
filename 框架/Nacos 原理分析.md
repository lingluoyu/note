#### Nacos 源码

##### Nacos config client

```java
// SpringBoot 启动时调用 NacosConfigBootstrapConfiguration 初始化配置类
// SpringBoot 启动初始化 Bean，调用 NacosConfigBootstrapConfiguration 中的 nacosConfigManager 创建NacosConfigManager
@Bean
@ConditionalOnMissingBean
public NacosConfigManager nacosConfigManager(
        NacosConfigProperties nacosConfigProperties) {
    return new NacosConfigManager(nacosConfigProperties);
}
// NacosConfigManager 初始化方法，通过 NacosConfigProperties 配置初始化 configService
public NacosConfigManager(NacosConfigProperties nacosConfigProperties) {
    this.nacosConfigProperties = nacosConfigProperties;
    // Compatible with older code in NacosConfigProperties,It will be deleted in the
    // future.
    createConfigService(nacosConfigProperties);
}

/**
 * Compatible with old design,It will be perfected in the future.
 */
static ConfigService createConfigService(
        NacosConfigProperties nacosConfigProperties) {
    if (Objects.isNull(service)) {
        synchronized (NacosConfigManager.class) {// 加锁，防止重复创建
            try {
                if (Objects.isNull(service)) {
                    // 调用 NacosFactory 静态方法创建 ConfigService
                    service = NacosFactory.createConfigService(
                            nacosConfigProperties.assembleConfigServiceProperties());
                }
            }
            catch (NacosException e) {
                log.error(e.getMessage());
                throw new NacosConnectionFailureException(
                        nacosConfigProperties.getServerAddr(), e.getMessage(), e);
            }
        }
    }
    return service;
}
// NacosFactory
public static ConfigService createConfigService(Properties properties) throws NacosException {
    return ConfigFactory.createConfigService(properties);
}
// 反射调用 NacosConfigService 构造方法
public static ConfigService createConfigService(Properties properties) throws NacosException {
    try {
        Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
        Constructor constructor = driverImplClass.getConstructor(Properties.class);
        ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
        return vendorImpl;
    } catch (Throwable e) {
        throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
    }
}
// NacosConfigService
public NacosConfigService(Properties properties) throws NacosException {
    PreInitUtils.asyncPreLoadCostComponent();
    final NacosClientProperties clientProperties = NacosClientProperties.PROTOTYPE.derive(properties);
    ValidatorUtils.checkInitParam(clientProperties);

    initNamespace(clientProperties);
    this.configFilterChainManager = new ConfigFilterChainManager(clientProperties.asProperties());
    ServerListManager serverListManager = new ServerListManager(clientProperties);
    serverListManager.start();
	// 创建 ClientWorker
    this.worker = new ClientWorker(this.configFilterChainManager, serverListManager, clientProperties);
    // will be deleted in 2.0 later versions
    agent = new ServerHttpAgent(serverListManager);

}
// ClientWorker
public ClientWorker(final ConfigFilterChainManager configFilterChainManager, ServerListManager serverListManager,
            final NacosClientProperties properties) throws NacosException {
    this.configFilterChainManager = configFilterChainManager;

    init(properties);
	// 创建 ConfigTransportClient，与配置中心通信，底层为 HttpCLient
    agent = new ConfigRpcTransportClient(properties, serverListManager);
    int count = ThreadUtils.getSuitableThreadCount(THREAD_MULTIPLE);
    // 启动定时的 worker 线程，从配置中心拉取配置信息
    ScheduledExecutorService executorService = Executors.newScheduledThreadPool(Math.max(count, MIN_THREAD_NUM),
            r -> {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker");
                t.setDaemon(true);
                return t;
            });
    agent.setExecutor(executorService);
    // 调用ConfigTransportClient启动线程
    agent.start();
}

// ConfigTransportClient
public void start() throws NacosException {
    securityProxy.login(this.properties);
    // 默认 5s 执行一次
    this.executor.scheduleWithFixedDelay(() -> securityProxy.login(properties), 0,
            this.securityInfoRefreshIntervalMills, TimeUnit.MILLISECONDS);
    // 调用 ClientWorker 方法，监听配置信息变更
    startInternal();
}
// ClientWorker
@Override
public void startInternal() {
    executor.schedule(() -> {
        while (!executor.isShutdown() && !executor.isTerminated()) {
            try {
                listenExecutebell.poll(5L, TimeUnit.SECONDS);
                if (executor.isShutdown() || executor.isTerminated()) {
                    continue;
                }
                // 执行配置信息变更监听
                executeConfigListen();
            } catch (Throwable e) {
                LOGGER.error("[rpc listen execute] [rpc listen] exception", e);
                try {
                    Thread.sleep(50L);
                } catch (InterruptedException interruptedException) {
                    //ignore
                }
                notifyListenConfig();
            }
        }
    }, 0L, TimeUnit.MILLISECONDS);

}

@Override
public void executeConfigListen() {

    Map<String, List<CacheData>> listenCachesMap = new HashMap<>(16);
    Map<String, List<CacheData>> removeListenCachesMap = new HashMap<>(16);
    long now = System.currentTimeMillis();
    boolean needAllSync = now - lastAllSyncTime >= ALL_SYNC_INTERNAL;
    for (CacheData cache : cacheMap.get().values()) {

        synchronized (cache) {

            checkLocalConfig(cache);

            // check local listeners consistent.
            if (cache.isConsistentWithServer()) {
                // 检查 MD5 是否匹配，若不匹配，则触发通知 listener 更新配置
                cache.checkListenerMd5();
                if (!needAllSync) {
                    continue;
                }
            }

            // If local configuration information is used, then skip the processing directly.
            if (cache.isUseLocalConfigInfo()) {
                continue;
            }

            if (!cache.isDiscard()) {
                List<CacheData> cacheDatas = listenCachesMap.computeIfAbsent(String.valueOf(cache.getTaskId()),
                        k -> new LinkedList<>());
                cacheDatas.add(cache);
            } else {
                List<CacheData> cacheDatas = removeListenCachesMap.computeIfAbsent(
                        String.valueOf(cache.getTaskId()), k -> new LinkedList<>());
                cacheDatas.add(cache);
            }
        }

    }

    //execute check listen ,return true if has change keys.
    // 拉取配置中心配置信息
    boolean hasChangedKeys = checkListenCache(listenCachesMap);

    //execute check remove listen.
    checkRemoveListenCache(removeListenCachesMap);

    if (needAllSync) {
        lastAllSyncTime = now;
    }
    //If has changed keys,notify re sync md5.
    /**
     * 若有配置发生了变更，则调用 notifyListenConfig() 方法，向 listenExecutebell 队列中插入一个对象，表示配置发生了变更
     * @Override
        public void notifyListenConfig() {
            listenExecutebell.offer(bellItem);
        }
     * 此时 startInternal() 方法定时触发，若阻塞队列 listenExecutebell 中存在对象，则可进行下一次配置更新检查
     */
    if (hasChangedKeys) {
        notifyListenConfig();
    }

}

private boolean checkListenCache(Map<String, List<CacheData>> listenCachesMap) {
        
        final AtomicBoolean hasChangedKeys = new AtomicBoolean(false);
        if (!listenCachesMap.isEmpty()) {
            List<Future> listenFutures = new ArrayList<>();
            for (Map.Entry<String, List<CacheData>> entry : listenCachesMap.entrySet()) {
                String taskId = entry.getKey();
                ExecutorService executorService = ensureSyncExecutor(taskId);
                Future future = executorService.submit(() -> {
                    List<CacheData> listenCaches = entry.getValue();
                    //reset notify change flag.
                    for (CacheData cacheData : listenCaches) {
                        cacheData.getReceiveNotifyChanged().set(false);
                    }
                    ConfigBatchListenRequest configChangeListenRequest = buildConfigRequest(listenCaches);
                    configChangeListenRequest.setListen(true);
                    try {
                        // 通过 RPC 调用获取配置信息
                        RpcClient rpcClient = ensureRpcClient(taskId);
                        ConfigChangeBatchListenResponse listenResponse = (ConfigChangeBatchListenResponse) requestProxy(
                                rpcClient, configChangeListenRequest);
                        if (listenResponse != null && listenResponse.isSuccess()) {

                            Set<String> changeKeys = new HashSet<String>();

                            List<ConfigChangeBatchListenResponse.ConfigContext> changedConfigs = listenResponse.getChangedConfigs();
                            //handle changed keys,notify listener
                            if (!CollectionUtils.isEmpty(changedConfigs)) {
                                hasChangedKeys.set(true);
                                for (ConfigChangeBatchListenResponse.ConfigContext changeConfig : changedConfigs) {
                                    String changeKey = GroupKey.getKeyTenant(changeConfig.getDataId(),
                                            changeConfig.getGroup(), changeConfig.getTenant());
                                    changeKeys.add(changeKey);
                                    boolean isInitializing = cacheMap.get().get(changeKey).isInitializing();
                                    refreshContentAndCheck(changeKey, !isInitializing);
                                }

                            }

                            for (CacheData cacheData : listenCaches) {
                                if (cacheData.getReceiveNotifyChanged().get()) {
                                    String changeKey = GroupKey.getKeyTenant(cacheData.dataId, cacheData.group,
                                            cacheData.getTenant());
                                    if (!changeKeys.contains(changeKey)) {
                                        boolean isInitializing = cacheMap.get().get(changeKey).isInitializing();
                                        refreshContentAndCheck(changeKey, !isInitializing);
                                    }
                                }
                            }

                            //handler content configs
                            for (CacheData cacheData : listenCaches) {
                                cacheData.setInitializing(false);
                                String groupKey = GroupKey.getKeyTenant(cacheData.dataId, cacheData.group,
                                        cacheData.getTenant());
                                if (!changeKeys.contains(groupKey)) {
                                    synchronized (cacheData) {
                                        if (!cacheData.getReceiveNotifyChanged().get()) {
                                            cacheData.setConsistentWithServer(true);
                                        }
                                    }
                                }
                            }

                        }
                    } catch (Throwable e) {
                        LOGGER.error("Execute listen config change error ", e);
                        try {
                            Thread.sleep(50L);
                        } catch (InterruptedException interruptedException) {
                            //ignore
                        }
                        notifyListenConfig();
                    }
                });
                listenFutures.add(future);

            }
            for (Future future : listenFutures) {
                try {
                    future.get();
                } catch (Throwable throwable) {
                    LOGGER.error("Async listen config change error ", throwable);
                }
            }

        }
        return hasChangedKeys.get();
    }
// CacheData
void checkListenerMd5() {
    for (ManagerListenerWrap wrap : listeners) {
        if (!md5.equals(wrap.lastCallMd5)) {
            safeNotifyListener(dataId, group, content, type, md5, encryptedDataKey, wrap);
        }
    }
}

private void safeNotifyListener(final String dataId, final String group, final String content, final String type,
            final String md5, final String encryptedDataKey, final ManagerListenerWrap listenerWrap) {
    final Listener listener = listenerWrap.listener;
    if (listenerWrap.inNotifying) {
        LOGGER.warn(
                "[{}] [notify-currentSkip] dataId={}, group={},tenant={}, md5={}, listener={}, listener is not finish yet,will try next time.",
                envName, dataId, group, tenant, md5, listener);
        return;
    }
    NotifyTask job = new NotifyTask() {

        @Override
        public void run() {
            long start = System.currentTimeMillis();
            ClassLoader myClassLoader = Thread.currentThread().getContextClassLoader();
            ClassLoader appClassLoader = listener.getClass().getClassLoader();
            ScheduledFuture<?> timeSchedule = null;

            try {
                if (listener instanceof AbstractSharedListener) {
                    AbstractSharedListener adapter = (AbstractSharedListener) listener;
                    adapter.fillContext(dataId, group);
                    LOGGER.info("[{}] [notify-context] dataId={}, group={},tenant={}, md5={}", envName, dataId,
                            group, tenant, md5);
                }
                // Before executing the callback, set the thread classloader to the classloader of
                // the specific webapp to avoid exceptions or misuses when calling the spi interface in
                // the callback method (this problem occurs only in multi-application deployment).
                Thread.currentThread().setContextClassLoader(appClassLoader);

                ConfigResponse cr = new ConfigResponse();
                cr.setDataId(dataId);
                cr.setGroup(group);
                cr.setContent(content);
                cr.setEncryptedDataKey(encryptedDataKey);
                configFilterChainManager.doFilter(null, cr);
                String contentTmp = cr.getContent();
                timeSchedule = getNotifyBlockMonitor().schedule(
                        new LongNotifyHandler(listener.getClass().getSimpleName(), dataId, group, tenant, md5,
                                notifyWarnTimeout, Thread.currentThread()), notifyWarnTimeout,
                        TimeUnit.MILLISECONDS);
                listenerWrap.inNotifying = true;
                listener.receiveConfigInfo(contentTmp);
                // compare lastContent and content
                if (listener instanceof AbstractConfigChangeListener) {
                    Map<String, ConfigChangeItem> data = ConfigChangeHandler.getInstance()
                            .parseChangeData(listenerWrap.lastContent, contentTmp, type);
                    ConfigChangeEvent event = new ConfigChangeEvent(data);
                    ((AbstractConfigChangeListener) listener).receiveConfigChange(event);
                    listenerWrap.lastContent = contentTmp;
                }

                listenerWrap.lastCallMd5 = md5;
                LOGGER.info(
                        "[{}] [notify-ok] dataId={}, group={},tenant={}, md5={}, listener={} ,job run cost={} millis.",
                        envName, dataId, group, tenant, md5, listener, (System.currentTimeMillis() - start));
            } catch (NacosException ex) {
                LOGGER.error(
                        "[{}] [notify-error] dataId={}, group={},tenant={},md5={}, listener={} errCode={} errMsg={},stackTrace :{}",
                        envName, dataId, group, tenant, md5, listener, ex.getErrCode(), ex.getErrMsg(),
                        getTrace(ex.getStackTrace(), 3));
            } catch (Throwable t) {
                LOGGER.error("[{}] [notify-error] dataId={}, group={},tenant={}, md5={}, listener={} tx={}",
                        envName, dataId, group, tenant, md5, listener, getTrace(t.getStackTrace(), 3));
            } finally {
                listenerWrap.inNotifying = false;
                Thread.currentThread().setContextClassLoader(myClassLoader);
                if (timeSchedule != null) {
                    timeSchedule.cancel(true);
                }
            }
        }
    };

    try {
        if (null != listener.getExecutor()) {
            LOGGER.info(
                    "[{}] [notify-listener] task submitted to user executor, dataId={}, group={},tenant={}, md5={}, listener={} ",
                    envName, dataId, group, tenant, md5, listener);
            job.async = true;
            listener.getExecutor().execute(job);
        } else {
            LOGGER.info(
                    "[{}] [notify-listener] task execute in nacos thread, dataId={}, group={},tenant={}, md5={}, listener={} ",
                    envName, dataId, group, tenant, md5, listener);
            job.run();
        }
    } catch (Throwable t) {
        LOGGER.error("[{}] [notify-listener-error] dataId={}, group={},tenant={}, md5={}, listener={} throwable={}",
                envName, dataId, group, tenant, md5, listener, t.getCause());
    }
}
```

