### Mybatis

#### 简介

MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

#### 使用

Mybatis的核心是SqlSessionFactory，SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先配置的 Configuration 实例来构建出 SqlSessionFactory 实例。

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

XML文件中包含Mybatis的核心配置，例如数据源（DataSource）和事务管理器（TransactionManager）

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
</configuration>
```

也可以通过Java代码配置Mybatis

```java
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

**通过SqlSessionFactory开启SqlSession**

SqlSession中提供了执行SQL的方法

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
}
```

#### XML文件

**SQL 映射文件顶级元素**

- `cache` – 该命名空间的缓存配置。
- `cache-ref` – 引用其它命名空间的缓存配置。
- `resultMap` – 描述如何从数据库结果集中加载对象，是最复杂也是最强大的元素。
- ~~`parameterMap` – 老式风格的参数映射。此元素已被废弃，并可能在将来被移除！请使用行内参数映射。~~
- `sql` – 可被其它语句引用的可重用语句块。
- `insert` – 映射插入语句。
- `update` – 映射更新语句。
- `delete` – 映射删除语句。
- `select` – 映射查询语句。

**动态SQL**

- if：根据条件拼接SQL。
- choose (when, otherwise)：不想使用所有的条件，而只是想从多个条件中选择一个使用。
- trim (where, set)：*where* 元素只会在子元素返回任何内容的情况下才插入 “WHERE” 子句。而且，若子句的开头为 “AND” 或 “OR”，*where* 元素也会将它们去除。
- foreach：对集合进行遍历（比如在构建 IN 条件语句的时候）

**命名空间（Namespaces）**

Mybatis的mapper映射文件通过命名空间来区分不同文件中拥有相同id的sql语句，防止冲突。只要确保同一命名空间中的id唯一就可以。

#### 原理

mybatis工作原理图

![mybatis-1](https://gitee.com/LoopSup/image/raw/master/img/mybatis-1.png)

1. 通过SqlSessionFactoryBuilder从 mybatis-config.xml 配置文件（也可以用 Java 文件配置的方式，需要添加 @Configuration）中构建出 SqlSessionFactory，主要是获取DataSource

```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

`SqlSessionFactoryBuilder().build(inputStream)`最终是通过`org.apache.ibatis.builder.xml.XMLConfigBuilder#parse`方法解析全局配置文件和Mapper映射文件

```java
public Configuration parse() {
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    ////解析全局配置文件
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}

private void parseConfiguration(XNode root) {
    try {
        // issue #117 read properties first
        propertiesElement(root.evalNode("properties"));
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        loadCustomLogImpl(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        //解析映射文件
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

查看Mapper映射文件解析代码，最终走到 `org.apache.ibatis.builder.xml.XMLMapperBuilder#parse`

```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
        //解析映射文件的根节点mapper元素
        //将mapper文件中的元素信息，比如insert、select这等信息解析到MappedStatement对象，并保存到Configuration类中的mappedStatements属性中，以便于后续动态代理类执行CRUD操作时能够获取真正的Sql语句信息
        configurationElement(parser.evalNode("/mapper"));
        configuration.addLoadedResource(resource);
        //这个方法内部会根据namespace属性值，生成动态代理类
        bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
}
```

`bindMapperForNamespace()`方法根据mapper文件中的namespace属性值，为接口生成动态代理类

```java
private void bindMapperForNamespace() {
    //获取mapper元素的namespace属性值
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        Class<?> boundType = null;
        try {
            //获取namespace属性值对应的Class对象
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
            // ignore, bound type is not required
            //如果没有这个类，则直接忽略，这是因为namespace属性值只需要唯一即可，并不一定对应一个XXXMapper接口
            //没有XXXMapper接口的时候，我们可以直接使用SqlSession来进行增删改查
        }
        if (boundType != null && !configuration.hasMapper(boundType)) {
            // Spring may not know the real resource name so we set a flag
            // to prevent loading again this resource from the mapper interface
            // look at MapperAnnotationBuilder#loadXmlResource
            configuration.addLoadedResource("namespace:" + namespace);
            //如果namespace属性值有对应的Java类，调用Configuration的addMapper方法，将其添加到MapperRegistry中
            configuration.addMapper(boundType);
        }
    }
}
```

Configuration类里面通过MapperRegistry对象维护了所有要生成动态代理类的XxxMapper接口信息

```java
public class Configuration {
    ...
    protected MapperRegistry mapperRegistry = new MapperRegistry(this);
    ...
    //mybatis在解析配置文件时，会将需要生成动态代理类的接口注册到其中
    public <T> void addMapper(Class<T> type) {
        mapperRegistry.addMapper(type);
    }
    //用于创建接口的动态类
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
    }
    ...
}
```

