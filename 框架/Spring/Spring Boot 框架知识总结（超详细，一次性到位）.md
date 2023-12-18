> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/362984115)

**前言**

本篇文章包含 Springboot 配置文件解释、热部署、自动装配原理源码级剖析、内嵌 tomcat 源码级剖析、缓存深入、多环境部署等等，如果能耐心看完，想必会有不少收获。

### 一、Spring Boot 基础应用

### Spring Boot 特征

**概念：**

约定优于配置，简单来说就是你所期待的配置与约定的配置一致，那么就可以不做任何配置，约定不符合期待时才需要对约定进行替换配置。

**特征：**

1. SpringBoot Starter：他将常用的依赖分组进行了整合，将其合并到一个依赖中，这样就可以一次性添加到项目的 Maven 或 Gradle 构建中。

2, 使编码变得简单，SpringBoot 采用 JavaConfig 的方式对 Spring 进行配置，并且提供了大量的注解，极大的提高了工作效率，比如 @Configuration 和 @bean 注解结合，基于 @Configuration 完成类扫描，基于 @bean 注解把返回值注入 IOC 容器。

3. 自动配置：SpringBoot 的自动配置特性利用了 Spring 对条件化配置的支持，合理地推测应用所需的 bean 并自动化配置他们。

4. 使部署变得简单，SpringBoot 内置了三种 Servlet 容器，Tomcat，Jetty,undertow. 我们只需要一个 Java 的运行环境就可以跑 SpringBoot 的项目了，SpringBoot 的项目可以打成一个 jar 包。

### Spring Boot 创建

Spring Boot 项目结构图：

