### 配置

- Spring 1.2 **基于接口配置**：最早的 Spring AOP 是完全基于几个接口
- Spring 2.0 **schema-based配置**：Spring 2.0 以后使用 XML 的方式来配置，使用 命名空间 `<aop/>`
- Spring 2.0 **@AspectJ配置**：使用注解的方式来配置；这里虽然叫做 `@AspectJ`，但是这个和 AspectJ 其实没啥关系。

#### Spring 1.2 配置

简单示例

定义接口与实现

```java
public interface UserService {
    User createUser(String firstName, String lastName, int age);

    User queryUser();
}
public class UserServiceImpl implements UserService {
    @Override
    public User createUser(String firstName, String lastName, int age) {
        User user = new User();
        user.setFirstName(firstName);
        user.setLastName(lastName);
        user.setAge(age);
        return user;
    }

    @Override
    public User queryUser() {
        User user = new User();
        user.setFirstName("test");
        user.setLastName("test");
        user.setAge(20);
        return user;
    }
}
public interface OrderService {
    Order createOrder(String username, String product);

    Order queryOrder(String username);
}
public class OrderServiceImpl implements OrderService {
    @Override
    public Order createOrder(String username, String product) {
        Order order = new Order();
        order.setUsername(username);
        order.setProduct(product);
        return order;
    }

    @Override
    public Order queryOrder(String username) {
        Order order = new Order();
        order.setUsername("test");
        order.setProduct("test");
        return order;
    }
}
```

定义 **advice**，用于拦截**方法执行前**和**方法返回后**

```java
public class LogArgsAdvice implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("[advice]准备执行方法: " + method.getName() + ", 参数列表：" + Arrays.toString(args));
    }
}
public class LogResultAdvice implements AfterReturningAdvice {
    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target)
            throws Throwable {
        System.out.println("[advice]方法返回：" + returnValue);
    }
}
```

xml配置

```xml
<bean id="userServiceImpl" class="com.javadoop.springaoplearning.service.imple.UserServiceImpl"/>
<bean id="orderServiceImpl" class="com.javadoop.springaoplearning.service.imple.OrderServiceImpl"/>

<!--定义两个 advice-->
<bean id="logArgsAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogArgsAdvice"/>
<bean id="logResultAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogResultAdvice"/>

<bean id="userServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
    <!--代理的接口-->
    <property name="proxyInterfaces">
        <list>
            <value>com.javadoop.springaoplearning.service.UserService</value>
        </list>
    </property>
    <!--代理的具体实现-->
    <property name="target" ref="userServiceImpl"/>

    <!--配置拦截器，这里可以配置 advice、advisor、interceptor, 这里先介绍 advice-->
    <property name="interceptorNames">
        <list>
            <value>logArgsAdvice</value>
            <value>logResultAdvice</value>
        </list>
    </property>
</bean>

<!--===========================================-->
<!--同理，我们也可以配置一个 orderServiceProxy......-->
<!--===========================================-->
```

运行

```java
public static void test_Spring_1_2_Advice() {

    // 启动 Spring 的 IOC 容器
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring_1_2_advice.xml");

    // 我们这里要取 AOP 代理：userServiceProxy，这非常重要
    UserService userService = (UserService) context.getBean("userServiceProxy");

    userService.createUser("Tom", "Cruise", 55);
    userService.queryUser();
}
```

输出结果

```shell
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: queryUser, 参数列表：[]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
```

此示例就是一个简单的代理模式

> 代理模式需要一个接口、一个具体实现类，然后就是定义一个代理类，用来包装实现类，添加自定义逻辑，在使用的时候，需要用代理类来生成实例。

这种方法若需要拦截OrderService中的方法，还需要定义一个OrderService的代理......

且拦截器粒度只控制到了类级别，类中所有方法都进行了拦截。

下面看一下如何**只拦截特定的方法**

**Advisor**：**内部需要指定一个 Advice**，Advisor 决定该拦截哪些方法，拦截后需要完成的工作还是内部的 Advice 来做。

Advisor有几个实现类，这里用 **NameMatchMethodPointcutAdvisor** 演示，它需要我们给它提供方法名字，这样符合该配置的方法才会做拦截。

