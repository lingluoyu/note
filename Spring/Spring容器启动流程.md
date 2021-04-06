## 容器初始化

```mermaid
graph LR;
	创建Spring容器["new AnnotationConfigApplicationContext()</br>创建Spring容器"]-->调用构造方法;
	调用构造方法-->this["this()</br>调用无参构造方法"];
	调用构造方法-->register["register(componentClasses)</br>注入传入的配置类（可传入多个）"];
	调用构造方法-->refresh["refresh()</br>容器刷新"];
	this-->AnnotatedBeanDefinitionReader["AnnotatedBeanDefinitionReader</br>实例化BeanDefinitionReader读取器"];
	AnnotatedBeanDefinitionReader-->registerAnnotationConfigProcessors["registerAnnotationConfigProcessors()</br>代码运行到这里时候，</br>Spring 容器已经构造完毕，</br>那么就可以为容器添加一些内置组件了，</br>其中最主要的组件便是</br> ConfigurationClassPostProcessor 和</br> AutowiredAnnotationBeanPostProcessor ，</br>前者是一个 beanFactory 后置处理器，</br>用来完成 bean 的扫描与注入工作，</br>后者是一个 bean 后置处理器，</br>用来完成 @AutoWired 自动注入"];
	this-->ClassPathBeanDefinitionScanner["ClassPathBeanDefinitionScanner</br>实例化BeanDefinition扫包器"];
	register-->doRegisterBean["doRegisterBean</br>将用户传入的 Spring 配置类注册到容器中"];
```

1. 生成 Bean 对象，需要 BeanFactory 工厂（`DefaultListableBeanFactory`）
2. 将加了特定注解的类（`@Controller`、`@Service`）读取转化为 BeanDefinition 对象（`BeanDefinition` 是 Spring 中极其重要的一个概念，它存储了 bean 对象的所有特征信息，如是否单例，是否懒加载，factoryBeanName 等），需要一个注解配置读取器（`AnnotatedBeanDefinitionReader`）
3. 对用户指定的包目录进行扫描查找 Bean 对象，需要一个路径扫描器（`ClassPathBeanDefinitionScanner`）



## 刷新

```mermaid
graph LR;
	AbstractApplicationContext#refresh["AbstractApplicationContext#refresh</br>容器刷新"]-->prepareRefresh["prepareRefresh()</br>刷新前预处理"];
	AbstractApplicationContext#refresh-->obtainFreshBeanFactory["obtainFreshBeanFactory()</br>获取BeanFactory，在前面的步骤已经初始化"];
	obtainFreshBeanFactory-->refreshBeanFactory["refreshBeanFactory()</br>刷新BeanFactory，设置序列化id"];
	obtainFreshBeanFactory-->getBeanFactory["getBeanFactory()</br>返回之前初始化过程中创建的BeanFactory，</br>即DefaultListableBeanFactory"];
	AbstractApplicationContext#refresh-->prepareBeanFactory["prepareBeanFactory(beanFactory)</br>预处理BeanFactory，向容器中添加一些组件"]-->组件["Bean后置处理器：</br>ApplicationContextAwareProcessor、ApplicationListenerDetector</br>添加可解析的自动装配器，即在任何组件中都可以自动注入：</br>BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext</br>注册environment Bean：</br>environment->ConfigurableEnvironment systemProperties->Map< String, Object></br> systemEnvironment->Map< String, Object>"];
	AbstractApplicationContext#refresh-->postProcessBeanFactory["postProcessBeanFactory(beanFactory)</br>允许在上下文子类中重写此方法在</br>BeanFactory初始化及创建完成后进行进一步处理"];
	AbstractApplicationContext#refresh-->invokeBeanFactoryPostProcessors["invokeBeanFactoryPostProcessors(beanFactory)</br>执行BeanFactoryPostProcessor方法"];
	invokeBeanFactoryPostProcessors-->PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors["invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors())</br>此处的getBeanFactoryPostProcessors()获取的是通过</br>AbstractApplicationContext.addBeanFactoryPostProcessor</br>(BeanFactoryPostProcessor postProcessor)传入的，</br>不是Spring管理的BeanFactoryPostProcessor"]-->invokeBeanFactoryPostProcessors2["invokeBeanFactoryPostProcessors(postProcessors, beanFactory)</br>执行BeanFactoryPostProcessor"];
	AbstractApplicationContext#refresh-->registerBeanPostProcessors["registerBeanPostProcessors(beanFactory)</br>注册拦截Bean创建的Bean处理器"]-->registerBeanPostProcessors2["registerBeanPostProcessors(beanFactory, beanPostProcessors)</br>按顺序注册后置处理器</br>先注册实现了PriorityOrdered接口的BeanPostProcessor</br>再注册实现了Ordered接口的BeanPostProcessor</br>然后注册常规的BeanPostProcessor</br>最后注册MergedBeanDefinitionPostProcessor类型的BeanPostProcessor"];
	AbstractApplicationContext#refresh-->initMessageSource["initMessageSource()</br>初始化MessageSource：国际化，消息绑定与解析"];
	AbstractApplicationContext#refresh-->initApplicationEventMulticaster["initApplicationEventMulticaster()</br>初始化事件派发器，在注册监听器时会用到"]-->detail["判断容器中是否有名为applicationEventMulticaster的组件，</br>若有，就取出并赋值给applicationEventMulticaster，</br>若没有，就创建一个SimpleApplicationEventMulticaster事件派发器"];
	AbstractApplicationContext#refresh-->onRefresh["onRefresh()</br>在特定上下文子类中初始化其他特殊bean"];
	AbstractApplicationContext#refresh-->registerListeners["registerListeners()</br>注册监听器，派发之前步骤产生的事件"]-->detail2["将之前步骤产生的ApplicationListener及容器中的ApplicationListener保存到</br>applicationEventMulticaster中，然后进行事件派发"];
	AbstractApplicationContext#refresh-->finishBeanFactoryInitialization["finishBeanFactoryInitialization(beanFactory)</br>初始化所有的单实例bean"]-->preInstantiateSingletons["preInstantiateSingletons()</br>初始化所有剩余的单实例bean</br>遍历所有beanNames，循环调用getBean(beanName)创建bean</br>若bean是SmartInitializingSingleton类型，则调用afterSingletonsInstantiated()方法"];
	AbstractApplicationContext#refresh-->finishRefresh["finishRefresh()</br>发布容器刷新完成事件"]-->initLifecycleProcessor["initLifecycleProcessor() 初始化和生命周期有关的BeanPostProcessor"];
	finishRefresh-->getLifecycleProcessor["getLifecycleProcessor().onRefresh() 拿到前面定义的生命周期处理器</br>LifecycleProcessor回调onRefresh方法"];
	finishRefresh-->publishEvent["publishEvent(new ContextRefreshedEvent(this)) 发布容器刷新完成事件"];
```

