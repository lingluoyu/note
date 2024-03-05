### Nacos 源码

#### Nacos config client

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
                // 触发回调
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
            ......
        }
    };

    ......
}
    
// com.alibaba.nacos.client.config.listener.impl.PropertiesListener
public abstract class PropertiesListener extends AbstractListener {
    
    private static final Logger LOGGER = LogUtils.logger(PropertiesListener.class);
    
    @Override
    public void receiveConfigInfo(String configInfo) {
        if (StringUtils.isEmpty(configInfo)) {
            return;
        }
        
        Properties properties = new Properties();
        try {
            properties.load(new StringReader(configInfo));
            // 最终是调用 listener 的抽象方法
            innerReceive(properties);
        } catch (IOException e) {
            LOGGER.error("load properties error：" + configInfo, e);
        }
        
    }
    
    /**
     * properties type for receiver.
     *
     * @param properties properties
     */
    public abstract void innerReceive(Properties properties);
    
}
// 通过NacosContextRefresher类的registerNacosListener方法来去实现innerReceive这个抽象方法
// com.alibaba.cloud.nacos.refresh.NacosContextRefresher#registerNacosListener
private void registerNacosListener(final String groupKey, final String dataKey) {
		String key = NacosPropertySourceRepository.getMapKey(dataKey, groupKey);
		Listener listener = listenerMap.computeIfAbsent(key,
				lst -> new AbstractSharedListener() {
					@Override
					public void innerReceive(String dataId, String group,
							String configInfo) {
						refreshCountIncrement();
						nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo);
                        // 发布配置刷新事件，实现配置刷新
						applicationContext.publishEvent(
								new RefreshEvent(this, null, "Refresh Nacos config"));
						if (log.isDebugEnabled()) {
							log.debug(String.format(
									"Refresh Nacos config group=%s,dataId=%s,configInfo=%s",
									group, dataId, configInfo));
						}
					}
				});
		try {
			configService.addListener(dataKey, groupKey, listener);
			log.info("[Nacos Config] Listening config: dataId={}, group={}", dataKey,
					groupKey);
		}
		catch (NacosException e) {
			log.warn(String.format(
					"register fail for nacos listener ,dataId=[%s],group=[%s]", dataKey,
					groupKey), e);
		}
	}
```

使用 spring-cloud 刷新配置信息

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(name = "spring.cloud.nacos.config.enabled", matchIfMissing = true)
public class NacosConfigAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(value = NacosConfigProperties.class, search = SearchStrategy.CURRENT)
	public NacosConfigProperties nacosConfigProperties(ApplicationContext context) {
		if (context.getParent() != null
				&& BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
						context.getParent(), NacosConfigProperties.class).length > 0) {
			return BeanFactoryUtils.beanOfTypeIncludingAncestors(context.getParent(),
					NacosConfigProperties.class);
		}
		return new NacosConfigProperties();
	}

	@Bean
	public NacosRefreshHistory nacosRefreshHistory() {
		return new NacosRefreshHistory();
	}

	@Bean
	public NacosConfigManager nacosConfigManager(
			NacosConfigProperties nacosConfigProperties) {
		return new NacosConfigManager(nacosConfigProperties);
	}

	@Bean
	public NacosContextRefresher nacosContextRefresher(
			NacosConfigManager nacosConfigManager,
			NacosRefreshHistory nacosRefreshHistory) {
		// Consider that it is not necessary to be compatible with the previous
		// configuration
		// and use the new configuration if necessary.
        // 创建 nacos 上下文刷新器
		return new NacosContextRefresher(nacosConfigManager, nacosRefreshHistory);
	}

	@Bean
	@ConditionalOnMissingBean(search = SearchStrategy.CURRENT)
	@ConditionalOnNonDefaultBehavior
	public ConfigurationPropertiesRebinder smartConfigurationPropertiesRebinder(
			ConfigurationPropertiesBeans beans) {
		// If using default behavior, not use SmartConfigurationPropertiesRebinder.
		// Minimize te possibility of making mistakes.
		return new SmartConfigurationPropertiesRebinder(beans);
	}

}
```