Configuration将addMapper方法委托给MapperRegistry的addMapper方法

```java
public <T> void addMapper(Class<T> type) {
    //这个class必须是一个接口，因为是使用JDK动态代理，所以需要是接口，否则不会针对其生成动态代理
    if (type.isInterface()) {
        if (hasMapper(type)) {
            throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }
        boolean loadCompleted = false;
        try {
            //生成一个MapperProxyFactory，用于之后生成动态代理类
            knownMappers.put(type, new MapperProxyFactory<>(type));
            // It's important that the type is added before the parser is run
            // otherwise the binding may automatically be attempted by the
            // mapper parser. If the type is already known, it won't try.
            //解析我们定义的XxxMapper接口里面使用的注解
            MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
            parser.parse();
            loadCompleted = true;
        } finally {
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}
```

MapperRegistry内部维护一个映射关系，每个接口对应一个MapperProxyFactory（生成动态代理工厂类）,便于在后面调用MapperRegistry的getMapper()时，直接从Map中获取某个接口对应的动态代理工厂类，然后再利用工厂类针对其接口生成真正的动态代理类。

2. SqlSessionFactory开启SqlSession，主要代码如下：

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        //获取Environment
        final Environment environment = configuration.getEnvironment();
        //从Environment中获取TransactionFactory
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        //在数据库连接上创建事务Transaction
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        //创建Executor对象
        final Executor executor = configuration.newExecutor(tx, execType);
        //创建sqlsession对象
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        closeTransaction(tx); // may have fetched a connection so lets call close()
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```
创建Executor对象
```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {//批量执行器，用于执行批量sql操作
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {//重用statement执行sql操作
        executor = new ReuseExecutor(this, transaction);
    } else {//简单执行sql
        executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {//缓存执行器，在查询数据库前先查找缓存，若没找到的话调用delegate（就是构造时传入的Executor对象）从数据库查询，并将查询结果存入缓存中
        executor = new CachingExecutor(executor);
    }
    //Executor对象是可以被插件拦截的，如果定义了针对Executor类型的插件，最终生成的Executor对象是被各个插件插入后的代理对象。
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```
Executor有三种不同的实现：

**SimpleExecutor** ：每执⾏⼀次  update 或  select，就开启⼀个  Statement 对象，⽤完⽴刻关闭Statement 对象。
**ReuseExecutor** ：执⾏  update 或  select，以  sql 作为  key 查找  Statement 对象，存在就使⽤，不存在就创建，⽤完后，不关闭  Statement 对象，⽽是放置于  Map<String, Statement>内，供下⼀次使⽤。简⾔之，就是重复使⽤  Statement 对象。
**BatchExecutor** ：执⾏  update（没有  select，JDBC 批处理不⽀持  select），将所有  sql 都添加到批处理中（addBatch()），等待统⼀执⾏（executeBatch()），它缓存了多个  Statement 对象，每个  Statement 对象都是addBatch()完毕后，等待逐⼀执⾏  executeBatch()批处理。与  JDBC 批处理相同。
**作⽤范围**：Executor 的这些特点，都严格限制在  SqlSession ⽣命周期范围内。

选择SimpleExecutor进行查看

```java
public class SimpleExecutor extends BaseExecutor {
    public SimpleExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Statement stmt = null;
        try {
            //获取配置属性
            Configuration configuration = ms.getConfiguration();
            //获取StatementHandler
            StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
            //获取prepareStatement
            stmt = prepareStatement(handler, ms.getStatementLog());
            //使用prepareStatement执行sql
            return handler.update(stmt);
        } finally {
            closeStatement(stmt);
        }
    }
    ......
}
```

创建prepareStatement对象，是通过RoutingStatementHandler获取具体的StatementHandler，包括SimpleStatementHandler、PreparedStatementHandler和CallableStatementHandler三种。

StatementHandler是可以被拦截器拦截的，和Executor一样，被拦截器拦截后的对像是一个代理对象。

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

3. 通过SqlSession 实例获得 Mapper 对象并运行 Mapper 映射的 SQL 语句，完成对数据库的 CRUD 和事务提交，之后关闭 SqlSession。代码如下：

```java
//org.apache.ibatis.binding.MapperRegistry#getMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}

//MapperProxyFactory
@SuppressWarnings("unchecked")
protected T newInstance(MapperProxy<T> mapperProxy) {
    //可以看到，Mybatis最终通过动态代理获取Mapper对象
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}
```

通过MapperProxy执行，mapperProxy是InvocationHandler的子类

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else {
            return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}

private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
    try {
        // A workaround for https://bugs.openjdk.java.net/browse/JDK-8161372
        // It should be removed once the fix is backported to Java 8 or
        // MyBatis drops Java 8 support. See gh-1929
        MapperMethodInvoker invoker = methodCache.get(method);
        if (invoker != null) {
            return invoker;
        }

        return methodCache.computeIfAbsent(method, m -> {
            if (m.isDefault()) {
                try {
                    if (privateLookupInMethod == null) {
                        return new DefaultMethodInvoker(getMethodHandleJava8(method));
                    } else {
                        return new DefaultMethodInvoker(getMethodHandleJava9(method));
                    }
                } catch (IllegalAccessException | InstantiationException | InvocationTargetException
                         | NoSuchMethodException e) {
                    throw new RuntimeException(e);
                }
            } else {
                return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
            }
        });
    } catch (RuntimeException re) {
        Throwable cause = re.getCause();
        throw cause == null ? re : cause;
    }
}
```

MapperProxy将执行权交给MapperMethod，MapperMethod根据参数和返回值类型选择不同的SqlSession方法执行

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
        case INSERT: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
            } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
                if (method.returnsOptional()
                    && (result == null || !method.getReturnType().equals(result.getClass()))) {
                    result = Optional.ofNullable(result);
                }
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
        throw new BindingException("Mapper method '" + command.getName()
                                   + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
}
```

**SqlSessionFactoryBuilder**

这个类可以被实例化、使用和丢弃，一旦创建了 SqlSessionFactory，就不再需要它了。 因此 SqlSessionFactoryBuilder 实例的最佳作用域是方法作用域（也就是局部方法变量）。 你可以重用 SqlSessionFactoryBuilder 来创建多个 SqlSessionFactory 实例，但最好还是不要一直保留着它，以保证所有的 XML 解析资源可以被释放给更重要的事情。

**SqlSessionFactory**

SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例。 使用 SqlSessionFactory 的最佳实践是在应用运行期间不要重复创建多次，多次重建 SqlSessionFactory 被视为一种代码“坏习惯”。因此 SqlSessionFactory 的最佳作用域是应用作用域。 有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式。

**SqlSession**

每个线程都应该有它自己的 SqlSession 实例。SqlSession 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。 绝对不能将 SqlSession 实例的引用放在一个类的静态域，甚至一个类的实例变量也不行。 也绝不能将 SqlSession 实例的引用放在任何类型的托管作用域中，比如 Servlet 框架中的 HttpSession。 如果你现在正在使用一种 Web 框架，考虑将 SqlSession 放在一个和 HTTP 请求相似的作用域中。 换句话说，每次收到 HTTP 请求，就可以打开一个 SqlSession，返回一个响应后，就关闭它。 这个关闭操作很重要，为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 finally 块中。 

**类型别名（typeAliases）**

类型别名可为 Java 类型设置一个缩写名字。 仅用于 XML 配置，意在降低冗余的全限定类名书写。

```xml
<typeAliases>
  <typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
  <typeAlias alias="Comment" type="domain.blog.Comment"/>
  <typeAlias alias="Post" type="domain.blog.Post"/>
  <typeAlias alias="Section" type="domain.blog.Section"/>
  <typeAlias alias="Tag" type="domain.blog.Tag"/>
</typeAliases>
```

#### 插件（plugins）

MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
- ParameterHandler (getParameterObject, setParameters)
- ResultSetHandler (handleResultSets, handleOutputParameters)
- StatementHandler (prepare, parameterize, batch, update, query)

实现 Interceptor 接口，并指定想要拦截的方法签名：

```java
// ExamplePlugin.java
@Intercepts({@Signature(
  type= Executor.class,
  method = "update",
  args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
  private Properties properties = new Properties();
  public Object intercept(Invocation invocation) throws Throwable {
    // implement pre processing if need
    Object returnObject = invocation.proceed();
    // implement post processing if need
    return returnObject;
  }
  public void setProperties(Properties properties) {
    this.properties = properties;
  }
}
```

```xml
<!-- mybatis-config.xml -->
<plugins>
  <plugin interceptor="org.mybatis.example.ExamplePlugin">
    <property name="someProperty" value="100"/>
  </plugin>
</plugins>
```

除了用插件来修改 MyBatis 核心行为以外，还可以通过完全覆盖配置类来达到目的。只需继承配置类后覆盖其中的某个方法，再把它传递到 SqlSessionFactoryBuilder.build(myConfig) 方法即可。


