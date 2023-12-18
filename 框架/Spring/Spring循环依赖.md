### 循环依赖三种情况
* 通过构造方法进行依赖注入时产生的循环依赖问题。
* 通过setter方法进行依赖注入且是在多例（原型）模式下产生的循环依赖问题。
* 通过setter方法进行依赖注入且是在单例模式下产生的循环依赖问题。

注意：在Spring中，只有【第三种方式】的循环依赖问题被解决了，其他两种方式在遇到循环依赖问题时都会产生异常。

* 第一种构造方法注入的情况下，在new对象的时候就会堵塞住了，其实也就是”先有鸡还是先有蛋“的历史难题。
* 第二种setter方法&&多例的情况下，每一次getBean()时，都会产生一个新的Bean，如此反复下去就会有无穷无尽的Bean产生了，最终就会导致OOM问题的出现。

### 解决循环依赖
#### Spring三层缓存
Spring中有三个缓存，用于存储单例的Bean实例，这三个缓存是彼此互斥的，不会针对同一个Bean的实例同时存储。

如果调用getBean，则需要从三个缓存中依次获取指定的Bean实例。 读取顺序依次是一级缓存-->二级缓存-->三级缓存

##### 一级缓存：Map<String, Object> singletonObjects

第一级缓存的作用？
* 用于存储单例模式下创建的Bean实例（已经创建完毕）。
* 该缓存是对外使用的，指的就是使用Spring框架的程序员。

存储什么数据？

* K：bean的名称
* V：bean的实例对象（有代理对象则指的是代理对象，已经创建完毕）
##### 第二级缓存：Map<String, Object> earlySingletonObjects
第二级缓存的作用？

* 用于存储单例模式下创建的Bean实例（该Bean被提前暴露的引用,该Bean还在创建中）。
* 该缓存是对内使用的，指的就是Spring框架内部逻辑使用该缓存。
* 为了解决第一个classA引用最终如何替换为代理对象的问题（如果有代理对象）

存储什么数据？
* K：bean的名称
* V：bean的实例对象（有代理对象则指的是代理对象，该Bean还在创建中）

##### 第三级缓存：Map<String, ObjectFactory<?>> singletonFactories
第三级缓存的作用？
* 通过ObjectFactory对象来存储单例模式下提前暴露的Bean实例的引用（正在创建中）。
* 该缓存是对内使用的，指的就是Spring框架内部逻辑使用该缓存。
* 此缓存是解决循环依赖最大的功臣

存储什么数据？
* K：bean的名称
* V：ObjectFactory，该对象持有提前暴露的bean的引用

为什么第三级缓存要使用ObjectFactory？

需要提前产生代理对象。

什么时候将Bean的引用提前暴露给第三级缓存的ObjectFactory持有？

时机就是在第一步实例化之后，第二步依赖注入之前，完成此操作。

### 源码

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