```java
public class NacosContextRefresher
		implements ApplicationListener<ApplicationReadyEvent>, ApplicationContextAware {

	private final static Logger log = LoggerFactory
			.getLogger(NacosContextRefresher.class);

	private static final AtomicLong REFRESH_COUNT = new AtomicLong(0);

	private NacosConfigProperties nacosConfigProperties;

	private final boolean isRefreshEnabled;

	private final NacosRefreshHistory nacosRefreshHistory;

	private final ConfigService configService;

	private ApplicationContext applicationContext;

	private AtomicBoolean ready = new AtomicBoolean(false);

	private Map<String, Listener> listenerMap = new ConcurrentHashMap<>(16);

	public NacosContextRefresher(NacosConfigManager nacosConfigManager,
			NacosRefreshHistory refreshHistory) {
		this.nacosConfigProperties = nacosConfigManager.getNacosConfigProperties();
		this.nacosRefreshHistory = refreshHistory;
		this.configService = nacosConfigManager.getConfigService();
		this.isRefreshEnabled = this.nacosConfigProperties.isRefreshEnabled();
	}

	@Override
	public void onApplicationEvent(ApplicationReadyEvent event) {
		// many Spring context
		if (this.ready.compareAndSet(false, true)) {
            // 注册 nacos 监听
			this.registerNacosListenersForApplications();
		}
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
	}

	/**
	 * register Nacos Listeners.
	 */
	private void registerNacosListenersForApplications() {
		if (isRefreshEnabled()) {
			for (NacosPropertySource propertySource : NacosPropertySourceRepository
					.getAll()) {
				if (!propertySource.isRefreshable()) {
					continue;
				}
				String dataId = propertySource.getDataId();
                // 注册 nacos 监听
				registerNacosListener(propertySource.getGroup(), dataId);
			}
		}
	}

	private void registerNacosListener(final String groupKey, final String dataKey) {
		String key = NacosPropertySourceRepository.getMapKey(dataKey, groupKey);
		Listener listener = listenerMap.computeIfAbsent(key,
				lst -> new AbstractSharedListener() {
					@Override
					public void innerReceive(String dataId, String group,
							String configInfo) {
						refreshCountIncrement();
						nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo);
                        // 当有配置信息变动的时候，通知spring cloud 刷新事件
						applicationContext.publishEvent(
								new RefreshEvent(this, null, "Refresh Nacos config"));
						if (log.isDebugEnabled()) {
							log.debug(String.format(
									"Refresh Nacos config group=%s,dataId=%s,configInfo=%s",
									group, dataId, configInfo));
						}
					}
				});
		try {
			configService.addListener(dataKey, groupKey, listener);
			log.info("[Nacos Config] Listening config: dataId={}, group={}", dataKey,
					groupKey);
		}
		catch (NacosException e) {
			log.warn(String.format(
					"register fail for nacos listener ,dataId=[%s],group=[%s]", dataKey,
					groupKey), e);
		}
	}

	public NacosConfigProperties getNacosConfigProperties() {
		return nacosConfigProperties;
	}

	public NacosContextRefresher setNacosConfigProperties(
			NacosConfigProperties nacosConfigProperties) {
		this.nacosConfigProperties = nacosConfigProperties;
		return this;
	}

	public boolean isRefreshEnabled() {
		if (null == nacosConfigProperties) {
			return isRefreshEnabled;
		}
		// Compatible with older configurations
		if (nacosConfigProperties.isRefreshEnabled() && !isRefreshEnabled) {
			return false;
		}
		return isRefreshEnabled;
	}

	public static long getRefreshCount() {
		return REFRESH_COUNT.get();
	}

	public static void refreshCountIncrement() {
		REFRESH_COUNT.incrementAndGet();
	}

}
```

1. 在`spring.factories`中配置自动配置类`NacosConfigAutoConfiguration`
2. `NacosConfigAutoConfiguration`创建`NacosContextRefresher`
3. `NacosContextRefresher`监听`ApplicationReadyEvent`事件
4. 触发`ApplicationReadyEvent`事件后注册监听，创建监听配置的`dataId`，`group`对应的配置信息的监听类
5. `configService`绑定监听器，如果监听到配置信息的变化，则发布`RefreshEvent`
6. `spring cloud`监听到`RefreshEvent`后刷新配置，刷新spring容器

#### Nacos Discovery

##### 服务注册

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnDiscoveryEnabled
@ConditionalOnNacosDiscoveryEnabled
public class NacosServiceAutoConfiguration {

    // spring boot 启动时创建 NacosServiceManager，创建 NamingService
	@Bean
	public NacosServiceManager nacosServiceManager() {
		return new NacosServiceManager();
	}
}

public class NacosServiceManager {

	private static final Logger log = LoggerFactory.getLogger(NacosServiceManager.class);

