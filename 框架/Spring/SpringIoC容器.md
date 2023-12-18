### 容器

#### ApplicationContext：Spring容器上下文

**BeanFactory**：Spring Bean容器的根接口

**ApplicationContext**：Spring IoC容器，负责实例化，配置和组装Bean。

- 用于访问应用程序组件的Bean工厂方法。继承自[`ListableBeanFactory`](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/beans/factory/ListableBeanFactory.html)。
- 以通用方式加载文件资源的能力。从[`ResourceLoader`](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/core/io/ResourceLoader.html)接口继承。
- 将事件发布给注册的侦听器的能力。从[`ApplicationEventPublisher`](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/context/ApplicationEventPublisher.html)接口继承。
- 解决消息的能力，支持国际化。从[`MessageSource`](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/context/MessageSource.html)接口继承。
- 从父上下文继承。在后代上下文中的定义将始终优先。

通过[`ClassPathXmlApplicationContext`](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/context/support/ClassPathXmlApplicationContext.html) 或[`FileSystemXmlApplicationContext`](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/context/support/FileSystemXmlApplicationContext.html)实例化容器

```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:config/Bean.xml");
```

通过`T getBean(String name, Class<T> requiredType)`方法获取Bean实例

```java
//根据配置文件创建Bean
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:config/Bean.xml");
//获取Bean实例
TestBean testBean = applicationContext.getBean("testBean", TestBean.class);
//使用Bean
System.out.println(testBean.toString());
```

**BeanDefinition**：Spring容器使用配置元数据创建Bean（XML`<bean/>`，注解@Bean等方式），这些Bean定义为BeanDefinition对象

#### Spring创建Bean流程

```java
//调用ClassPathXmlApplicationContext构造器创建Spring容器
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
	this(new String[] {configLocation}, true, null);
}

//初始化方法
public ClassPathXmlApplicationContext(
    String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
    throws BeansException {
	//调用父类构造方法创建容器上下文
    super(parent);
    //配置文件路径
    setConfigLocations(configLocations);
    if (refresh) {
        //调用AbstractApplicationContext.refresh()方法
        refresh();
    }
}

//Spring容器构造核心类
//org.springframework.context.support.AbstractApplicationContext#refresh
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        //刷新前设置一些参数
        // Prepare this context for refreshing.
        prepareRefresh();

        //此步结束后，配置文件就会被解析成BeanDefinition，注册到BeanFactory中
        //此时Bean还未被初始化，只是将配置信息提取出来
        //将这些信息都保存到注册中心(核心是一个 beanName-> beanDefinition 的 map)
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        //见1.2
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            //调用1.2prepareBeanFactory(beanFactory)中注册的BeanFactoryProcessor
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            //MessageSource事件监听器，初始化国际化信息资源
            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

`AbstractApplicationContext.obtainFreshBeanFactory`方法，刷新BeanFactory，在这一步中，也会将BeanDefinition加载到BeanFactory中。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}

@Override
protected final void refreshBeanFactory() throws BeansException {
    //如果BeanFactory已存在，就销毁
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        //创建DefaultListableBeanFactory
        //此类是整个Bean加载的核心部分，是Spring注册加载Bean的默认实现，是整个Spring IOC的始祖。
        //对beanFactory进行设置,bean注册等操作,最后将beanFactory赋值给本类的beanFactory属性
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
		//设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
        customizeBeanFactory(beanFactory);
        //加载BeanDefinitions到BeanFactory中
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}

//org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    //创建XmlBeanDefinitionReader，用于读取BeanDefinition
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // 设置环境属性以及资源加载器ResourceLoader为this（实际为 ClassPathXmlApplicationContext）
    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    //初始化读取器，提供给子类重写
    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    //加载BeanDefinition，从配置文件中读取BeanDefiniton
    loadBeanDefinitions(beanDefinitionReader);
}

//org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    //加载配置资源
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    //加载ClassPathXmlApplicationContext初始化方法中设置的xml配置文件
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        //循环加载xml文件的Bean返回Bean总个数
        reader.loadBeanDefinitions(configLocations);
    }
}

//org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    ResourceLoader resourceLoader = getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException(
            "Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
    }

    if (resourceLoader instanceof ResourcePatternResolver) {
        // Resource pattern matching available.
        try {
            //获取加载器中的资源
            Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
            //从resources循环中加载BeanDefinition，返回Bean总数
            int count = loadBeanDefinitions(resources);
            if (actualResources != null) {
                Collections.addAll(actualResources, resources);
            }
            if (logger.isTraceEnabled()) {
                logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
            }
            return count;
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                "Could not resolve bean definition resource pattern [" + location + "]", ex);
        }
    }
    else {
        // Can only load single resources by absolute URL.
        Resource resource = resourceLoader.getResource(location);
        //通过绝对路径读取单个resources中BeanDefinition
        int count = loadBeanDefinitions(resource);
        if (actualResources != null) {
            actualResources.add(resource);
        }
        if (logger.isTraceEnabled()) {
            logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
        }
        return count;
    }
}

//org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isTraceEnabled()) {
        logger.trace("Loading XML bean definitions from " + encodedResource);
    }

    //private final ThreadLocal<Set<EncodedResource>> resourcesCurrentlyBeingLoaded = new NamedThreadLocal<>("XML bean definition resources currently being loaded");
    //将当前正在读取的xml resources放入ThreadLocal，保证只有本次线程可访问
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            //解析Document, 并在解析之后进行注册
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        currentResources.remove(encodedResource);
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}

//org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition
//把beanName和beanDefinition对象作为键值对放到BeanFactory对象的beanDefinitionMap
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                                                   "Validation of bean definition failed", ex);
        }
    }

    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }
        else if (existingDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            if (logger.isInfoEnabled()) {
                logger.info("Overriding user-defined bean definition for bean '" + beanName +
                            "' with a framework-generated bean definition: replacing [" +
                            existingDefinition + "] with [" + beanDefinition + "]");
            }
        }
        else if (!beanDefinition.equals(existingDefinition)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Overriding bean definition for bean '" + beanName +
                             "' with a different definition: replacing [" + existingDefinition +
                             "] with [" + beanDefinition + "]");
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Overriding bean definition for bean '" + beanName +
                             "' with an equivalent definition: replacing [" + existingDefinition +
                             "] with [" + beanDefinition + "]");
            }
        }
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        if (hasBeanCreationStarted()) {
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                removeManualSingletonName(beanName);
            }
        }
        else {
            // Still in startup registration phase
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            removeManualSingletonName(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

`AbstractApplicationContext#prepareBeanFactory`方法，为BeanFactory添加一些内置组件，预处理BeanFactory。