```xml
<bean id="userServiceImpl" class="com.javadoop.springaoplearning.service.imple.UserServiceImpl"/>
<bean id="orderServiceImpl" class="com.javadoop.springaoplearning.service.imple.OrderServiceImpl"/>

<!--定义两个 advice-->
<bean id="logArgsAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogArgsAdvice"/>
<bean id="logResultAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogResultAdvice"/>

<!--定义一个只拦截queryUser方法的 advisor-->
<bean id="logCreateAdvisor" class="org.springframework.aop.support.NameMatchMethodPointcutAdvisor">
    <!--advisor 实例的内部会有一个 advice-->
    <property name="advice" ref="logArgsAdvice" />
    <!--只有下面这两个方法才会被拦截-->
    <property name="mappedNames" value="createUser,createOrder" />
</bean>

<bean id="userServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
    <!--代理的接口-->
    <property name="proxyInterfaces">
        <list>
            <value>com.javadoop.springaoplearning.service.UserService</value>
        </list>
    </property>
    <!--代理的具体实现-->
    <property name="target" ref="userServiceImpl"/>

    <!--配置拦截器，这里可以配置 advice、advisor、interceptor-->
    <property name="interceptorNames">
        <list>
            <value>logCreateAdvisor</value>
        </list>
    </property>
</bean>

<!--===========================================-->
<!--同理，我们也可以配置一个 orderServiceProxy......-->
<!--===========================================-->
```

> 可以看到，userServiceProxy 这个 bean 配置了一个 advisor，advisor 内部有一个 advice。advisor 负责匹配方法，内部的 advice 负责实现方法包装。
>
> 注意，这里的 mappedNames 配置是可以指定多个的，用逗号分隔，可以是不同类中的方法。相比直接指定 advice，advisor 实现了更细粒度的控制，因为在这里配置 advice 的话，所有方法都会被拦截。

运行代码

```java
public static void test_Spring_1_2_Advisor() {
    // 启动 Spring 的 IOC 容器
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring_1_2_advisor.xml");

    // 我们这里要取 AOP 代理：userServiceProxy，这非常重要
    UserService userService = (UserService) context.getBean("userServiceProxy");

    userService.createUser("Tom", "Cruise", 55);
    userService.queryUser();
}
```

输出结果，只有createUser方法被拦截

```shell
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
```

它们有个共同的问题，那就是我们得为每个 bean 都配置一个代理，之后获取 bean 的时候需要获取这个代理类的 bean 实例（如 `(UserService) context.getBean("userServiceProxy")`），这显然非常不方便，不利于我们之后要使用的自动根据类型注入。下面介绍 autoproxy 的解决方案。

**autoproxy**：从名字我们也可以看出来，它是实现自动代理，也就是说当 Spring 发现一个 bean 需要被切面织入的时候，Spring 会自动生成这个 bean 的一个代理来拦截方法的执行，确保定义的切面能被执行。

这里强调**自动**，也就是说 Spring 会自动做这件事，而不用像前面介绍的，我们需要显式地指定代理类的 bean。

xml配置

```xml
<bean id="userServiceImpl" class="com.javadoop.springaoplearning.service.imple.UserServiceImpl"/>
<bean id="orderServiceImpl" class="com.javadoop.springaoplearning.service.imple.OrderServiceImpl"/>

<!--定义两个 advice-->
<bean id="logArgsAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogArgsAdvice"/>
<bean id="logResultAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogResultAdvice"/>

<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
    <property name="interceptorNames">
        <list>
            <value>logArgsAdvice</value>
            <value>logResultAdvice</value>
        </list>
    </property>
    <property name="beanNames" value="*ServiceImpl" />
</bean>
```

beanNames 中可以使用正则来匹配 bean 的名字。这样配置出来以后，userServiceBeforeAdvice 和 userServiceAfterAdvice 这两个拦截器就不仅仅可以作用于 UserServiceImpl 了，也可以作用于 OrderServiceImpl、PostServiceImpl、ArticleServiceImpl......等等

运行