	private NacosDiscoveryProperties nacosDiscoveryProperties;

	private volatile NamingService namingService;

	private volatile NamingMaintainService namingMaintainService;

    // 创建 NamingService
	public NamingService getNamingService() {
		if (Objects.isNull(this.namingService)) {
			buildNamingService(nacosDiscoveryProperties.getNacosProperties());
		}
		return namingService;
	}

	@Deprecated
	public NamingService getNamingService(Properties properties) {
		if (Objects.isNull(this.namingService)) {
			buildNamingService(properties);
		}
		return namingService;
	}

	public NamingMaintainService getNamingMaintainService(Properties properties) {
		if (Objects.isNull(namingMaintainService)) {
			buildNamingMaintainService(properties);
		}
		return namingMaintainService;
	}

	public boolean isNacosDiscoveryInfoChanged(
			NacosDiscoveryProperties currentNacosDiscoveryPropertiesCache) {
		if (Objects.isNull(this.nacosDiscoveryProperties)
				|| this.nacosDiscoveryProperties.equals(currentNacosDiscoveryPropertiesCache)) {
			return false;
		}
		return true;
	}

	private NamingMaintainService buildNamingMaintainService(Properties properties) {
		if (Objects.isNull(namingMaintainService)) {
			synchronized (NacosServiceManager.class) {
				if (Objects.isNull(namingMaintainService)) {
					namingMaintainService = createNamingMaintainService(properties);
				}
			}
		}
		return namingMaintainService;
	}

	private NamingService buildNamingService(Properties properties) {
		if (Objects.isNull(namingService)) {
			synchronized (NacosServiceManager.class) {
				if (Objects.isNull(namingService)) {
					namingService = createNewNamingService(properties);
				}
			}
		}
		return namingService;
	}

	private NamingService createNewNamingService(Properties properties) {
		try {
            // 调用 NacosFactory.createNamingService() 方法创建 NamingService
			return createNamingService(properties);
		}
		catch (NacosException e) {
			throw new RuntimeException(e);
		}
	}

	private NamingMaintainService createNamingMaintainService(Properties properties) {
		try {
			return createMaintainService(properties);
		}
		catch (NacosException e) {
			throw new RuntimeException(e);
		}
	}

	public void nacosServiceShutDown() throws NacosException {
		if (Objects.nonNull(this.namingService)) {
			this.namingService.shutDown();
			this.namingService = null;
		}
		if (Objects.nonNull(this.namingMaintainService)) {
			this.namingMaintainService.shutDown();
			this.namingMaintainService = null;
		}
	}

	public void setNacosDiscoveryProperties(NacosDiscoveryProperties nacosDiscoveryProperties) {
		this.nacosDiscoveryProperties = nacosDiscoveryProperties;
	}
}
```

SpringCloud Nacos 实现了 ServiceRegistry，NacosServiceRegistry 的 register 方法实现服务注册功能

``` java
@Override
public void register(Registration registration) {

    if (StringUtils.isEmpty(registration.getServiceId())) {
        log.warn("No service to register for nacos client...");
        return;
    }
	// 获取 NamingService
    NamingService namingService = namingService();
    String serviceId = registration.getServiceId();
    String group = nacosDiscoveryProperties.getGroup();

    // 获取注册中心地址信息
    Instance instance = getNacosInstanceFromRegistration(registration);

    try {
        // 服务注册
        namingService.registerInstance(serviceId, group, instance);
        log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
                instance.getIp(), instance.getPort());
    }
    catch (Exception e) {
        if (nacosDiscoveryProperties.isFailFast()) {
            log.error("nacos registry, {} register failed...{},", serviceId,
                    registration.toString(), e);
            rethrowRuntimeException(e);
        }
        else {
            log.warn("Failfast is false. {} register failed...{},", serviceId,
                    registration.toString(), e);
        }
    }
}
// NacosNamingService 的 registerInstance() 方法会是否支持 gRPC 调用不同的注册服务的实现
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    NamingUtils.checkInstanceIsLegal(instance);
    checkAndStripGroupNamePrefix(instance, groupName);
    clientProxy.registerService(serviceName, groupName, instance);
}