![](https://pic2.zhimg.com/v2-547fd47b7511da47e58c649b304b57dd_r.jpg)

### Spring Boot 热部署

通过引入 spring-bootdevtools 插件，可以实现不重启服务器情况下，对项目进行即时编译。引入热部署插件的步骤如下：

1. 在 pom.xml 添加热部署依赖

![](https://pic2.zhimg.com/v2-681b63a841ce017d7a63179702e7bccd_r.jpg)

2. IDEA 热部署工具设置

![](https://pic3.zhimg.com/v2-4467a1e457af28033b7ec6380f011d4a_r.jpg)

3. 在项目任意页面中使用组合快捷键 “Ctrl+Shift+Alt+/” 打开 Maintenance 选项框，选中并打开 Registry 页面，列表中找到“compiler.automake.allow.when.app.running”，将该选项后的 Value 值勾选，用于指定 IDEA 工具在程序运行过程中自动编译，最后单击【Close】按钮完成设置。

![](https://pic3.zhimg.com/v2-563eaf5ce882686336c8c0aff07207ce_r.jpg)

**热部署原理**：

基本原理就是我们在编辑器上启动项目，然后改动相关的代码，然后编辑器自动触发编译，替换掉历史的. class 文件后，项目检测到有文件变更后会重启 srpring-boot 项目。内部主要是通过引入的插件对我们的 classpath 资源变化进行监听，当 classpath 有变化，才会触发重启。

![](https://pic4.zhimg.com/v2-14d162040ff1c4f5475f2c2cab0de8b3_r.jpg)

从官方文档可以得知，其实这里对类加载采用了两种类加载器，对于第三方 jar 包采用 baseclassloader 来加载，对于开发人员自己开发的代码则使用 restartClassLoader 来进行加载，这使得比停掉服务重启要快的多，因为使用插件只是重启开发人员编写的代码部分。

![](https://pic3.zhimg.com/v2-7546b2b745a3b9975442508d7db5a952_r.jpg)

**排除资源:**

默认情况下，改变资源 /META-INF/maven ， /META-INF/resources ， /resources ， /static ， /public ，或 /templates 不触发重新启动，但确会触发现场重装。如果要自定义这些排除项，则可以使用该 spring.devtools.restart.exclude 属性。例如，仅排除 /static ， /public 在 application.properties 设置以下属性。

spring.devtools.restart.exclude=static/**,public/**,config/**

### 全局配置文件优先级

优先级：以下图顺序号代表配置文件的优先级，并且相同配置文件按顺序加载可以实现互补，但是不会被覆盖。

![](https://pic2.zhimg.com/v2-82fed3dc047df18450d04703cfb59d09_r.jpg)

注意：Spring Boot 有 application.properties 和 application.yaml 两种配置文件的方式，yaml 是一种 JSON 超文本格式文件，如果是 2.4.0 之前版本，优先级 properties>yaml；但是如果是 2.4.0 的版本，优先级 yaml>properties。

### 自定义 application.properties 配置文件注入 IOC 容器

![](https://pic1.zhimg.com/v2-9d6b40b04964e201dce3dbc6641f75d4_r.jpg)![](https://pic1.zhimg.com/v2-3c46f5da42820de49e5eb8c7fab65ee0_r.jpg)

填加相应依赖配置可以实现在自定义配置 properties 配置提示

![](https://pic2.zhimg.com/v2-92dc165f57f1e236ea6d73653a8db5ad_r.jpg)

@ConfigurationProperties(prefix = "person") 注解的作用是将配置文件中以 person 开头的属性值通过 setXX() 方法注入到实体类对应属性中。

@Component 注解的作用是将当前注入属性值的 Person 类对象作为 Bean 组件放到 Spring 容器中，只有这样才能被 @ConfigurationProperties 注解进行赋值。

### application.yaml 配置文件

YAML 文件格式是 Spring Boot 支持的一种 JSON 超集文件格式，以数据为中心，比 properties、xml 等更

适合做配置文件.

1.yml 和 xml 相比，少了一些结构化的代码，使数据更直接，一目了然

2. 相比 properties 文件更简洁

3.yaml 文件的扩展名可以使用. yml 或者. yaml。

4.application.yml 文件使用 “key:（空格）value” 格式配置属性，使用缩进控制层级关系。

![](https://pic1.zhimg.com/v2-f8d9419e8254f857d0539b2e146583fc_r.jpg)

### 属性注入

如果配置属性是 Spring Boot 已有属性，例如服务端口 server.port，那么 Spring Boot 内部会自动扫描并读取这些配置文件中的属性值并覆盖默认属性。

@Configuration：声明一个类作为配置类。

@Bean：声明在方法上，将方法的返回值加入 Bean 容器。

@Value：属性注入

![](https://pic4.zhimg.com/v2-46bc2d61fb489ac6479cc6dc3d904b3f_r.jpg)![](https://pic4.zhimg.com/v2-7d01e58adb8aa9b9e084346fb3457f6f_r.jpg)

@ConfigurationProperties(prefix = "jdbc")：批量属性注入。

![](https://pic4.zhimg.com/v2-66cfb71ce0fe557b0f01ee2f5fc962db_r.jpg)

@PropertySource("classpath:/jdbc.properties") 指定外部属性文件，在类上添加。

**第三方配置：**

@ConfigurationProperties 用于注释类之外，您还可以在公共 @Bean 方法上使用它。将属性绑定到控件之外的第三方组件

![](https://pic3.zhimg.com/v2-2a25e9bf2907c9324e02df140a0a479e_r.jpg)![](https://pic3.zhimg.com/v2-200a5543f214f7ffc51a355cd8010386_r.jpg)

**松散绑定：**

Spring Boot 使用一些宽松的规则将环境属性绑定到 @ConfigurationProperties bean，因此环境属性名和 bean 属性名之间不需要完全匹配，比如在 application.properties 文件里定义一个 first-name=tom，在对应 bean 类中使用 firstName 也能获取到对应的值，这就是松散绑定。

![](https://pic1.zhimg.com/v2-196fe96ce06902e7edad3a4d2c8f195c_r.jpg)![](https://pic4.zhimg.com/v2-5a894b73358fa839b3a62fb2ac1c9d37_r.jpg)

### Spring Boot 日志框架

![](https://pic2.zhimg.com/v2-bab52f59e94863057502641359786101_r.jpg)

**SLF4J 的使用：**

![](https://pic3.zhimg.com/v2-f2280e410b105c13c3e1a6be4c4cdc12_r.jpg)

注意: 由于每一个日志的实现框架都有自己的配置文件，所以在使用 SLF4j 之后，配置文件还是要使用实现日志框架的配置文件。

**统一日志框架使用：**

实现步骤

1. 排除系统中的其他日志框架。

2. 使用中间包替换要替换的日志框架。

3. 导入我们选择的 SLF4J 实现。

![](https://pic3.zhimg.com/v2-70bb63dd32b6ae135ee44f0a82aa4e12_r.jpg)

从图中我们得到一种统一日志框架使用的方式，可以使用一种要替换的日志框架类完全一样的 jar 进行替换，这样不至于原来的第三方 jar 报错，而这个替换的 jar 其实使用了 SLF4J API. 这样项目中的日志就都可以通过 SLF4J API 结合自己选择的框架进行日志输出。

**Spring Boot 的日志关系：**

![](https://pic4.zhimg.com/v2-a2fe7a4e7005ecd3f44e152ba691b49b_r.jpg)

Spring Boot 默认已经使用了 SLF4J + LogBack . 所以我们在不进行任何额外操作的情况下就可以使用 SLF4J + Logback 进行日志输出。SLF4J 日志级别从小到大 trace,debug,info,warn,error，默认是 info 级别。

**自定义日志输出：**

可以在配置文件编写日志相关配置实现自定义日志输出。

![](https://pic4.zhimg.com/v2-cb1e6741b77460de2218ff471954e083_r.jpg)

**替换日志框架：**

![](https://pic3.zhimg.com/v2-cd1bd350c238d0dfaf964490be563cee_r.jpg)

### 二、Spring Boot 源码分析

### spring-boot-starter-parent

Spring Boot 项目的统一版本父项目依赖管理。

![](https://pic1.zhimg.com/v2-5627b3a03583ea6364b89d39d4864970_r.jpg)

在底层源文件定义了工程的 Java 版本；工程代码的编译源文件编码格式；工程编译后的文件编码格式；Maven 打包编译的版本。

![](https://pic2.zhimg.com/v2-81d3560f78fa1d53e0476fe26ca62a4d_r.jpg)

接着在 build 节点做了资源过滤

![](https://pic3.zhimg.com/v2-efbdd5a6b19c41cf74eb48480555ad1a_r.jpg)

接着从 spring-boot-starter-parent 找到他父依赖 spring-boot-dependencies，从里面就可以发现里面定义了各种版本声明，通过这里声明可以让部分依赖不需要写版本号，一些没有引入的第三方 jar 包仍然需要自己声明版本号。

![](https://pic2.zhimg.com/v2-0ee0fcbd1904be49bd45070a54d9eaf9_r.jpg)

### spring-boot-starter-web

Spring Boot 项目的所依赖 jar 包进行打包起步依赖管理

在 spring-boot-starter-web 的父依赖 spring-boot-starters 包中，可以发现在他的 dependencies 标签有着各种依赖包引入，点进去就是具体包的导入配置管理。

![](https://pic2.zhimg.com/v2-c0f97725f30aadbf366c8f918fab4e21_r.jpg)![](https://pic4.zhimg.com/v2-69cc508febb9538441aede0170b18873_r.jpg)

注意：Spring Boot 官方并不是针对所有场景开发的技术框架都提供了场景启动器，例如阿里巴巴的 Druid 数据源等，Spring Boot 官方就没有提供对应的依赖启动器。为了充分利用 Spring Boot 框架的优势，在 Spring Boot 官方没有整合这些技术框架的情况下，Druid 等技术框架所在的开发团队主动与 Spring Boot 框架进行了整合，实现了各自的依赖启动器，例如 druid-spring-boot-starter 等。我们在 pom.xml 文件中引入这些第三方的依赖启动器时，切记要配置对应的版本号。

### 自动配置 @SpringBootApplication

他是一个组合注解，核心代码：

![](https://pic1.zhimg.com/v2-ab6e7c4d9197aa1ea09d82e7f00178cc_r.jpg)

### 自动配置 @SpringBootConfiguration

### 通过上面可以发现我们的核心启动类注解源码中含此注解，这个注解标注在某个类上，表示这是一个 Spring Boot 的配置类。他的核心代码中，内部有一个核心注解 @Configuration 来表明当前类是配置类，并且可以被组件扫描器扫到，所以 @SpringBootConfiguration 与 @Configuration 具有相同作用，只是前者又做了一次封装。

![](https://pic4.zhimg.com/v2-046327ed30d8c95dea3575fef62d47e3_r.jpg)

### 自动配置 @ EnableAutoConfiguration

Spring 中有很多以 Enable 开头的注解，其作用就是借助 @Import 来收集并注册特定场景相关的 Bean ，并加载到 IOC 容器。而这个注解就是借助 @Import 来收集所有符合自动配置条件的 bean 定义，并加载到 IoC 容器，他的核心源码如下：

![](https://pic1.zhimg.com/v2-2179ecd3c26db2e07b22c501b92799d4_r.jpg)

通过 @AutoConfigurationPackage 注解进入类别，发现他通过 import 引入了一个 AutoConfigurationPackages.Registrar.class，在 Registrar.class 中就重写了一个 registerBeanDefinitions 方法，在方法内部调用了一个 register 方法来实现将注解标注的元信息传入，获取到相应的包名。通俗点就是注册 bean，然后根据 @AutoConfigurationPackage 找到需要注册 bean 的类路径，这个路径就被自动保存了下来，后面需要使用 bean，就直接获取使用，比如 Spring Boot 整合 JPA 可以完成一些注解扫描。

![](https://pic1.zhimg.com/v2-ed7df0ce4f9bf2b1e44dd98a10f7bfb0_r.jpg)

### 自动配置 @Import(AutoConfigurationImportSelector.class)

该注解是 Spring boot 的底层注解，AutoConfigurationImportSelector 类可以帮助 Springboot 应用将所有符合条件的 @Configuration 配置都加载到当前 Spring Boot 创建并使用的 IOC 容器 (ApplicationContext) 中。

![](https://pic1.zhimg.com/v2-aebe7f29357a93dfa7d3e8583c3704d4_r.jpg)![](https://pic1.zhimg.com/v2-921a13c0228d4e55da498b0418c7196c_r.jpg)

该注解实现了实现了 DeferredImportSelector 接口和各种 Aware 接口，在源码中截图中，通过四个接口回调，把值返回给了

定义的四个成员变量。

1. 自动配置逻辑相关的入口方法在 DeferredImportSelectorGrouping 类的 getImports 方法。

![](https://pic2.zhimg.com/v2-1a3d199490c191d25e0f709ce4b16c85_r.jpg)

2. 自动配置的相关的绝大部分逻辑全在第一处也就是 this.group.proces 方法里，主要做的事就是在方法中，传入的 AutoConfigurationImportSelector 对象来选择一些符合条件的自动配置类，过滤掉一些不符合条件的自动配置类，而第二处的 this.group.selectImports 的方法主要是针对前面的 process 方法处理后的自动配置类再进一步有选择的选择导入。

![](https://pic3.zhimg.com/v2-1408952343e2c4cf8055fa0c4b2a91c6_r.jpg)

3. 进入 getAutoConfigurationEntry 方法，这个方法主要是用来获取自动配置类有关，承担了自动配置的主要逻辑。AutoConfigurationEntry 方法主要做的事情就是获取符合条件的自动配置类，避免加载不必要的自动配置类从而造成内存浪费

![](https://pic2.zhimg.com/v2-ea80b4ce6df3e4f6566e529500b6e6d9_r.jpg)![](https://pic1.zhimg.com/v2-af8820e9cbd3bcf8c2d62d0a315b4bac_r.jpg)

4. 进入 getCandidateConfigurations 方法，里面有一个重要方法 loadFactoryNames ，这个方法是让 SpringFactoryLoader 去加载一些组件的名字。

![](https://pic1.zhimg.com/v2-75d35b6d5596b483dadac68cedc3b68c_r.jpg)

5. 进入 loadFactoryNames 方法，获取到出入的键 factoryClassName。

![](https://pic2.zhimg.com/v2-868085c7957d052673fc4e8b4e0b7185_r.jpg)

6. 进入 loadSpringFactories 方法，加载配置文件，而这个配置文件就是 spring.factories 文件

![](https://pic4.zhimg.com/v2-a470c307d7fd48396a578ffb742601db_r.jpg)![](https://pic3.zhimg.com/v2-bc946ff6b39e3cb434d3debcc30d23a2_r.jpg)

由此我们可以知道，在这个方法中会遍历整个 ClassLoader 中所有 jar 包下的 spring.factories 文件。spring.factories 里面保存着 springboot 的默认提供的自动配置类。

**AutoConfigurationEntry 方法主要做的事情：**

【1】从 spring.factories 配置文件中加载 EnableAutoConfiguration 自动配置类）, 获取的自动配置类如图所示。

【2】若 @EnableAutoConfiguration 等注解标有要 exclude 的自动配置类，那么再将这个自动配置类排除掉；

【3】排除掉要 exclude 的自动配置类后，然后再调用 filter 方法进行进一步的过滤，再次排除一些不符合条件的自动配置类；

【4】经过重重过滤后，此时再触发 AutoConfigurationImportEvent 事件，告诉 ConditionEvaluationReport 条件评估报告器对象来记录符合条件的自动配置类；

【5】 最后再将符合条件的自动配置类返回。

**AutoConfigurationImportSelector 的 filter 方法**

主要做的事情就是调用 AutoConfigurationImportFilter 接口的 match 方法来判断每一个自动配置类上的条件注解（若有的话） @ConditionalOnClass , @ConditionalOnBean 或 @ConditionalOnWebApplication 是否满足条件，若满足，则返回 true，说明匹配，若不满足，则返回 false 说明不匹配。其实就是排除自动配置类，因为全部加载出来的类太多，不需要全部都反射成对象，而这个方法就是通过注解进行该自动配置类是否有相应匹配的类的判断，存在即加入，不存在即过滤。

@Conditional 是 Spring4 新提供的注解，它的作用是按照一定的条件进行判断，满足条件给容器注册 bean。

@ConditionalOnBean：仅仅在当前上下文中存在某个对象时，才会实例化一个 Bean。

@ConditionalOnClass：某个 class 位于类路径上，才会实例化一个 Bean。

@ConditionalOnExpression：当表达式为 true 的时候，才会实例化一个 Bean。基于 SpEL 表达式的条件判断。

@ConditionalOnMissingBean：仅仅在当前上下文中不存在某个对象时，才会实例化一个 Bean。

@ConditionalOnMissingClass：某个 class 类路径上不存在的时候，才会实例化一个 Bean。

@ConditionalOnNotWebApplication：不是 web 应用，才会实例化一个 Bean。

@ConditionalOnWebApplication：当项目是一个 Web 项目时进行实例化。

@ConditionalOnNotWebApplication：当项目不是一个 Web 项目时进行实例化。

@ConditionalOnProperty：当指定的属性有指定的值时进行实例化。

@ConditionalOnJava：当 JVM 版本为指定的版本范围时触发实例化。

@ConditionalOnResource：当类路径下有指定的资源时触发实例化。

@ConditionalOnJndi：在 JNDI 存在的条件下触发实例化。

@ConditionalOnSingleCandidate：当指定的 Bean 在容器中只有一个，或者有多个但是指定了首选的 Bean 时触发实例化。

**有选择的导入自动配置类**

在第一步最后的一个方法 this.group.selectImports 主要是针对经过排除掉 exclude 的和被 AutoConfigurationImportFilter 接口过滤后的满足条件的自动配置类再进一步排除 exclude 的自动配置类，然后再排序，至此实现了自动配置。

![](https://pic2.zhimg.com/v2-ccbdb3c4674074d54a32a5b165eb7475_r.jpg)

综合以上总结：

![](https://pic4.zhimg.com/v2-70b19eb776556de1c01454ff58de3e17_r.jpg)

### 自动配置 HttpEncodingAutoConfiguration 实例

![](https://pic2.zhimg.com/v2-3a37ceedb2cc9d19a13ef7a34bf8f7d1_r.jpg)![](https://pic1.zhimg.com/v2-6935ab05923e55d2e938200e3a12b988_r.jpg)

1. SpringBoot 启动会加载大量的自动配置类

2. 我们看我们需要实现的功能有没有 SpringBoot 默认写好的自动配置类

3. 我们再来看这个自动配置类中到底配置了哪些组件；（只要我们有我们要用的组件，我们就不需要再来配置了）

4. 给容器中自动配置类添加组件的时候，会从 properties 类中获取某些属性，我们就可以在配置文件中指定这些属性的值。

xxxAutoConfiguration ：自动配置类，用于给容器中添加组件从而代替之前我们手动完成大量繁琐的配置。

xxxProperties : 封装了对应自动配置类的默认属性值，如果我们需要自定义属性值，只需要根据 xxxProperties 寻找相关属性在配置文件设值即可。

### @ComponentScan 注解

主要作用是从定义的扫描路径中，找出标识了需要装配的类自动装配到 spring 的 bean 容器中。默认扫描路径是为 @ComponentScan 注解的类所在的包为基本的扫描路径（也就是标注了 @SpringBootApplication 注解的项目启动类所在的路径），所以这里就解释了之前 spring boot 为什么只能扫描自己所在类的包及其子包。

常用属性：

basePackages、value：指定扫描路径，如果为空则以 @ComponentScan 注解的类所在的包为基本的扫描路径。

basePackageClasses：指定具体扫描的类。

includeFilters：指定满足 Filter 条件的类。

excludeFilters：指定排除 Filter 条件的类。

![](https://pic3.zhimg.com/v2-0b9745335e09d707774295329dd2f0da_r.jpg)

SpringApplication() 构造方法

![](https://pic4.zhimg.com/v2-e547ed4682934cadb3153aff2604b1af_r.jpg)

第二步 getSpringFactoriesInstances 方法解析

![](https://pic4.zhimg.com/v2-bb5291c3b15d677b91e58af4f9916907_r.jpg)

主要就是 loadFactoryNames() 方法，这个方法是 spring-core 中提供的从 META-INF/spring.factories 中获取指定的类（key）的同一入口方法，获取的是 key 为 org.springframework.context.ApplicationContextInitializer 的类，是 Spring 框架的类, 这个类的主要目的就是在 ConfigurableApplicationContext 调用 refresh() 方法之前，回调这个类的 initialize 方法。通过 ConfigurableApplicationContext 的实例获取容器的环境 Environment，从而实现对配置文件的修改完善等工作。

![](https://pic3.zhimg.com/v2-dab2c72f5a1f878026c75aa1a51a9e6a_r.jpg)![](https://pic3.zhimg.com/v2-1dc9669949a7ac5b9eaf0a8c3f36171a_r.jpg)

### 源码剖析 Run 方法整体流程

重要六步：

第一步：获取并启动监听器

第二步：构造应用上下文环境

第三步：初始化应用上下文

第四步：刷新应用上下文前的准备阶段

第五步：刷新应用上下文

第六步：刷新应用上下文后的扩展接口

![](https://pic4.zhimg.com/v2-1e6100b11d0252991089d6764d65cbfb_r.jpg)![](https://pic2.zhimg.com/v2-94ecab2625048a815b5d8231dcea5c81_r.jpg)![](https://pic3.zhimg.com/v2-0823169e8e7cd2ccdb3cf9d5110a180a_r.jpg)

### Run 方法第一步：获取并启动监听器

事件机制在 Spring 是很重要的一部分内容，通过事件机制我们可以监听 Spring 容器中正在发生的一些事件，同样也可以自定义监听事件。Spring 的事件为 Bean 和 Bean 之间的消息传递提供支持。当一个对象处理完某种任务后，通知另外的对象进行某些处理，常用的场景有进行某些操作后发送通知，消息、邮件等情况。

![](https://pic4.zhimg.com/v2-11fd5acfa745d0b7c33f0c6c30aae50f_r.jpg)

通过 getRunListeners 方法来获取监听器，在 getRunListeners 方法内部调用了一个 getSpringFactoriesInstances 方法，返回值是一个 SpringApplicationRunListeners 有参构造的监听器类，这个方法加载 SpringApplicationRunListener 类，把这个类当做 key, 这个类的作用就是负责在 SpringBoot 启动的不同阶段，广播出不同的消息，传递给 ApplicationListener 监听器实现类。

![](https://pic1.zhimg.com/v2-0791d3fc7432e2fa2a83ec9f39a87b70_r.jpg)

getSpringFactoriesInstances 方法被重复使用。

![](https://pic3.zhimg.com/v2-f03ae84183ddc2644f347e493277af6e_r.jpg)

总结：如何获取到监听器并进行启动开启监听。

### Run 方法第二步：构造应用上下文环境

应用上下文环境包括什么呢？包括计算机的环境，Java 环境，Spring 的运行环境，Spring 项目的配置（在 SpringBoot 中就是那个熟悉的 application.properties/yml）等等。

![](https://pic3.zhimg.com/v2-e53f2643f3a0606937cc7baf0ebf8886_r.jpg)

通过 prepareEnvironment 方法创建并按照相应的应用类型配置相应的环境，然后根据用户的配置，配置系统环境，然后启动监听器，并加载系统配置文件。

主要步骤方法：

getOrCreateEnvironment 方法

configureEnvironment 方法

listeners.environmentPrepared 方法

![](https://pic4.zhimg.com/v2-89a1924292b8390a144718807ed97b8f_r.jpg)

总结：最终目的就是把环境信息封装到 environment 对象中，方便后面使用。

### Run 方法第三步：初始化应用上下文

通过 createApplicationContext 方法构建应用上下文对象 context，而 context 中有一个属性 beanFactory 他是一个 DefaultListableBeanFactory 类，这就是我们所说的 IoC 容器。应用上下文对象初始化的同时 IOC 容器也被创建了。

![](https://pic4.zhimg.com/v2-dc58004a87011062fb8cb341c354b9df_r.jpg)

在 SpringBoot 工程中，应用类型分为三种

![](https://pic4.zhimg.com/v2-757bf1e06eb243c698991599314efbb3_r.jpg)

通过反射拿到配置类的字节码对象并通过 BeanUtils.instantiateClass 方法进行实例化并返回。

![](https://pic3.zhimg.com/v2-b0231e81c593502a788c0fa7ec71ce0e_r.jpg)

总结：就是创建应用上下文对象同时创建了 IOC 容器。

### Run 方法第四步：刷新应用上下文前的准备阶段

主要的目的就是为前面的上下文对象 context 进行一些属性值的设置，在执行过程中还要完成一些 Bean 对象的创建，其中就包含核心启动类的创建。

![](https://pic4.zhimg.com/v2-7f6d3c5722c9674e3250045553f92423_r.jpg)

属性设置

![](https://pic4.zhimg.com/v2-f58152cb940e9f6975db26a80874c30f_r.jpg)

Bean 对象创建

![](https://pic2.zhimg.com/v2-542814d1a89f8ab7f6d966be93f19a59_r.jpg)

Spring 容器在启动的时候，会将类解析成 Spring 内部的 beanDefintion 结构，并将 beanDefintion 存储到 DefaultListableBeanFactory 的 Map 中。BeanDefinitionLoader 方法就是完成赋值。

![](https://pic3.zhimg.com/v2-b907de96388ba5d329b230dbf3e318a6_r.jpg)![](https://pic3.zhimg.com/v2-ab266871b53a12740877843c927b8a2e_r.jpg)

总结：就是应用上下文属性的设置并把核心启动类生成实例化对象存储到容器中。

### Run 方法第五步：刷新应用上下文

Spring Boot 的自动配置原理：通常来说主要就是依赖核心启动类上面的 @SpringBootApplication 注解，这个注解是一个组合注解，他组合了 @EnableAutoConfiguration 这个注解，在 run 方法启动会执行 getImport 方法，最终找到 process 方法，进行注解的扫描，通过注解组合关系，在底层借助 @Import 注解向容器导入 AutoConfigurationImportSelector.class 组件类，这个类在执行过程中他会去加载 WEB-INF 下名称为 spring.factories 的文件，从这个文件中根据 EnableAutoConfiguration 这个 key 来加载 pom.xml 引入的所有对应自动配置工厂类的全部路径配置，在经过过滤，选出真正生效的自动配置工厂类去生成实例存到容器中，从而完成自动装配。如果从 Main 方法的 Run 方法出发，了解实际实现的原理，就能知道他是怎么通过 Main 方法找到主类，然后再扫描主类注解，完成一系列操作。而在刷新应用上下文这步就是根据找到的主类来执行解析注解，完成自动装配的一系列过程。

通过 refreshContext() 方法一路跟下去，最终来到 AbstractApplicationContext 类的 refresh() 方法，其中最重要的方法就是 invokeBeanFactoryPostProcessors 方法，他就是在上下文中完成 Bean 的注册。

![](https://pic3.zhimg.com/v2-63c4fa85ff2494daf3bbce66ba53b18e_r.jpg)

**运行步骤：**

1.prepareRefresh() 刷新上下文

2.obtainFreshBeanFactory() 在第三步初始化应用上下文中我们创建了应用的上下文，并触发了 GenericApplicationContext 类的构造方法如下所示，创建了 beanFactory，也就是创建了 DefaultListableBeanFactory 类，这里就是拿到之前创建的 beanFactory。

3.prepareBeanFactory() 对上面获取的 beanFactory，准备 bean 工厂，以便在此上下文中使用。

4.postProcessBeanFactory() 向上下文中添加了一系列的 Bean 的后置处理器。

接着就进入到我们最重要的 invokeBeanFactoryPostProcessors() 方法，完成了 IoC 容器初始化过程的三个步骤：

1） 第一步：Resource 定位

![](https://pic3.zhimg.com/v2-db59491e6362196eaa484927b8300472_r.jpg)

2） 第二步：BeanDefinition 的载入

![](https://pic3.zhimg.com/v2-ec058905218e8316dda9999fa534af6a_r.jpg)

3） 第三个过程：注册 BeanDefinition

![](https://pic1.zhimg.com/v2-7a2302437a9a09e4bfddeec9904eed30_r.jpg)

总结：spring 启动过程中，就是通过各种扫描，获取到对应的类，然后将类解析成 spring 内部的 BeanDefition 结构，存到容器中 (注入到 ConCurrentHashMap 中)，也就是最后的 beanDefinitionMap 中。

**第一步分析：**

主要方法，从 invokeBeanFactoryPostProcessors 方法一直往下跟，直到 ConfigurationClassPostProcessor 类的 parse 方法，会发现他把核心启动类传入了这个方法中。

![](https://pic1.zhimg.com/v2-6dac40a4c057c30652ebee8dd78e98d0_r.jpg)

在这个方法内部，他判断这个类上知否存在注解，如果存在继续进入下一个方法，直到真正做事的 doProcessConfigurationClass 方法，在这个方法类，他就开始处理 @ComponentScan 注解，获取到 componentScans 对象，然后调用 this.componentScanParser.parse 方法对他进行解析。

![](https://pic3.zhimg.com/v2-110be9e066c85757b0b8f4988db4e772_r.jpg)

在方法内部根据 basePackages 获取对应类全限定集合，如果集合为空，就把当前的核心启动类全限定名的包名即 com.lg 加入，设置为 basePackages(扫描的包范围)，这里就完成了第一步，获取扫描路径。

![](https://pic2.zhimg.com/v2-221356e1a3b48e21e5b7796c4734f1e1_r.jpg)

**第二步分析**

接着再跳到 doScan 方法，开始把他转成 BeanDefition 并注入 IOC 容器。

![](https://pic4.zhimg.com/v2-886203bc7497163c3b430965d3dcf78f_r.jpg)

在 doScan 方法中第一个关键点 findCandidateComponents 方法，根据传入的初始路径地址扫描该包及其子包所有的 class，并封装成 BeanDefinition 并存入一个 Set 集合中，完成第二步。

![](https://pic1.zhimg.com/v2-2913ef8d3f9de278faff8a33ad02bfd4_r.jpg)

**第三步分析**

有了 BeanDefinition 集合之后，对他进行遍历，在遍历的最后调用了一个 registerBeanDefinition 方法进行注册 BeanDefinition。

![](https://pic2.zhimg.com/v2-efdeaf3907d34324c98e3bcf04a52401_r.jpg)

在方法内部，执行到他的实现类 DefaultListableBeanFactory 中的 registerBeanDefinition 方法，就直接通过 put 方式把 BeanDefinition 注册进了 beanDefinitionMap 中。

![](https://pic4.zhimg.com/v2-aa0336560f0e349e5966b2514c8f8853_r.jpg)

**@Import 注解指定类解析**

解析完主类扫描包之后，接着又开始解析 @import 注解指定类。

![](https://pic1.zhimg.com/v2-78b8e1008e1bb87088de9e9cce56aae8_r.jpg)

首先参数里面有一个 getImports 方法，他作用就是根据 @import 注解来获取到要导入到容器中的组件类。他从核心启动类中找到对应的 @Import 注解。在内部最终要的 collectImports 方法中，进行递归调用一直找到有 @import 注解的全类名，最后返回所有有 @Import 注解的组件类。

![](https://pic1.zhimg.com/v2-d10a7b4bb6fa282af8cc5228edbba92c_r.jpg)

获取到注解组件类之后，就需要去执行组件类了，回到 ConfigurationClassParser 类的 parse 方法，执行 this.deferredImportSelectorHandler.process 方法。

![](https://pic1.zhimg.com/v2-91a2a844807619655cf374d1bc51cf7c_r.jpg)

接着往下走最后到 processGroupImports 方法内，里面有非常重要的一步 grouping.getImports()。

![](https://pic2.zhimg.com/v2-94e3df046e0ec7c271dbc389e8f3f0f5_r.jpg)

先通过 grouping.getImports() 方法里面调用了 process 方法，加载 spring.factories 文件配置所有类，一步步过滤，最后封装成 AutoConfigurationEntry 对象返回，把这些对象放入 Map<String, AnnotationMetadata> entries 集合中。

![](https://pic1.zhimg.com/v2-791dcbe38404ce499c7fe9827f7e1ba0_r.jpg)![](https://pic3.zhimg.com/v2-bf315b92ded859ce1dfa007eaec2cdf6_r.jpg)

最后通过 this.group.selectImports() 方法再进行过滤排序，返回要生效的自动装配对象全路径集合，最后通过 this.reader.loadBeanDefinitions(configClasses) 方法使这些自动装配类全部生效。

### Run 方法第六步：刷新应用上下文后的扩展接口

afterRefresh 方法，他其实就是一个扩展接口，设计模式中的模板方法，默认为空实现。如果有自定义需求，可以重写该方法。比如打印一些启动结束 log，或者一些其它后置处理。

![](https://pic1.zhimg.com/v2-f3521450557074b44d319a72a2e8b6f0_r.jpg)

### 自定义 Starter

Spring Boot 中的 starter 是一种非常重要的机制，能够抛弃以前繁杂的配置，将其统一集成进 starter，应用者只需要在 maven 中引入 starter 依赖，Spring Boot 就能自动扫描到要加载的信息并启动相应的默认配置。starter 让我们摆脱了各种依赖库的处理，需要配置各种信息的困扰。Spring Boot 会自动通过 classpath 路径下的类发现需要的 Bean，并注册进 IOC 容器。Spring Boot 提

供了针对日常企业应用研发各种场景的 spring-boot-starter 依赖模块。所有这些依赖模块都遵循着约定成俗的默认配置，并允许我们调整这些配置，即遵循 “约定大于配置” 的理念。简而言之，starter 就是一个外部的项目，我们需要使用它的时候就可以在当前 Spring Boot 项目中引入它，Spring Boot 会自动完成装配。

**使用场景**

比如动态数据源、登陆模块、AOP 日志切面等等就可以将这些功能封装成一个 starter, 复用的时候只需要在 pom.xml 引入即可，比如阿里的 Druid 数据源，就是阿里自己实现了一个第三方 starter, 因此 Spring Boot 就可以直接引入并使用这个数据库。命令规则 SpringBoot 提供的 starter 以 spring-boot-starter-xxx 的方式命名的。官方建议自定义的 starter 使用 xxx-spring-boot-starter 命名规则。以区分 SpringBoot 生态提供的 starter，比如阿里的 druid-spring-boot-starter。

**自定义 starter 代码实现：**

1. 新建 maven jar 工程，工程名为 zdy-spring-boot-starter，导入依赖

![](https://pic1.zhimg.com/v2-7251da8f274c9aa755fdb9b0a8d8b308_r.jpg)

2. 编写 JavaBean

![](https://pic2.zhimg.com/v2-54fa7e5acd2b99de10399b0381c9e239_r.jpg)

3. 编写配置类 MyAutoConfiguration

![](https://pic2.zhimg.com/v2-60ad1e7b85b1b09b1657a5a3b21176dd_r.jpg)

4. resources 下创建 / META-INF/spring.factories

![](https://pic1.zhimg.com/v2-1f146796a60dd6a35f5ac1d66f952d90_r.jpg)

**使用自定义 starter**

1. 对应项目导入自定义 starter 的依赖

![](https://pic3.zhimg.com/v2-8c2053fed56505afa7fb5331cf0981c6_r.jpg)

2. 在全局配置文件中配置属性值

![](https://pic4.zhimg.com/v2-05ac9cf467e73e800fc7c919087a4c9b_r.jpg)

### 自定义 Starter 热插拔技术

@Enablexxx 注解就是一种热拔插技术，加了这个注解就可以启动对应的 starter，当不需要对应的 starter 的时候只需要把这个注解注释掉就行。

1. 新增标记类 ConfigMarker

![](https://pic1.zhimg.com/v2-47f90b0226dcf38fd3c16967794da174_r.jpg)

2. 新增 EnableRegisterServer 注解，将 @Import 引入的组件类生成实例，添加进容器

![](https://pic2.zhimg.com/v2-24de1d301b3a2f8c4a4ebbf2871daf85_r.jpg)

3. 改造 MyAutoConfiguration 新增条件注解 @ConditionalOnBean(ConfigMarker.class) ，@ConditionalOnBean 这个是条件注解，前面的意思代表只有当期上下文中含有 ConfigMarker 对象，被标注的类才会被实例化，才能让自动配置类生效。

![](https://pic3.zhimg.com/v2-1a4a3bd1f504ddfe8f23bc47ea082dca_r.jpg)

4. 在启动类上新增上面自定义 @EnableRegisterServer 注解，根据之前的源码分析可以知道他在执行过程中会解析这个注解，再去解析里面的 @Import 注解，从而拿到组件类生成实例化对象存入容器，满足自动装配条件。

![](https://pic1.zhimg.com/v2-b679a7eabe826e02ee5f8bb4d1a81b18_r.jpg)

**关于条件注解的讲解：**

@ConditionalOnBean：仅仅在当前上下文中存在某个对象时，才会实例化一个 Bean。

@ConditionalOnClass：某个 class 位于类路径上，才会实例化一个 Bean。

@ConditionalOnExpression：当表达式为 true 的时候，才会实例化一个 Bean。基于 SpEL 表达式的条件判断。

@ConditionalOnMissingBean：仅仅在当前上下文中不存在某个对象时，才会实例化一个 Bean。

@ConditionalOnMissingClass：某个 class 类路径上不存在的时候，才会实例化一个 Bean。

@ConditionalOnNotWebApplication：不是 web 应用，才会实例化一个 Bean。

@ConditionalOnWebApplication：当项目是一个 Web 项目时进行实例化。

@ConditionalOnNotWebApplication：当项目不是一个 Web 项目时进行实例化。

@ConditionalOnProperty：当指定的属性有指定的值时进行实例化。

@ConditionalOnJava：当 JVM 版本为指定的版本范围时触发实例化。

@ConditionalOnResource：当类路径下有指定的资源时触发实例化。

@ConditionalOnJndi：在 JNDI 存在的条件下触发实例化。

@ConditionalOnSingleCandidate：当指定的 Bean 在容器中只有一个，或者有多个但是指定了首选的 Bean 时触发实例化。

### 内嵌 tomcat 原理

Spring Boot 默认支持 Tomcat，Jetty，和 Undertow 作为底层容器。而 Spring Boot 默认使用 Tomcat，一旦引入 spring-boot-starter-web 模块，就默认使用 Tomcat 容器。

通过前面的源码分析我们可以知道核心启动类在启动的时候，进入 AutoConfigurationImportSelector 类中的 getAutoConfigurationEntry 方法去各个模块 WEB-INF 下的 spring.factories 配置文件中加载相关配置类，获取到 ServletWebServerFactoryAutoConfiguration 自动配置类，也就是 tomcat 自动配置。

![](https://pic3.zhimg.com/v2-8416f57e9c3abbbfcb5f4e6b5abf13de_r.jpg)![](https://pic3.zhimg.com/v2-a4287bd4dcbdff7b00e69dc891e1959a_r.jpg)

通过 spring.factories 配置文件找到 ServletWebServerFactoryAutoConfiguration 注解类分析上面的注解。

![](https://pic4.zhimg.com/v2-fac36b97a7a3ff5ef53b2c38994b895f_r.jpg)

通过 @Import 发现他将 EmbeddedTomcat、EmbeddedJetty、EmbeddedUndertow 等嵌入式容器类加载进来了，进入 EmbeddedTomcat.class 类，通过 TomcatServletWebServerFactory 类的 getWebServer 方法，实现了 tomcat 的实例化，最后调用 getTomcatWebServer 方法进入下一步操作。

![](https://pic2.zhimg.com/v2-1d49eccbcea5106d56f9a61236363e85_r.jpg)

根据上面方法继续往下走，到了 initialize，就开始启动了 tomcat。

![](https://pic3.zhimg.com/v2-e014fb195912d262ce990fc43fe899da_r.jpg)

总结：@Import 引入的 EmbeddedTomcat 类里面 TomcatServletWebServerFactory 工厂类的 getWebServer 方法一旦被启动他就会创建并启动内嵌 tomcat。

**getWebServer 方法被调用点分析**

调用的地方其实就是之前分析过的 refresh 方法中的 onRefresh 方法

![](https://pic2.zhimg.com/v2-4f888e403b2cc12c5c7191ed53432375_r.jpg)

在方法里面进入到 ServletWebServerApplicationContext 类的 createWebServer 方法，他会获取嵌入式的 Servlet 容器工厂，并通过工厂来获取 Servlet 容器，当获取到了容器工厂之后就通过工厂来调用 getWebServer 方法，也就是上面那个，来完成内嵌 tomcat 的创建和启动。

![](https://pic1.zhimg.com/v2-d0be20f039bda01b04a5426f7995110c_r.jpg)![](https://pic2.zhimg.com/v2-2b4460b951722c9fb5a69f784f28f965_r.jpg)

### 自动配置 Spring MVC 源码分析

SpringBoot 项目里面是可以直接使用诸如 @RequestMapping 这类的 SpringMVC 的注解，在以前的项目中除了引入包之外还需要在 web.xml 配置一个前端控制器 org.springframework.web.servlet.DispatcherServlet。spring Boot 通过自动装配就实现了相同的效果, IOC 容器的注入后，最重要的一点就是需要把 DispatcherServlet 再注册进 servletContext 中，也就是 servlet 容器中。

**自动配置 DispatcherServlet 加载**

和前面 tomcat 分析一样，我们还是要先去找到 DispatcherServlet 的自动配置类

![](https://pic2.zhimg.com/v2-c8231acd9add5a2bed864d099eab1a9d_r.jpg)

接着到 META-INF/spring.factories 下，找到我们对应的 DispatcherServletAutoConfiguration 全限定名路径，进入对应注解类查看源码。

![](https://pic3.zhimg.com/v2-d688f2047200d3cf9f2a18ae2bb8853e_r.jpg)

最下面那个类点进去发现他就是 tomcat 的那个注解类，也就是先需要 tomcat 完成自动装配再来解析 DispatcherServletAutoConfiguration 类。

![](https://pic1.zhimg.com/v2-92a4a36eb8c919d094c12bbbea887e98_r.jpg)

在 DispatcherServletAutoConfiguration 类里面有两个内部类 DispatcherServletConfiguration 类和 DispatcherServletRegistrationCondition 类。

在第一个内部类中创建了 DispatcherServlet 实例化对象设值存到 IOC 容器, 但是还未添加到 servletContext。

![](https://pic4.zhimg.com/v2-665388be3d784959880a6dcd331b9c9f_r.jpg)

在第二个内部类中就判断存在 DispatcherServlet 类对象再执行，而这个类对象已经在第一个内部类已经创建存在于上下文中。接着他构建了 DispatcherServletRegistrationBean 对象，并进行了返回。由此我们得到了两个对象，第一个前端控制器 DispatcherServlet，第二个 DispatcherServletRegistrationBean 是 DispatcherServlet 的注册类，他的作用就是把他注册进 servletContext 中。

**自动配置 DispatcherServlet 注册**

![](https://pic1.zhimg.com/v2-2f1fc50606209f0662aa0cfb2c4508a0_r.jpg)

通过类图分析最上面是一个 ServletContextInitializer 接口。我们可以知道，实现该接口意味着是用来初始化 ServletContext 的。

![](https://pic2.zhimg.com/v2-1855478c6b18e13af401bf9b762fdec1_r.jpg)

进入 onStartup 方法内部，根据类图一直往下直到 ServletRegistrationBean 类中的 addRegistration 方法，发现他把 DispatcherServlet 给 add 进了 servletContext 完成了注册。

![](https://pic1.zhimg.com/v2-1dc2975352bb1ac39e66b1cb22f22208_r.jpg)

而这个 addRegistration 方法的触发跟上面 tomcat 的触发是相同地方，通过 getWebServer 方法里面的 getSelfInitializer 方法在方法内部继续调用 selfInitialize 方法，通过 getServletContextInitializerBeans 拿到所有的 ServletContextInitializer 集合，而集合中就包含了我们需要的 DispatcherServlet 控制器，接着遍历这个集合，使用遍历结果集调用 onStartup 方法，就完成了 DispatcherServlet 的注册。

![](https://pic4.zhimg.com/v2-6763b3ffa62947f6299dfdba870876a7_r.jpg)

### 三、Spring Boot 高级进阶

### Spring Boot 数据源自动配置

application.properties 文件中数据源指定的命名规则是因为底层源码设定了需要这样才能获取，注入 DataSourceProperties 类对象中。

![](https://pic4.zhimg.com/v2-cd2be429c6ac10e46bd2cce8b6725623_r.jpg)![](https://pic3.zhimg.com/v2-e9f0a93babe6ed72cd412c13c6f1517a_r.jpg)

数据源自动配置连接池默认指定 Hikari，通过指定 TYPE 就可以实现更换连接池。

![](https://pic1.zhimg.com/v2-e9d8cd047e16e5e0712bf8a13e396e88_r.jpg)

在引入 spring-boot-starter-jdbc 包后就把 HikariCP 也一并引入进来了，满足条件，而其他连接池未引入，不满足因此默认是 HikariCP 连接池。

![](https://pic4.zhimg.com/v2-ee56a796e1266521ec7340135ee81003_r.jpg)![](https://pic4.zhimg.com/v2-2cb227ea1cecc040ce5366597eadcf77_r.jpg)

如果多个连接池都满足的情况下，按照配置的数组顺序取值，第一个仍然是 HikariCP，所以在不指定的情况下，默认永远是他。

![](https://pic1.zhimg.com/v2-a242d178d6cc34a0a0930ff368b21e8c_r.jpg)

### Mybatis 自动配置源码分析

与前面的 tomcat 自动配置，DispatcherServlet 自动配置是一样的，都是通过 run 方法解析 EnableAutoConfiguration 注解，进入 AutoConfigurationImportSelector 类，然后去 WEB-INF 下的 spring.factories 配置文件中加载相关配置类，完成自动配置类组装。

![](https://pic4.zhimg.com/v2-28d7f43f426e7894dba034e37b5993d7_r.jpg)

1. 通过 application.properties 文件设置 mybatis 属性可以注解注入到对应的 MybatisProperties 类使用。

![](https://pic2.zhimg.com/v2-237804439ac6f0c8c6e7bdfa11183391_r.jpg)

2.. 这个类里面第一个方法 sqlSessionFactory，通过他实现了创建 sqlSessionFactory 类、 Configuration 类并添加容器，内部实现其实就是先解析配置，封装 Configuration 对象，完成准备工作，然后调用 MyBatis 的初始化流程。

![](https://pic1.zhimg.com/v2-f9b64da4ef9f06205a829ebca5b9ca60_r.jpg)

3. 这个类第二个方法 sqlSessionTemplate，作用是与 mapperProoxy 代理类有关。SqlSessionTemplate 是线程安全的，可以被多个 Dao 持有。

![](https://pic1.zhimg.com/v2-bc44d5e3552f0c8edc831969de8f7298_r.jpg)

4. 得到了 SqlSessionFactory 了，接下来就是如何扫描到相关的 Mapper 接口了，通过注解 @MapperScan(basePackages = “com.mybatis.mapper”) 实现，在内部 @Import 引入 MapperScannerRegistrar 类调用 registerBeanDefinitions 方法进行注册。

![](https://pic2.zhimg.com/v2-05164c305abf60fe8bf959d847558b99_r.jpg)

通俗点就是：@MapperScan(basePackages = “com.mybatis.mapper”) 这个定义，扫描指定包下的 mapper 接口，通过动态代理生成了实现类，然后设置每个 mapper 接口的 beanClass 属性为 MapperFactoryBean 类型并加入到 spring 的 bean 容器中。使用者就可以通过 @Autowired 或者 getBean 等方式，从 spring 容器中获取。

### SpringBoot + Mybatis 实现动态数据源切换

**业务背景**

电商订单项目分正向和逆向两个部分：其中正向数据库记录了订单的基本信息，包括订单基本信息、订单商品信息、优惠卷信息、发票信息、账期信息、结算信息、订单备注信息、收货人信息等；逆向数据库主要包含了商品的退货信息和维修信息。数据量超过 500 万行就要考虑分库分表和读写分离，那么我们在正向操作和逆向操作的时候，就需要动态的切换到相应的数据库，进行相关的操作。

**解决思路**

现在项目的结构设计基本上是基于 MVC 的，那么数据库的操作集中在 dao 层完成，主要业务逻辑在 service 层处理，controller 层处理请求。假设在执行 dao 层代码之前能够将数据源（DataSource）换成我们想要执行操作的数据源，那么这个问题就解决了。

![](https://pic1.zhimg.com/v2-69960208ca588f7fb7d2b393df67b74c_r.jpg)

**实现原理**

Spring 内置了一个 AbstractRoutingDataSource 抽象类，它可以把多个数据源配置成一个 Map，然后，根据不同的 key 返回不同的数据源。因为 AbstractRoutingDataSource 也是一个 DataSource 接口，因此，应用程序可以先设置好 key， 访问数据库的代码就可以从 AbstractRoutingDataSource 拿到对应的一个真实的数据源，从而访问指定的数据库。

![](https://pic3.zhimg.com/v2-52a2bb88a8fedd8bd309924254b7d0be_r.jpg)

通过内部的 getConnection 方法获取数据源，在连接数据库之前会执行 determineCurrentLookupKey() 方法，这个方法返回的数据作为 key 去 targetDataSources 中查找相应的值，如果查到相对应的 DataSource，那么就使用此 DataSource 获取数据库连接。

![](https://pic4.zhimg.com/v2-d29c899e7638e1b9fbddb21739bf744f_r.jpg)![](https://pic2.zhimg.com/v2-94e9179911eaf0bd2e82719d1ed49d89_r.jpg)

**环境准备工作**

实体类：

![](https://pic2.zhimg.com/v2-634cd2cb8b24b5be8cf956129c29d421_r.jpg)

Mapper 类：

![](https://pic2.zhimg.com/v2-99dbac12251ffcb84e15f36a0750f181_r.jpg)

Service 类

![](https://pic2.zhimg.com/v2-8bd615ec6abadf5d8980c085a6075649_r.jpg)

Controller 类：

![](https://pic2.zhimg.com/v2-3c06ef257590dc41bad4cffba2362675_r.jpg)

**具体实现步骤**

多数据源配置：

![](https://pic4.zhimg.com/v2-daa0d37e4aed0bdda47205a12d19104f_r.jpg)

读取初始数据源类：

![](https://pic1.zhimg.com/v2-74b24e8335e737a9ee695214320e0b14_r.jpg)

创造一个 RoutingDataSourceContext 类指定数据源

![](https://pic2.zhimg.com/v2-9e8d75d0c68d69f7326266879f8458f1_r.jpg)

创建一个继承 AbstractRoutingDataSource 类的 RoutingDataSource 类，重写 determineCurrentLookupKey 方法

![](https://pic4.zhimg.com/v2-ab21c289eb47ea2fbb09a825f26577f3_r.jpg)

创建一个 primaryDataSource 方法，引入两个数据源 bean 对象, 获取数据源，设置 key 存入继承类 RoutingDataSource 中。

![](https://pic4.zhimg.com/v2-d6c95e2b23a50d0d58481525601040e7_r.jpg)

改造 Controller 类，在具体业务逻辑执行前，进行数据源绑定确认。

![](https://pic2.zhimg.com/v2-9cbea00fd757354a4fc181c7a60ef495_r.jpg)

测试结果，分别调用两个方法，返回了不同数据库的结果

![](https://pic2.zhimg.com/v2-04565ee1a9c48494f03c79f530ff55a1_r.jpg)

**动态数据源优化**

需要读数据库的地方，在对应 controller 类方法里面就需要加上一大段 RoutingDataSourceContext routingDataSourceContext = new RoutingDataSourceContext(key); 代码，这里可以通过自定义注解 @RoutingWith("slaveDataSource")，来实现这个效果，取消大量重复代码编写。

新增 RoutingWith 注解类：

![](https://pic3.zhimg.com/v2-11b5dd5cd8864de9df3bc058cda887a6_r.jpg)

借助 AOP 的动态代理对方法拦截实现横切逻辑增强，添加 Maven 依赖

![](https://pic1.zhimg.com/v2-db9d37638ce464cbcf6736cdeadefc9c_r.jpg)

切面类 RoutingAspect

![](https://pic2.zhimg.com/v2-2df1b16817301255f0c7ce15659d7ae9_r.jpg)

自定义注解定义完毕，把 controller 类改造成注解方式就完成了优化改造。

### 四、Spring Boot 缓存深入

### JSR107

关于如何使用缓存的规范，是 java 提供的一个接口规范，类似于 JDBC 规范。

五个核心接口：

**CachingProvider**（缓存提供者）：创建、配置、获取、管理和控制多个 CacheManager。

**CacheManager**（缓存管理器）：创建、配置、获取、管理和控制多个唯一命名的 Cache，Cache 存在于 CacheManager 的上下文中。一个 CacheManager 仅对应一个 CachingProvider。

**Cache**（缓存）：是由 CacheManager 管理的，CacheManager 管理 Cache 的生命周期，Cache 存在于 CacheManager 的上下文中，是一个类似 map 的数据结构，并临时存储以 key 为索引的值。一个 Cache 仅被一个 CacheManager 所拥有

**Entry**（缓存键值对）：是一个存储在 Cache 中的 key-value 对。

**Expiry**（缓存时效）：每一个存储在 Cache 中的条目都有一个定义的有效期。一旦超过这个时间，条目就自动过期，过期后，条目将不可以访问、更新和删除操作。缓存有效期可以通过 ExpiryPolicy 设置。

![](https://pic1.zhimg.com/v2-09cd0ccbc5df718f26b663aa4893be38_r.jpg)

一个应用里面可以有多个缓存提供者 (CachingProvider)，一个缓存提供者可以获取到多个缓存管理器 (CacheManager)，一个缓存管理器管理着不同的缓存 (Cache)，缓存中是一个个的缓存键值对 (Entry)，每个 entry 都有一个有效期 (Expiry)。缓存管理器和缓存之间的关系有点类似于数据库中连接池和连接的关系。

### 缓存概念及缓存注解

Spring 从 3.1 开始定义了 org.springframework.cache.Cache 和 org.springframework.cache.CacheManager 接口来统一不同的缓存技术；并支持使用 JavaCaching（JSR-107）注解简化我们进行缓存开发。

Spring Cache 只负责维护抽象层，具体的实现由自己的技术选型来决定。将缓存处理和缓存技术解除耦合。

每次调用需要缓存功能的方法时，Spring 会检查指定参数的指定的目标方法是否已经被调用过，如果有就直接从缓存中获取方法调用后的结果，如果没有就调用方法并缓存结果后返回给用户。下次调用直接从缓存中获取。

两个接口：

Cache：缓存抽象的规范接口，缓存实现有：RedisCache、EhCache、ConcurrentMapCache 等。

CacheManager：缓存管理器，管理 Cache 的生命周期。

![](https://pic2.zhimg.com/v2-305c479dacd750a6e1d544488c925921_r.jpg)

1.@Cacheable 标注在方法上，表示该方法的结果需要被缓存起来，缓存的键由 keyGenerator 的策略决定，缓存的值的形式则由 serialize 序列化策略决定 (序列化还是 json 格式)；标注上该注解之后，在缓存时效内再次调用该方法时将不会调用方法本身而是直接从缓存获取结果。

2.@CachePut 也标注在方法上，和 @Cacheable 相似也会将方法的返回值缓存起来，不同的是标注 @CachePut 的方法每次都会被调用，而且每次都会将结果缓存起来，适用于对象的更新。

### 缓存注解 @Cacheable 实现

1. 在核心启动类开启基于注解的缓存功能：主启动类标注 @EnableCaching

![](https://pic3.zhimg.com/v2-5a21de1d1235398d2c7e2eacb6282ff2_r.jpg)

2. 标注缓存相关注解：@Cacheable、CacheEvict、CachePut

@Cacheable 开启缓存查询 会将查询出来的值存到缓存中

* value/cacheNames 指定缓存的名称，cacheManager 是管理多个 cache, 以名称进行区分

* key: 缓存数据时指定 key 值，(key,value),value 就是查询的结果，key 默认方法的参数值，也可以去使用 spEl 去计 key 值

* keyGenerator:key 的生成策略，和 key 二选一，通过他可以自定义 keyGenerator

* cacheManager：指定缓存管理器，比如 redis 和 ehcache

* cacheResolver: 功能和 cacheManager 相同 二选一

* condition：条件属性，必须要满足这个条件才会进行缓存

*unless: 否定条件，满足这个条件不进行缓存

*sync : 是否使用异步模式进行缓存

![](https://pic2.zhimg.com/v2-a677117aae8d0fb0a22378d7c286d239_r.jpg)

注意：既满足 condition 又满足 unless 条件的也不进行缓存；使用异步模式进行缓存时 (sync=true)：unless 条件将不被支持

可用的 SpEL 表达式：

![](https://pic3.zhimg.com/v2-2f23200dcd4e2532a11df843b5c5e2ae_r.jpg)

### 自动配置缓存实现源码分析

Spring Boot 中所有的自动配置都是基于 AutoConfigurationImportSelector 类，和前面的自动配置一样，都是在 getAutoConfigurationEntry 方法中在 WEB-INF 下的 spring.factories 配置文件中加载相关配置类，找到 org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration，这就是对应的缓存配置类。

![](https://pic2.zhimg.com/v2-393bc0aca366793e64a891cbf8c58775_r.jpg)

进入 CacheAutoConfiguration 缓存配置类。

![](https://pic3.zhimg.com/v2-52fa3afa1f920aed20d176a04ae1ea7e_r.jpg)

进入引入的组件类 CacheConfigurationImportSelector，发现他实现了 ImportSelector 接口重写了 selectImports 方法，返回了所有的缓存组件类数组。这些加载顺序是自上而下的。

![](https://pic4.zhimg.com/v2-13cb8f60c6eff8c2d52bd42a9883473f_r.jpg)

打开 RedisCacheConfiguration 组件类，通过上面的注解条件分析，在没有引入 redis 的情况下，这些组件类是不会生效的。

![](https://pic4.zhimg.com/v2-8b0652661067fdd5c4897901ade8faaf_r.jpg)

而 SimpleCacheConfiguration 类里面的注解条件满足，因此默认的缓存组件类就是 SimpleCacheConfiguration。

![](https://pic3.zhimg.com/v2-7e295458d1b45bdfa8d48b261cd7a036_r.jpg)

分析源码得到通过 cacheManager 方法，会创建 ConcurrentMapCacheManager 对象并返回，在这个对象类里面实现了 CacheManager 接口，在里面有一个 getCache 方法，通过双重校验锁机制，先从缓存中取缓存对象，如果有直接返回，如果没得，就创建，并且添加到 cacheMap，也就是缓存池中。而我们的这个 cacheMap 他的底层结构就是通过 createConcurrentMapCache 方法创建并返回的 Cache，里面定义了 Cache 的属性以及操作 Cache 的方法，比如 lookup、put 方法，在执行查询的时候，他会先调用 lookup 方法查询是否有缓存结果，如果没有查询数据库，就把结果拿到再调用 put 方法存入缓存。

![](https://pic3.zhimg.com/v2-de6c983050e5d3b4b9bfa731aab872aa_r.jpg)![](https://pic2.zhimg.com/v2-08ae46c53da3da717ed62173a7d1abd1_r.jpg)

### @CachePut&@CacheEvict&@CacheConfig

**@CachePut**

既调用方法，又更新缓存数据，一般用于更新操作，在更新缓存时**一定要和想更新的缓存有相同的缓存名称和相同的 key**(可类比同一张表的同一条数据)。

1. 先调用目标方法

2. 将目标方法的结果缓存起来

![](https://pic1.zhimg.com/v2-8f478f762c2b7c0fa5d36976a05c3de8_r.jpg)

**@CacheEvict**

缓存清除，清除缓存时要指明缓存的名字和 key，相当于告诉数据库要删除哪个表中的哪条数据，key 默认为参数的值

**value/cacheNames**：缓存的名字

**key**：缓存的键

**allEntries**：是否清除指定缓存中的所有键值对，默认为 false，设置为 true 时会清除缓存中的所有键值对，与 key 属性二选一使用，就是是否清除所有。

**beforeInvocation**：在 @CacheEvict 注解的方法调用之前清除指定缓存，默认为 false，即在方法调用之后清除缓存，设置为 true 时则会在方法调用之前清除缓存 (在方法调用之前还是之后清除缓存的区别在于方法调用时是否会出现异常，若不出现异常，这两种设置没有区别，若出现异常，设置为在方法调用之后清除缓存将不起作用，因为方法调用失败了)。

**@CacheConfig**

标注在类上，抽取缓存相关注解的公共配置，可抽取的公共配置有缓存名字、主键生成器等 (如注解中的属性所示)

就是把下面方法上的注解抽取上来统一配置，避免繁琐的重复配置代码。

![](https://pic3.zhimg.com/v2-618adf4f755b8c91b3dbeff1419beb4e_r.jpg)

### 基于 Redis 的缓存实现

SpringBoot 默认开启的缓存管理器是 ConcurrentMapCacheManager，创建缓存组件是 ConcurrentMapCache，将缓存数据保存在一个个的 ConcurrentHashMap<Object, Object> 中。开发时我们可以使用缓存中间件：redis、memcache、ehcache 等，这些缓存中间件的启用很简单——只要向容器中加入相关的 bean 就会启用，可以启用多个缓存中间件。

1. 引入 Redis 的 starter

![](https://pic1.zhimg.com/v2-5d18af624ef9091d27ea395813be514c_r.jpg)

引入相关 Bean 之后，Spring Boot 的自动装配就会在初始化时，找到 Redis 自定义的 RedisAutoConfiguration 进行装配，里面有两个返回类型，RedisTemplate 和 StringRedisTemplate(用来操作字符串：key 和 value 都是字符串)，template 中封装了操作各种数据类型的操作 (stringRredisTemplate.opsForValue()、stringRredisTemplate.opsForList() 等)。

![](https://pic2.zhimg.com/v2-9189812aa4917e8aadc073cd9c910681_r.jpg)

2. 配置 redis：只需要配置 redis 的主机地址 spring.redis.host=127.0.0.1，

注意：如何要使用 Redis 缓存数据，对应操作的实体类一定要实现 Serializable 接口。

在前面自定义配置缓存上说了缓存的九大组件，并且自上而下加载顺序，当 Redis 被引入之后，他的条件先被满足完成自动装配并创建了 CacheManager 类对象，那么下面的 SimpleCacheConfiguration 类，因为 CacheManager.class 已经存在，就不会生效。

![](https://pic4.zhimg.com/v2-13cb8f60c6eff8c2d52bd42a9883473f_r.jpg)![](https://pic2.zhimg.com/v2-e1182da66cbfd7ad99f78140fc5e2b0d_r.jpg)

**自定义 RedisCacheManager**

SpringBoot 默认采用的是 JDK 的对象序列化方式，我们可以切换为使用 JSON 格式进行对象的序列化操作，这时需要我们自定义序列化规则。

![](https://pic2.zhimg.com/v2-e45dddb579bc96a05bd020e1772164ed_r.jpg)

RedisConfig 配置类中使用 @Bean 注解注入了一个默认名称为方法名的 cacheManager 组件。在定义的 Bean 组件中，通过 RedisCacheConfiguration 对缓存数据的 key 和 value 分别进行了序列化方式的定制，其中缓存数据的 key 定制为 StringRedisSerializer（即 String 格式），而 value 定制为了 Jackson2JsonRedisSerializer（即 JSON 格式），同时还使用 entryTtl(Duration.ofDays(1)) 方法将缓存数据有效期设置为 1 天完成基于注解的 Redis 缓存管理器 RedisCacheManager 定制后，可以对该缓存管理器的效果进行测试（使用自定义序列化机制的 RedisCacheManager 测试时，实体类可以不用实现序列化接口）

### 五、Spring Boot 部署与监控

### JAR 包部署

1. 项目打包

![](https://pic1.zhimg.com/v2-55e97a8fc559350fa73f4c710a14eb6c_r.jpg)

2. 项目启动

![](https://pic3.zhimg.com/v2-88231afdb96f1cdb515e5084f7a60dba_r.jpg)

### WAR 包部署

1. 修改 pom.xml 配置

![](https://pic3.zhimg.com/v2-2e5765e5ec954a126b611c35fc980cfa_r.jpg)

2. 添加依赖

![](https://pic3.zhimg.com/v2-4f1b6ceb210ab203c1144f5865143c66_r.jpg)

3. 排除 Spring Boot 内置 Tomcat

![](https://pic4.zhimg.com/v2-3d31b3f5965fe29ee2b8f2f469bf5de3_r.jpg)

4. 改造启动类

![](https://pic2.zhimg.com/v2-410531b9da17c0889ce8affc2af1f411_r.jpg)

5. 打包交给外置 Tomcat 运行

**JAR 包和 WAR 包部署差异：**

1.jar 更加简单方便，使用 java -jar xx.jar 就可以启动。所以打成 jar 包的最多。而 war 包可以部署到 tomcat 的 webapps 中，随 Tomcat 的启动而启动。具体使用哪种方式，应视应用场景而定。

2、打 jar 包时不会把 src/main/webapp 下的内容打到 jar 包里 (你认为的打到 jar 包里面，路径是不行的会报 404) 打 war 包时会把 src/main/webapp 下的内容打到 war 包里。

3. 打成什么文件包进行部署与项目业务有关，就像提供 rest 服务的项目需要打包成 jar 文件，用命令运行很方便。。。而有大量 css、js、html，且需要经常改动的项目，打成 war 包去运行比较方便，因为改动静态资源可以直接覆盖，很快看到改动后的效果，这是 jar 包不能比的。

### 多环境部署

线上环境 prod(product)、开发环境 dev(development)、测试环境 test、提测环境 qa、单元测试 unitest 等等多种环境进行不同配置。

1. 多环境配置文件

![](https://pic3.zhimg.com/v2-7aa328790a245530d948f4cd8fbc5772_r.jpg)

2. 指定要加载的配置文件

![](https://pic1.zhimg.com/v2-b3c3939dfc42f82e9209e504bd45897c_r.jpg)

### 监控插件 - Acturator

Spring boot 作为微服务框架，除了它强大的快速开发功能外，还有就是它提供了 actuator 模块，引入该模块能够自动为 Spring boot 应用提供一系列用于监控的端点。Spring Boot Actuator 提供了对单个 Spring Boot 的监控，信息包含：应用状态、内存、线程、堆栈等等，比较全面的监控了 Spring Boot 应用的整个生命周期。

**Actuator 的 REST 接口**

Actuator 监控分成两类：原生端点和用户自定义端点；自定义端点主要是指扩展性，用户可以根据自己的实际应用，定义一些比较关心的指标，在运行期进行监控。

原生端点是在应用程序里提供众多 Web 接口，通过它们了解应用程序运行时的内部状况。

原生端点又可以分成三类：

应用配置类：可以查看应用在运行期的静态信息：例如自动配置信息、加载的 springbean 信息、yml 文件配置信息、环境信息、请求映射信息；

度量指标类：主要是运行期的动态信息，例如堆栈、请求链、一些健康指标、metrics 信息等；

操作控制类：主要是指 shutdown, 用户可以发送一个请求将应用的监控功能关闭。

原生端点十三个接口：

![](https://pic4.zhimg.com/v2-da7e546f75383ef2bdb54cf1acb92553_r.jpg)![](https://pic4.zhimg.com/v2-d242ae69aad92775c0d7e5206a7d88a3_r.jpg)

**引入依赖**

![](https://pic3.zhimg.com/v2-6b76cf9ec0e8b63b237bfd4e21a5a27a_r.jpg)

注意：保证 actuator 暴露的监控接口的安全性，需要添加安全控制的依赖 spring-boot-startsecurity 依赖，访问应用监控端点时，都需要输入验证信息。

**属性详解**

在 Spring Boot 2.x 中为了安全期间，Actuator 只开放了两个端点 /actuator/health 和 / actuator/info。

开启所有的监控点，也可部分：

![](https://pic4.zhimg.com/v2-96dec4fc0e501bad7ce0daa3d273d91f_r.jpg)

**Health：**

health 主要用来检查应用的运行状态，这是我们使用最高频的一个监控点。通常使用此接口提醒我们应用实例的运行状态，以及应用不” 健康 “的原因，比如数据库连接、磁盘空间不够等，UP 表示程序健康运行。

![](https://pic2.zhimg.com/v2-87308799b45bfaadc74e6e91be5add7d_r.jpg)

**Info：**

info 就是我们自己配置在配置文件中以 info 开头的配置信息。

**Beans：**

展示了 bean 的别名、类型、是否单例、类的地址、依赖等信息。

**Conditions：**

Spring Boot 的自动配置功能非常便利，但有时候也意味着出问题比较难找出具体的原因。使用 conditions 可以在应用运行时查看代码了某个配置在什么条件下生效，或者某个自动配置为什么没有生效。

**Heapdump：**

返回一个 GZip 压缩的 JVM 堆 dump，我们可以使用 JDK 自带的 Jvm 监控工具 VisualVM 打开此文件查看内存快照。

**Mappings：**

描述全部的 URI 路径，以及它们和控制器的映射关系。

**Threaddump：**

生成当前线程活动的快照。这个功能非常好，方便我们在日常定位问题的时候查看线程的情况。 主要展示了线程名、线程 ID、线程的状态、是否等待锁资源等信息。

**Shutdown：**

开启接口优雅关闭 Spring Boot 应用，要使用这个功能首先需要在配置文件中开启。

management.endpoint.shutdown.enabled=true

### 监控插件 - Spring Boot Admin

Spring Boot Admin 是一个针对 spring-boot 的 actuator 接口进行 UI 美化封装的监控工具。他可以返回在列表中浏览所有被监控 spring-boot 项目的基本信息比如：Spring 容器管理的所有的 bean、详细的 Health 信息、内存信息、JVM 信息、垃圾回收信息、各种配置信息（比如数据源、缓存列表和命中率）等，Threads 线程管理，Environment 管理等。

![](https://pic3.zhimg.com/v2-25e9e6779217208d80bffead9d324eb6_r.jpg)

1. 搭建 Server 端

![](https://pic1.zhimg.com/v2-b7e60b6e1c051a366a8c392960c7bfe0_r.jpg)

2. 搭建 client 端

![](https://pic2.zhimg.com/v2-1ba001d1060194680fa5771c9f539655_r.jpg)![](https://pic3.zhimg.com/v2-4c798b18a7ff06cae11418ef4deea52a_r.jpg)

3. 通过地址 http://localhost:8080/applications 测试结果

![](https://pic1.zhimg.com/v2-3d5bb28c26a6060deb12219abbc528ac_r.jpg)