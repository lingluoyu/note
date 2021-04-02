#### 流程

- 实现 `org.apache.ibatis.plugin.Interceptor` 接口
- 实现类添加注解`@Intercepts({@Signature(type, method, args)})`

  - **type**：需要拦截的对象，可以选择以下四种对象

    - **Executor** 是SQL执行器，包含执行SQL的全过程，组装参数，组装结果集到返回值以及执行SQL的过程等都可以拦截。
    - **StatementHandler** 处理SQL的执行过程，拦截此对象可以重写SQL执行的过程。
    - **ParameterHandler** 处理传入**SQL**的参数，拦截此对象可以重写参数的处理规则。
    - **ResultSetHandler** 处理结果集，拦截此对象可以重写结果集的组装规则。
  - **method**：需要拦截的方法
  
  - **args**：拦截的方法的参数
- 在实现类中实现拦截的方法`Object intercept(Invocation invocation)`

#### Interceptor接口

```java
public interface Interceptor {
	//通过实现此方法覆盖被拦截对象的原有方法
    Object intercept(Invocation invocation) throws Throwable;
	//为被拦截对象生成代理对象
    default Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
	//设置插件配置的参数
    default void setProperties(Properties properties) {
        // NOP
    }

}
```

#### MetaObject

**Mybatis**提供了一个工具类`org.apache.ibatis.reflection.MetaObject`。它通过反射来读取和修改一些重要对象的属性。我们可以利用它来处理四大对象的一些属性，这是Mybatis插件开发的一个常用工具类。

使用`SystemMetaObject.forObject(Object object)`来实例化`MetaObject`对象。

**示例**

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