// com.alibaba.nacos.client.naming.remote.gprc.NamingGrpcClientProxy#registerService
@Override
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {
    NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance {}", namespaceId, serviceName,
            instance);
    if (instance.isEphemeral()) {
        registerServiceForEphemeral(serviceName, groupName, instance);
    } else {
        // 注册服务
        doRegisterServiceForPersistent(serviceName, groupName, instance);
    }
}
public void doRegisterServiceForPersistent(String serviceName, String groupName, Instance instance) throws NacosException {
    PersistentInstanceRequest request = new PersistentInstanceRequest(namespaceId, serviceName, groupName,
            NamingRemoteConstants.REGISTER_INSTANCE, instance);
    // 使用 rpc client 向服务中心注册服务
    requestToServer(request, Response.class);
}
private <T extends Response> T requestToServer(AbstractNamingRequest request, Class<T> responseClass)
            throws NacosException {
    Response response = null;
    try {
        request.putAllHeader(
                getSecurityHeaders(request.getNamespace(), request.getGroupName(), request.getServiceName()));
        response = requestTimeout < 0 ? rpcClient.request(request) : rpcClient.request(request, requestTimeout);
        if (ResponseCode.SUCCESS.getCode() != response.getResultCode()) {
            throw new NacosException(response.getErrorCode(), response.getMessage());
        }
        if (responseClass.isAssignableFrom(response.getClass())) {
            return (T) response;
        }
        NAMING_LOGGER.error("Server return unexpected response '{}', expected response should be '{}'",
                response.getClass().getName(), responseClass.getName());
        throw new NacosException(NacosException.SERVER_ERROR, "Server return invalid response");
    } catch (NacosException e) {
        recordRequestFailedMetrics(request, e, response);
        throw e;
    } catch (Exception e) {
        recordRequestFailedMetrics(request, e, response);
        throw new NacosException(NacosException.SERVER_ERROR, "Request nacos server failed: ", e);
    }
}
```

##### 服务发现

nacos 实现了 SpringCloud 的 DiscoveryClient 接口，NacosDiscoveryClient

```java
@Override
public List<ServiceInstance> getInstances(String serviceId) {
    try {
        // 通过 com.alibaba.cloud.nacos.discovery.NacosServiceDiscovery#getInstances 方法获取服务
        return Optional.of(serviceDiscovery.getInstances(serviceId))
                .map(instances -> {
                    ServiceCache.setInstances(serviceId, instances);
                    return instances;
                }).get();
    }
    catch (Exception e) {
        if (failureToleranceEnabled) {
            return ServiceCache.getInstances(serviceId);
        }
        throw new RuntimeException(
                "Can not get hosts from nacos server. serviceId: " + serviceId, e);
    }
}
// com.alibaba.cloud.nacos.discovery.NacosServiceDiscovery#getInstances
public List<ServiceInstance> getInstances(String serviceId) throws NacosException {
    String group = discoveryProperties.getGroup();
    // 调用 NacosNamingService#selectInstance 方法获取服务
    List<Instance> instances = namingService().selectInstances(serviceId, group,
            true);
    return hostToServiceInstanceList(instances, serviceId);
}
// com.alibaba.nacos.client.naming.NacosNamingService#selectInstances(java.lang.String, java.lang.String, java.util.List<java.lang.String>, boolean, boolean)
@Override
public List<Instance> selectInstances(String serviceName, String groupName, List<String> clusters, boolean healthy,
        boolean subscribe) throws NacosException {

    ServiceInfo serviceInfo;
    String clusterString = StringUtils.join(clusters, ",");
    if (subscribe) {// 订阅逻辑
        serviceInfo = serviceInfoHolder.getServiceInfo(serviceName, groupName, clusterString);
        if (null == serviceInfo || !clientProxy.isSubscribed(serviceName, groupName, clusterString)) {
            serviceInfo = clientProxy.subscribe(serviceName, groupName, clusterString);
        }
    } else {
        // 调用不同 client 实现向注册中心请求服务信息
        serviceInfo = clientProxy.queryInstancesOfService(serviceName, groupName, clusterString, false);
    }
    return selectInstances(serviceInfo, healthy);
}
// com.alibaba.nacos.client.naming.remote.gprc.NamingGrpcClientProxy#queryInstancesOfService
@Override
public ServiceInfo queryInstancesOfService(String serviceName, String groupName, String clusters,
        boolean healthyOnly) throws NacosException {
    ServiceQueryRequest request = new ServiceQueryRequest(namespaceId, serviceName, groupName);
    request.setCluster(clusters);
    request.setHealthyOnly(healthyOnly);
    QueryServiceResponse response = requestToServer(request, QueryServiceResponse.class);
    return response.getServiceInfo();
}
```