```java
 public static void test_Spring_1_2_BeanNameAutoProxy() {
     // 启动 Spring 的 IOC 容器
     ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring_1_2_BeanNameAutoProxy.xml");

     // 注意这里，不再需要根据代理找 bean
     UserService userService = context.getBean(UserService.class);
     OrderService orderService = context.getBean(OrderService.class);

     userService.createUser("Tom", "Cruise", 55);
     userService.queryUser();

     orderService.createOrder("Leo", "随便买点什么");
     orderService.queryOrder("Leo");
 }
```

输出结果， OrderService 和 UserService 中的每个方法都得到了拦截

```shell
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: queryUser, 参数列表：[]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: createOrder, 参数列表：[Leo, 随便买点什么]
方法返回：Order{username='Leo', product='随便买点什么'}
准备执行方法: queryOrder, 参数列表：[Leo]
方法返回：Order{username='Leo', product='随便买点什么'}
```

 BeanNameAutoProxyCreator 非常好用，它需要指定被拦截类名的模式(如 *ServiceImpl)，它可以配置多次，这样就可以用来匹配不同模式的类了。

在 BeanNameAutoProxyCreator 同一个包中，还有一个非常有用的类 **DefaultAdvisorAutoProxyCreator**，比上面的 BeanNameAutoProxyCreator 还要方便。

advisor 内部包装了 advice，advisor 负责决定拦截哪些方法，内部 advice 定义拦截后的逻辑。所以，仔细想想其实就是只要让我们的 advisor 全局生效就能实现我们需要的自定义拦截功能、拦截后的逻辑处理。

> BeanNameAutoProxyCreator 是自己匹配方法，然后交由内部配置 advice 来拦截处理；
>
> 而 DefaultAdvisorAutoProxyCreator 是让 ioc 容器中的所有 advisor 来匹配方法，advisor 内部都是有 advice 的，让它们内部的 advice 来执行拦截处理。

Advisor 还有一个更加灵活的实现类 **RegexpMethodPointcutAdvisor**，它能实现正则匹配

```xml
<bean id="userServiceImpl" class="com.javadoop.springaoplearning.service.imple.UserServiceImpl"/>
<bean id="orderServiceImpl" class="com.javadoop.springaoplearning.service.imple.OrderServiceImpl"/>

<!--定义两个 advice-->
<bean id="logArgsAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogArgsAdvice"/>
<bean id="logResultAdvice" class="com.javadoop.springaoplearning.aop_spring_1_2.LogResultAdvice"/>

<!--定义两个 advisor-->
<!--记录 create* 方法的传参-->
<bean id="logArgsAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
    <property name="advice" ref="logArgsAdvice" />
    <property name="pattern" value="com.javadoop.springaoplearning.service.*.create.*" />
</bean>
<!--记录 query* 的返回值-->
<bean id="logResultAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
    <property name="advice" ref="logResultAdvice" />
    <property name="pattern" value="com.javadoop.springaoplearning.service.*.query.*" />
</bean>


<!--定义DefaultAdvisorAutoProxyCreator-->
<!--配置 DefaultAdvisorAutoProxyCreator，它的配置非常简单，直接使用下面这段配置就可以了，它就会使得所有的 Advisor 自动生效，无须其他配置。-->
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />
```

运行

```java
public static void test_Spring_1_2_DefaultAdvisorAutoProxy() {
    // 启动 Spring 的 IOC 容器
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring_1_2_DefaultAdvisorAutoProxy.xml");

    UserService userService = context.getBean(UserService.class);
    OrderService orderService = context.getBean(OrderService.class);

    userService.createUser("Tom", "Cruise", 55);
    userService.queryUser();

    orderService.createOrder("Leo", "随便买点什么");
    orderService.queryOrder("Leo");
}
```

结果

```shell
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: createOrder, 参数列表：[Leo, 随便买点什么]
方法返回：Order{username='Leo', product='随便买点什么'}
```

create *方法使用了 logArgsAdvisor 进行传参输出，query* 方法使用了 logResultAdvisor 进行了返回结果输出。

 Advisor 不只有 NameMatchMethodPointcutAdvisor 和 RegexpMethodPointcutAdvisor，AutoProxyCreator 也不仅仅是 BeanNameAutoProxyCreator 和 DefaultAdvisorAutoProxyCreator。

#### Spring 2.0 @AspectJ 配置