```java
//准备BeanFactory，设置一些参数，如BeanPostProcesser
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //设置类加载器
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    //设置表达式解析器，用来解析BeanDefiniton中的带有表达式的值，spring3增加了表达式语言的支持，默认可以使用#{bean.xxx}的形式来调用相关属性值。
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    //增加了一个默认的propertyEditor，这个主要是对bean的属性等设置管理的一个工具
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    //配置后置处理器，主要的作用就是在spring实例化bean的前后做一些操作。
    //添加了一个处理aware相关接口的beanPostProcessor扩展，实现了 Aware 接口的 beans 在初始化的时候，这个 processor 负责回调，主要是使用beanPostProcessor的postProcessBeforeInitialization()前置处理方法实现aware相关接口的功能，aware接口是用来给bean注入一些资源的接口，回调 ApplicationContextAware、EnvironmentAware、ResourceLoaderAware。
    // Configure the bean factory with context callbacks.
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //设置忽略自动装配的接口类，这些类及其实现类都不能使用注解（@Resource或者@Autowired）方式注入，Spring会通过其他方式注入这些类及其实现类。
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    //注册可解析的自动装配类的特殊规则，如果是BeanFactory类型，则注入beanFactory对象，如果是ResourceLoader、ApplicationEventPublisher、ApplicationContext类型则注入当前对象（applicationContext对象）。
    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    //配置后置处理器，注册监听器，监听自定义ApplicationEvent（创建一个实现ApplicationListener并注册为Spring bean的类）
    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    //添加编译时的 AspectJ
    //如果定义了则添加loadTimeWeaver功能的beanPostProcessor扩展，并且创建一个临时的classLoader来让其处理真正的bean。spring的loadTimeWeaver主要是通过 instrumentation 的动态字节码增强在装载期注入依赖。tips: ltw 是 AspectJ 的概念，指的是在运行期进行织入，这个和 Spring AOP 不一样，
    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    //注册 environment 组件，类型是【ConfigurableEnvironment】
    // Register default environment beans.
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    
    //注册 systemProperties 组件，类型是【Map<String, Object>】
    //判断是否定义了名为systemProperties的bean，如果没有则加载系统获取当前系统属性System.getProperties()并注册为一个单例bean。假如有AccessControlException权限异常则创建一个ReadOnlySystemAttributesMap对象，可以看到创建时重写了getSystemAttribute()方法，查看ReadOnlySystemAttributesMap的代码可以得知在调用get方法的时候会去调用这个方法来获取key对应的对象，当获取依旧有权限异常AccessControlException的时候则返回null。
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    //注册 systemEnvironment 组件，类型是【Map<String, Object>】
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

#### 获取Bean

获取Bean调用的是`ApplicationContext`的子类`AbstractBeanFactory`的`getBean()`方法

```java
@Override
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
    return doGetBean(name, requiredType, null, false);
}

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    //转换Bean别名
    //平时开发中传入的参数name可能只是别名，也可能是FactoryBean，所以需要进行解析转换：
    //（1）消除修饰符，比如name="&test"，会去除&使name="test"；
    //（2）解决spring中alias标签的别名问题
    final String beanName = transformedBeanName(name);
    Object bean;

    //尝试从缓存（DefaultSingletonBeanRegistry.singletonObjects）中加载Bean实例
    // Eagerly check singleton cache for manually registered singletons.
    Object sharedInstance = getSingleton(beanName);
    //缓存中存在对应Bean
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                             "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        //从缓存中拿到的bean的实例化。从缓存中获取的bean是原始状态的bean,需要在这里对bean进行bean实例化。
        //此时会进行 合并RootBeanDefinition、BeanPostProcessor进行实例前置处理、实例化、实例后置处理。
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {//如果缓存中没有对应bean
        //循环依赖检查。 (构造器的循环依赖)循环依赖存在，则报错。
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        //如果缓存中没有数据，就会转到父类工厂去加载
        
        //获取父工厂
        // Check if bean definition exists in this factory.
        BeanFactory parentBeanFactory = getParentBeanFactory();
        //!containsBeanDefinition(beanName)就是检测如果当前加载的xml配置文件中不包含beanName所对应的配置，若找不到就只能到parentBeanFacotory去尝试加载bean。
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                    nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        
        try {
            //存储XML配置文件的GernericBeanDefinition转换成RootBeanDefinition，即为合并父类定义。
        	//XML配置文件中读取到的bean信息是存储在GernericBeanDefinition中的，但Bean的后续处理是针对于RootBeanDefinition的，所以需要转换后才能进行后续操作。
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);
			
            //初始化依赖的bean
            // Guarantee initialization of beans that the current bean depends on.
            String[] dependsOn = mbd.getDependsOn();
            //bean中可能依赖了其他bean属性，在初始化bean之前会先初始化这个bean所依赖的bean属性。
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            //创建Bean
            // Create bean instance.
            if (mbd.isSingleton()) {//单例Bean
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        //创建Bean代码
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
			//原型Bean
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }
			//如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                                                    "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                                    "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                                    ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    //检查获取到的Bean类型是否正确
    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                             ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```
`DefaultSingletonBeanRegistry#getSingleton`方法，Spring使用三级缓存解决**循环依赖**问题

```java
//第一级缓存，Map<BeanName, BeanInstance>，用于存放完整的单例Bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
//第二级缓存，Map<BeanName, BeanInstance>，用于存放提前暴露的单例对象（Bean被提前暴露的引用,Bean还在创建中）
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
//第三级缓存，Map<BeanName, ObjectFactory>，用于存储单例模式下提前暴露的Bean实例的引用（正在创建中）。
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
//创建中的单例对象
private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //先从第一级缓存singletonObjects获取Bean
    Object singletonObject = this.singletonObjects.get(beanName);
    //获取不到且是正在创建中的Bean
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            //从二级缓存earlySingletonObjects中获取Bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            //从二级缓存中获取不到，则从三级缓存中获取存入二级缓存，并从三级缓存中移除
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```
`AbstractAutowireCapableBeanFactory#doCreateBean`方法中代码

```java
//判断是否需要提前暴露
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                  isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
                     "' to allow for resolving potential circular references");
    }
    //解决循环依赖 第二个参数是回调接口，实现的功能是将切面动态织入 bean
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}

//org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        //singletonObjects中是否存在Bean
        if (!this.singletonObjects.containsKey(beanName)) {
            //放入 beanName -> beanFactory，到时在 getSingleton() 获取单例时，可直接获取创建对应 bean 的工厂，解决循环依赖
            this.singletonFactories.put(beanName, singletonFactory);
            //从提前曝光的缓存中移除之前在 getSingleton() 放入的Bean
            this.earlySingletonObjects.remove(beanName);
            //向注册缓存中添加Bean
            this.registeredSingletons.add(beanName);
        }
    }
}
```

继续创建Bean的代码

```java
//org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {

    if (logger.isTraceEnabled()) {
        logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    //确保 BeanDefinition 中的 Class 被加载
    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                                               beanName, "Validation of method overrides failed", ex);
    }

    try {
        //让 InstantiationAwareBeanPostProcessor 在这一步有机会返回代理
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                                        "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
        //创建 bean
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // A previously detected exception with proper bean creation context already,
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}

//org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        //说明不是 FactoryBean，这里实例化 Bean
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    //这个就是 Bean 里面的 我们定义的类 的实例
    final Object bean = instanceWrapper.getWrappedInstance();
    //类型
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    //解决循环依赖
    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                         "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        //属性装配
        populateBean(beanName, mbd, instanceWrapper);
        //处理 bean 初始化完成后的各种回调，init-method  InitializingBean 接口  BeanPostProcessor 接口
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                                                               "Bean with name '" + beanName + "' has been injected into other beans [" +
                                                               StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                                               "] in its raw version as part of a circular reference, but has eventually been " +
                                                               "wrapped. This means that said other beans do not use the final version of the " +
                                                               "bean. This is often the result of over-eager type matching - consider using " +
                                                               "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