Spring 2.0 以后，引入了 @AspectJ 和 Schema-based 的两种配置方式，我们先来介绍 @AspectJ 的配置方式，之后我们再来看使用 xml 的配置方式。

注意了，**@AspectJ 和 AspectJ 没多大关系**，并不是说基于 AspectJ 实现的，而仅仅是使用了 AspectJ 中的概念，包括使用的注解也是直接来自于 AspectJ 的包。

**开启 @AspectJ 的注解配置方式**

1. 在 xml 中配置：

```xml
<aop:aspectj-autoproxy/>
```

2. 使用 @EnableAspectJAutoProxy

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```

一旦开启了上面的配置，那么所有使用 @Aspect 注解的 **bean** 都会被 Spring 当做**用来实现 AOP 的配置类**，我们称之为一个 **Aspect**。

@Aspect 注解的 bean 中，我们需要配置哪些内容。

**首先，我们需要配置 Pointcut，**Pointcut 在大部分地方被翻译成切点，用于定义哪些方法需要被增强或者说需要被拦截，有点类似于之前介绍的 **Advisor** 的方法匹配。

Spring AOP 只支持 bean 中的方法（不像 AspectJ 那么强大），所以我们可以认为 **Pointcut** 就是用来匹配 Spring 容器中的所有 bean 的方法的。

```java
@Pointcut("execution(* transfer(..))")// the pointcut expression
private void anyOldTransfer() {}// the pointcut signature
```

@Pointcut 中使用了 **execution** 来正则匹配方法签名，这也是最常用的，除了 execution，我们再看看其他的几个比较常用的匹配方式：

- within：指定所在类或所在包下面的方法（Spring AOP 独有）

  > 如 @Pointcut("within(com.javadoop.springaoplearning.service..*)")

- @annotation：方法上具有特定的注解，如 @Subscribe 用于订阅特定的事件。

  > 如 @Pointcut("execution( .*(..)) && @annotation(com.javadoop.annotation.Subscribe)")

- bean(idOrNameOfBean)：匹配 bean 的名字（Spring AOP 独有）

  > 如 @Pointcut("bean(*Service)")

Tips：上面匹配中，通常 "." 代表一个包名，".." 代表包及其子包，方法参数任意匹配使用两个点 ".."。

对于 web 开发者，Spring 有个很好的建议，就是定义一个 **SystemArchitecture**：

```java
@Aspect
public class SystemArchitecture {

    // web 层
    @Pointcut("within(com.javadoop.web..*)")
    public void inWebLayer() {}

    // service 层
    @Pointcut("within(com.javadoop.service..*)")
    public void inServiceLayer() {}

    // dao 层
    @Pointcut("within(com.javadoop.dao..*)")
    public void inDataAccessLayer() {}

    // service 实现，注意这里指的是方法实现，其实通常也可以使用 bean(*ServiceImpl)
    @Pointcut("execution(* com.javadoop..service.*.*(..))")
    public void businessService() {}

    // dao 实现
    @Pointcut("execution(* com.javadoop.dao.*.*(..))")
    public void dataAccessOperation() {}
}
```

上面这个 SystemArchitecture 很好理解，该 Aspect 定义了一堆的 Pointcut，随后在任何需要 Pointcut 的地方都可以直接引用（如 xml 中的 pointcut-ref=""）。

配置 pointcut 就是配置我们需要拦截哪些方法，接下来，我们要配置需要对这些被拦截的方法做什么，也就是前面介绍的 Advice。

**接下来，我们要配置 Advice。**

```java
@Aspect
public class AdviceExample {

    // 这里会用到我们前面说的 SystemArchitecture
    // 下面方法就是写拦截 "dao层实现"
    @Before("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doAccessCheck() {
        // ... 实现代码
    }

    // 当然，我们也可以直接"内联"Pointcut，直接在这里定义 Pointcut
    // 把 Advice 和 Pointcut 合在一起了，但是这两个概念我们还是要区分清楚的
    @Before("execution(* com.javadoop.dao.*.*(..))")
    public void doAccessCheck() {
        // ... 实现代码
    }

    @AfterReturning("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doAccessCheck() {
        // ...
    }

    @AfterReturning(
        pointcut="com.javadoop.aop.SystemArchitecture.dataAccessOperation()",
        returning="retVal")
    public void doAccessCheck(Object retVal) {
        // 这样，进来这个方法的处理时候，retVal 就是相应方法的返回值，是不是非常方便
        //  ... 实现代码
    }

    // 异常返回
    @AfterThrowing("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doRecoveryActions() {
        // ... 实现代码
    }

    @AfterThrowing(
        pointcut="com.javadoop.aop.SystemArchitecture.dataAccessOperation()",
        throwing="ex")
    public void doRecoveryActions(DataAccessException ex) {
        // ... 实现代码
    }

    // 注意理解它和 @AfterReturning 之间的区别，这里会拦截正常返回和异常的情况
    @After("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doReleaseLock() {
        // 通常就像 finally 块一样使用，用来释放资源。
        // 无论正常返回还是异常退出，都会被拦截到
    }

    // 感觉这个很有用吧，既能做 @Before 的事情，也可以做 @AfterReturning 的事情
    @Around("com.javadoop.aop.SystemArchitecture.businessService()")
    public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
        // start stopwatch
        Object retVal = pjp.proceed();
        // stop stopwatch
        return retVal;
    }

}
```

Spring 提供了非常简单的获取入参的方法，使用 org.aspectj.lang.JoinPoint 作为 Advice 的第一个参数即可，如：

```java
@Before("com.javadoop.springaoplearning.aop_spring_2_aspectj.SystemArchitecture.businessService()")
public void logArgs(JoinPoint joinPoint) {
    System.out.println("方法执行前，打印入参：" + Arrays.toString(joinPoint.getArgs()));
}
```

> 注意：第一，必须放置在第一个参数上；第二，如果是 @Around，我们通常会使用其子类 ProceedingJoinPoint，因为它有 procceed()/procceed(args[]) 方法。

@AspectJ 来实现上一节实现的**记录方法传参**和**记录方法返回值**。

```java
@Aspect
public class LogArgsAspect {

    // 这里可以设置一些自己想要的属性，到时候在配置的时候注入进来
    private boolean trace = true;

    @Before("com.javadoop.springaoplearning.aop_spring_2_aspectj.SystemArchitecture.businessService()")
    public void logArgs(JoinPoint joinPoint) {
        if (trace) {
            System.out.println("[@AspectJ]方法执行前，打印入参：" + Arrays.toString(joinPoint.getArgs()));
        }
    }

    public void setTrace(boolean trace) {
        this.trace = trace;
    }

}
@Aspect
public class LogResultAspect {

    private boolean trace;

    @AfterReturning(pointcut = "com.javadoop.springaoplearning.aop_spring_2_aspectj.SystemArchitecture.businessService()",
            returning = "result")
    public void logResult(Object result) {
        if (trace) {
            System.out.println("[@AspectJ]返回值：" + result);
        }
    }

    public void setTrace(boolean trace) {
        this.trace = trace;
    }
}
```

xml配置

```xml
<bean id="userService" class="com.javadoop.springaoplearning.service.imple.UserServiceImpl"/>
<bean id="orderService" class="com.javadoop.springaoplearning.service.imple.OrderServiceImpl"/>

<!--开启 @AspectJ 配置-->
<aop:aspectj-autoproxy/>

<bean id="logArgsAspect" class="com.javadoop.springaoplearning.aop_spring_2_aspectj.LogArgsAspect">
    <!--如果需要配置参数，和普通的 bean 一样操作-->
    <property name="trace" value="true"/>
</bean>

<bean class="com.javadoop.springaoplearning.aop_spring_2_aspectj.LogResultAspect">
    <property name="trace" value="true"/>
</bean>
```

运行

```java
public static void test_Spring_2_0_AspectJ() {
    // 启动 Spring 的 IOC 容器
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring_2_0_aspectj.xml");

    UserService userService = context.getBean(UserService.class);

    userService.createUser("Tom", "Cruise", 55);
    userService.queryUser();
}
```

输出

```shell
方法执行前，打印入参：[Tom, Cruise, 55]
User{firstName='Tom', lastName='Cruise', age=55, address='null'}
方法执行前，打印入参：[]
User{firstName='Tom', lastName='Cruise', age=55, address='null'}
```

#### Spring 2.0 schema-based 配置

 Spring 2.0 以后提供的基于 `<aop />` 命名空间的 XML 配置。这里说的 schema-based 就是指基于 `aop`  这个 schema。

> 解析 `<aop />` 的源码在 org.springframework.aop.config.AopNamespaceHandler 中。

**配置Aspect**

```xml
<aop:config>
    <aop:aspect id="myAspect" ref="aBean">
        ...
    </aop:aspect>
</aop:config>

<bean id="aBean" class="...">
    ...
</bean>
```

> 所有的配置都在 `<aop:config>` 下面。
>
> `<aop:aspect >` 中需要指定一个 bean，和前面介绍的 LogArgsAspect  和 LogResultAspect 一样，我们知道该 bean 中我们需要写处理代码。
>
> 然后，我们写好 Aspect 代码后，将其“织入”到合适的 Pointcut 中，这就是面向切面。

**配置 Pointcut**

```xml
<aop:config>

    <aop:pointcut id="businessService"
        expression="execution(* com.javadoop.springaoplearning.service.*.*(..))"/>

    <!--也可以像下面这样-->
    <aop:pointcut id="businessService2"
        expression="com.javadoop.SystemArchitecture.businessService()"/>

</aop:config>
```

> 将 `<aop:pointcut>` 作为 `<aop:config>` 的直接子元素，将作为全局 Pointcut。

也可以在 `<aop:aspect />`内部配置 Pointcut，这样该 Pointcut 仅用于该 Aspect：

```xml
<aop:config>
    <aop:aspect ref="logArgsAspect">
        <aop:pointcut id="internalPointcut"
                expression="com.javadoop.SystemArchitecture.businessService()" />
    </aop:aspect>
</aop:config>
```

配置 **Advice** 

```java
public class LogArgsAspect {

    // 这里可以设置一些自己想要的属性，到时候在配置的时候注入进来

    public void logArgs(JoinPoint joinPoint) {
        System.out.println("[schema-based]方法执行前，打印入参：" + Arrays.toString(joinPoint.getArgs()));
    }
}
public class LogResultAspect {

    public void logResult(Object result) {
        System.out.println("[schema-based]返回值：" + result);
    }
}
```

xml

```xml
<bean id="userService" class="com.javadoop.springaoplearning.service.imple.UserServiceImpl"/>
<bean id="orderService" class="com.javadoop.springaoplearning.service.imple.OrderServiceImpl"/>

<!--定义 bean，将作为 Aspect 使用，我们需要处理的逻辑代码都在里面-->
<bean id="logArgsAspect" class="com.javadoop.springaoplearning.aop_spring_2_schema_based.LogArgsAspect" />
<bean id="logResultAspect" class="com.javadoop.springaoplearning.aop_spring_2_schema_based.LogResultAspect" />

<aop:config>
    <!--下面这两个 Pointcut 是全局的，可以被所有的 Aspect 使用-->
    <!--这里示意了两种 Pointcut 配置-->
    <aop:pointcut id="logArgsPointcut" expression="execution(* com.javadoop.springaoplearning.service.*.*(..))" />
    <aop:pointcut id="logResultPointcut" expression="com.javadoop.springaoplearning.aop_spring_2_schema_based.SystemArchitecture.businessService()" />

    <aop:aspect ref="logArgsAspect">
        <!--在这里也可以定义 Pointcut，不过这是局部的，不能被其他的 Aspect 使用-->
        <aop:pointcut id="internalPointcut"
                      expression="com.javadoop.springaoplearning.aop_spring_2_schema_based.SystemArchitecture.businessService()" />
        <aop:before method="logArgs" pointcut-ref="internalPointcut" />
    </aop:aspect>

    <aop:aspect ref="logArgsAspect">
        <aop:before method="logArgs" pointcut-ref="logArgsPointcut" />
    </aop:aspect>

    <aop:aspect ref="logResultAspect">
        <aop:after-returning method="logResult" returning="result" pointcut-ref="logResultPointcut" />
    </aop:aspect>
</aop:config>
```

启动

```java
public static void test_Spring_2_0_Schema_Based() {
    // 启动 Spring 的 IOC 容器
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring_2_0_schema_based.xml");

    UserService userService = context.getBean(UserService.class);

    userService.createUser("Tom", "Cruise", 55);
}
```

### SpringAOP原理

#### 流程

```bash
流程：
 　       1）、传入配置类，创建ioc容器
          2）、注册配置类，调用refresh（）刷新容器；
          3）、registerBeanPostProcessors(beanFactory);注册bean的后置处理器来方便拦截bean的创建；
              1）、先获取ioc容器已经定义了的需要创建对象的所有BeanPostProcessor
              2）、给容器中加别的BeanPostProcessor
              3）、优先注册实现了PriorityOrdered接口的BeanPostProcessor；
              4）、再给容器中注册实现了Ordered接口的BeanPostProcessor；
              5）、注册没实现优先级接口的BeanPostProcessor；
              6）、注册BeanPostProcessor，实际上就是创建BeanPostProcessor对象，保存在容器中；
                  创建internalAutoProxyCreator的BeanPostProcessor【AnnotationAwareAspectJAutoProxyCreator】
                  1）、创建Bean的实例
                  2）、populateBean；给bean的各种属性赋值
                  3）、initializeBean：初始化bean；
                          1）、invokeAwareMethods()：处理Aware接口的方法回调
                          2）、applyBeanPostProcessorsBeforeInitialization()：应用后置处理器的postProcessBeforeInitialization（）
                          3）、invokeInitMethods()；执行自定义的初始化方法
                          4）、applyBeanPostProcessorsAfterInitialization()；执行后置处理器的postProcessAfterInitialization（）；
                  4）、BeanPostProcessor(AnnotationAwareAspectJAutoProxyCreator)创建成功；--》aspectJAdvisorsBuilder
              7）、把BeanPostProcessor注册到BeanFactory中；
                  beanFactory.addBeanPostProcessor(postProcessor);
  =======以上是创建和注册AnnotationAwareAspectJAutoProxyCreator的过程========
  
              AnnotationAwareAspectJAutoProxyCreator => InstantiationAwareBeanPostProcessor
          4）、finishBeanFactoryInitialization(beanFactory);完成BeanFactory初始化工作；创建剩下的单实例bean
              1）、遍历获取容器中所有的Bean，依次创建对象getBean(beanName);
                  getBean->doGetBean()->getSingleton()->
              2）、创建bean
                  【AnnotationAwareAspectJAutoProxyCreator在所有bean创建之前会有一个拦截，InstantiationAwareBeanPostProcessor，
　　　　　　　　　　　　会调用postProcessBeforeInstantiation()】
                  1）、先从缓存中获取当前bean，如果能获取到，说明bean是之前被创建过的，直接使用，否则再创建；
                      只要创建好的Bean都会被缓存起来
                  2）、createBean（）;创建bean；
                      AnnotationAwareAspectJAutoProxyCreator 会在任何bean创建之前先尝试返回bean的实例
                      【BeanPostProcessor是在Bean对象创建完成初始化前后调用的】
                      【InstantiationAwareBeanPostProcessor是在创建Bean实例之前先尝试用后置处理器返回对象的】
                      1）、resolveBeforeInstantiation(beanName, mbdToUse);解析BeforeInstantiation
                          希望后置处理器在此能返回一个代理对象；如果能返回代理对象就使用，如果不能就继续
                          1）、后置处理器先尝试返回对象；
                              bean = applyBeanPostProcessorsBeforeInstantiation（）：
                                  拿到所有后置处理器，如果是InstantiationAwareBeanPostProcessor;
                                  就执行postProcessBeforeInstantiation
                              if (bean != null) {
                                bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                            }
  
                      2）、doCreateBean(beanName, mbdToUse, args);真正的去创建一个bean实例；和3.6流程一样；
                      3）、
                      
AnnotationAwareAspectJAutoProxyCreator【InstantiationAwareBeanPostProcessor】    的作用：
 1）、每一个bean创建之前，调用postProcessBeforeInstantiation()；
          关心MathCalculator和LogAspect的创建
          1）、判断当前bean是否在advisedBeans中（保存了所有需要增强bean）
          2）、判断当前bean是否是基础类型的Advice、Pointcut、Advisor、AopInfrastructureBean，
              或者是否是切面（@Aspect）
          3）、是否需要跳过
              1）、获取候选的增强器（切面里面的通知方法）【List<Advisor> candidateAdvisors】
                  每一个封装的通知方法的增强器是 InstantiationModelAwarePointcutAdvisor；
                  判断每一个增强器是否是 AspectJPointcutAdvisor 类型的；返回true
              2）、永远返回false
 2）、创建对象
  postProcessAfterInitialization；
          return wrapIfNecessary(bean, beanName, cacheKey);//包装如果需要的情况下
          1）、获取当前bean的所有增强器（通知方法）  Object[]  specificInterceptors
             1、找到候选的所有的增强器（找哪些通知方法是需要切入当前bean方法的）
             2、获取到能在bean使用的增强器。
             3、给增强器排序
         2）、保存当前bean在advisedBeans中；
         3）、如果当前bean需要增强，创建当前bean的代理对象；
             1）、获取所有增强器（通知方法）
             2）、保存到proxyFactory
             3）、创建代理对象：Spring自动决定
                 JdkDynamicAopProxy(config);jdk动态代理；
                 ObjenesisCglibAopProxy(config);cglib的动态代理；
         4）、给容器中返回当前组件使用cglib增强了的代理对象；
         5）、以后容器中获取到的就是这个组件的代理对象，执行目标方法的时候，代理对象就会执行通知方法的流程；       
  3）、目标方法执行    ；         容器中保存了组件的代理对象（cglib增强后的对象），这个对象里面保存了详细信息（比如增强器，目标对象，xxx）；
         1）、CglibAopProxy.intercept();拦截目标方法的执行
         2）、根据ProxyFactory对象获取将要执行的目标方法拦截器链；
             List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
             1）、List<Object> interceptorList保存所有拦截器 5
                 一个默认的ExposeInvocationInterceptor 和 4个增强器；
             2）、遍历所有的增强器，将其转为Interceptor；
                 registry.getInterceptors(advisor);
             3）、将增强器转为List<MethodInterceptor>；
                 如果是MethodInterceptor，直接加入到集合中
                 如果不是，使用AdvisorAdapter将增强器转为MethodInterceptor；
                 转换完成返回MethodInterceptor数组；
         3）、如果没有拦截器链，直接执行目标方法;拦截器链（每一个通知方法又被包装为方法拦截器，利用MethodInterceptor机制）
         4）、如果有拦截器链，把需要执行的目标对象，目标方法，截器链等信息传入创建一个 CglibMethodInvocation 对象，并调用 Object retVal =  mi.proceed();
         5）、拦截器链的触发过程;        
         		1)、如果没有拦截器执行执行目标方法，或者拦截器的索引和拦截器数组-1大小一样（指定到了最后一个拦截器）执行目标方法；        
         		2)、链式获取每一个拦截器，拦截器执行invoke方法，每一个拦截器等待下一个拦截器执行完成返回以后再来执行；
         		拦截器链的机制，保证通知方法与目标方法的执行顺序；
```

```bash
 1）、 @EnableAspectJAutoProxy 开启AOP功能
 2）、 @EnableAspectJAutoProxy 会给容器中注册一个组件 AnnotationAwareAspectJAutoProxyCreator
 3）、AnnotationAwareAspectJAutoProxyCreator是一个后置处理器；
 4）、容器的创建流程：
        1）、registerBeanPostProcessors（）注册后置处理器；创建AnnotationAwareAspectJAutoProxyCreator对象
        2）、finishBeanFactoryInitialization（）初始化剩下的单实例bean
            1）、创建业务逻辑组件和切面组件
            2）、AnnotationAwareAspectJAutoProxyCreator拦截组件的创建过程
            3）、组件创建完之后，判断组件是否需要增强
                     是：切面的通知方法，包装成增强器（Advisor）;给业务逻辑组件创建一个代理对象（cglib）；
 5）、执行目标方法：
        1）、代理对象执行目标方法
        2）、CglibAopProxy.intercept()；
             1）、得到目标方法的拦截器链（增强器包装成拦截器MethodInterceptor）
             2）、利用拦截器的链式机制，依次进入每一个拦截器进行执行；
             3）、效果：
                      正常执行：前置通知-》目标方法-》后置通知-》返回通知
                      出现异常：前置通知-》目标方法-》后置通知-》异常通知
```

