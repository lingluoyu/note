## Spring事务（SpringTransaction）

### 事务

事务是逻辑上的一组操作，要么都执行，要么都不执行。

#### 事务特性

- **原⼦性（Atomicity）**：  事务是最⼩的执⾏单位，不允许分割。事务的原⼦性确保动作要么全部完成，要么完全不起作⽤；
- **⼀致性（Consistency）**：  执⾏事务前后，数据保持⼀致，多个事务对同⼀个数据读取的结果是相同的；
- **隔离性（Isolation）**：  并发访问数据库时，⼀个⽤户的事务不被其他事务所⼲扰，各并发事务之间数据库是独⽴的；
- **持久性（Durability）**：  ⼀个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发⽣故障也不应该对其有任何影响。

#### 并发事务问题

- **脏读（Dirty  read）**: 当⼀个事务正在访问数据并且对数据进⾏了修改，⽽这种修改还没有提交到数据库中，这时另外⼀个事务也访问了这个数据，然后使⽤了这个数据。因为这个数据是还没有提交的数据，那么另外⼀个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。
- **丢失修改（Lost  to  modify）**:指在⼀个事务读取⼀个数据时，另外⼀个事务也访问了该数据，那么在第⼀个事务中修改了这个数据后，第⼆个事务也修改了这个数据。这样第⼀个事务内的修改结果就被丢失，因此称为丢失修改。例如：事务1读取某表中的数据A=20，事务2也读取 A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失。
- **不可重复读（Unrepeatableread）**:  指在⼀个事务内多次读同⼀数据。在这个事务还没有结束时，另⼀个事务也访问该数据。那么，在第⼀个事务中的两次读数据之间，由于第⼆个事务的修改导致第⼀个事务两次读取的数据可能不太⼀样。这就发⽣了在⼀个事务内两次读到的数据是不⼀样的情况，因此称为不可重复读。
- **幻读（Phantom  read）**:  幻读与不可重复读类似。它发⽣在⼀个事务（T1）读取了⼏⾏数据，接着另⼀个并发事务（T2）插⼊了⼀些数据时。在随后的查询，第⼀个事务（T1）就会发现多了⼀些原本不存在的记录，就好像发⽣了幻觉⼀样，所以称为幻读。

**幻象读读到了其他已经提交事务的新增数据**，**不可重复读是指读到了已经提交事务的更改数据(更改或者删除)** 

#### 数据库锁机制

数据库通过锁机制解决并发访问的问题。按照锁定的对象的不同，分为表锁定和行锁定。 从并发事务锁定的关系上看，分为共享锁定和独占锁定。共享锁定会防止独占锁定但运行其他的共享锁定，独占锁定防止其他独占锁定和其他的共享锁定。

- **表级锁**：⽐较少，加锁快，不会出现死锁。其锁定粒度最⼤，触发锁冲突的概率最⾼，并发度最低，MyISAM和  InnoDB引擎都⽀持表级锁。
- **⾏级锁**：  MySQL中锁定粒度最⼩的⼀种锁，只针对当前操作的⾏进⾏加锁。⾏级锁能⼤⼤减少数据库操作的冲突。其加锁粒度最⼩，并发度⾼，但加锁的开销也最⼤，加锁慢，会出现死锁。

#### 事务隔离级别

| 隔离级别        | 脏读 | 不可重复读 | 幻象读 | 第一类丢失更新 | 第二类丢失更新 |
| --------------- | ---- | ---------- | ------ | -------------- | -------------- |
| READ UNCOMMITED | Y    | Y          | Y      | N              | Y              |
| READ COMMITED   | N    | Y          | Y      | N              | N              |
| REPEATABLE READ | N    | N          | Y      | N              | N              |
| SERIALIZABLE    | N    | N          | N      | N              |                |

### Spring事务

#### Spring事务管理核心接口

- **`PlatformTransactionManager`**： （平台）事务管理器，Spring 事务策略的核心。
- **`TransactionDefinition`**： 事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)。
- **`TransactionStatus`**： 事务运行状态。

可以把 **`PlatformTransactionManager`** 接口可以被看作是事务上层的管理者，而 **`TransactionDefinition`** 和 **`TransactionStatus`** 这两个接口可以看作是事物的描述。

**`PlatformTransactionManager`** 会根据 **`TransactionDefinition`** 的定义比如事务超时时间、隔离级别、传播行为等来进行事务管理 ，而 **`TransactionStatus`** 接口则提供了一些方法来获取事务相应的状态比如是否新事务、是否可以回滚等等。

##### PlatformTransactionManager事务管理器接口

**Spring 并不直接管理事务，而是提供了多种事务管理器** 。Spring 事务管理器的接口是： **`PlatformTransactionManager`** 。

```java
package org.springframework.transaction;
import org.springframework.lang.Nullable;
public interface PlatformTransactionManager extends TransactionManager {
	//获取事务
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;
	//提交事务
	void commit(TransactionStatus status) throws TransactionException;
	//回滚事务
	void rollback(TransactionStatus status) throws TransactionException;
}
```

其他平台，如JDBC、JPA、Hibernate可以通过实现此接口定义自己的事务管理器

##### TransactionDefinition事务属性

`TransactionDefinition` 接口中定义了 5 个方法以及一些表示事务属性的常量比如隔离级别、传播行为、是否超时、只读状态等等。

```java
public interface TransactionDefinition {
	int PROPAGATION_REQUIRED = 0;
	int PROPAGATION_SUPPORTS = 1;
	int PROPAGATION_MANDATORY = 2;
	int PROPAGATION_REQUIRES_NEW = 3;
	int PROPAGATION_NOT_SUPPORTED = 4;
	int PROPAGATION_NEVER = 5;
	int PROPAGATION_NESTED = 6;
    
	int ISOLATION_DEFAULT = -1;
	int ISOLATION_READ_UNCOMMITTED = 1;  // same as java.sql.Connection.TRANSACTION_READ_UNCOMMITTED;
	int ISOLATION_READ_COMMITTED = 2;  // same as java.sql.Connection.TRANSACTION_READ_COMMITTED;
	int ISOLATION_REPEATABLE_READ = 4;  // same as java.sql.Connection.TRANSACTION_REPEATABLE_READ;
	int ISOLATION_SERIALIZABLE = 8;  // same as java.sql.Connection.TRANSACTION_SERIALIZABLE;
    
	int TIMEOUT_DEFAULT = -1;
	//返回事务的传播行为，默认值为 REQUIRED。
	default int getPropagationBehavior() {
		return PROPAGATION_REQUIRED;
	}
	//返回事务的隔离级别，默认值是 DEFAULT
	default int getIsolationLevel() {
		return ISOLATION_DEFAULT;
	}
	//回事务的超时时间，默认值为-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。
	default int getTimeout() {
		return TIMEOUT_DEFAULT;
	}
	//返回是否为只读事务，默认值为 false
	default boolean isReadOnly() {
		return false;
	}

	@Nullable
	default String getName() {
		return null;
	}
	//返回一个不可更改的事务实例
	// Static builder methods
	static TransactionDefinition withDefaults() {
		return StaticTransactionDefinition.INSTANCE;
	}

}
```
TransactionDefinition中定义了如下七种事务传播行为：

| 事务传播行为类型          | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | 没有事务，新建一个事务，已经存在一个事务，则加入到事务中     |
| PROPAGATION_SUPPORTS      | 支持当前事务，没有事务就以非事务运行                         |
| PROPAGATION_MANDATORY     | 使用当前事务，如果没有事务就抛异常                           |
| PROPAGATION_REQUIRES_NEW  | 新建事务，如果已经存在事务则把当前事务挂起                   |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行，如果当前存在事务，则挂起当前事务           |
| PROPAGATION_NEVER         | 以非事务方式执行，如果当前存在事务，则抛异常                 |
| PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行，如果没有当前事务，则执行PROPAGATION_REQUIRED类似操作 |

五种事务隔离级别：

| 事务隔离级别    | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| DEFAULT         | 默认值。由底层数据库自动判断使用什么隔离级别                 |
| READ_UNCOMMITED | 可以读取到未提交的数据，那么可能会出现脏读、不可重复读、幻读。效率最高。 |
| READ_COMMITED   | 只能读取到其他事务提交的数据。可以防止脏读，可能出现不可重复读和幻读。 |
| REPEATABLE_READ | 读取的数据会被添加锁，防止其他事务修改读取了的数据。可以防止脏读和不可重复读，但还是会出现幻读。 |
| SERIALIZABLE    | 排队操作，对整个表添加锁。一个事务在操作数据时，另一个事务等待事务操作完成时，才能操作这个表。这个事务隔离级别是最安全的，但也是效率最低的。 |

##### TransactionStatus事务状态

`TransactionStatus`接口用来记录事务的状态 该接口定义了一组方法,用来获取或判断事务的相应状态信息。

```java
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {
	//当前事务内部是否存在保存点
	boolean hasSavepoint();
    //刷新底层数据库session
	@Override
	void flush();

    //===========以下四个接口继承自TransactionExecution
    //当前事务是否为新事务，如果返回false，则表示当前事务是一个已经存在的事务，或者当前操作未运行在事务环境中。
    boolean isNewTransaction();
	//设置当前事务为只回滚
    void setRollbackOnly();
	//返回当前事务是否为只回滚
    boolean isRollbackOnly();
	//返回当前事务是否已结束
    boolean isCompleted();
}
```

在Spring中事务管理有两种方式：

- **编程式事务**：需要在代码中显式调用 beginTransaction()、commit()、rollback() 等事务管理相关的方法。
- **声明式事务**：Spring 的声明式事务管理在底层是建立在 AOP 的基础之上的。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

#### 编程式事务

Spring框架提供了两种编程式事务管理的方法：

- `TransactionTemplate`或 `TransactionalOperator`

- `TransactionManager`

`TransactionTemplate`方式

```java
@Autowired
private TransactionTemplate transactionTemplate;
public void testTransaction() {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {

            try {

                // ....  业务代码
            } catch (Exception e){
                //回滚
                transactionStatus.setRollbackOnly();
            }

        }
    });
}
```

基于`PlatformTransactionManager`，使用底层API

```java
@Autowired
private PlatformTransactionManager transactionManager;

public void testTransaction() {
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        // ....  业务代码
        transactionManager.commit(status);
    } catch (Exception e) {
        transactionManager.rollback(status);
    }
}
```

#### 声明式事务

Spring 的声明式事务管理在底层是建立在 AOP 的基础之上的。声明式事务最大的优点就是不需要通过编程的方式管理事务，这样就不需要在业务逻辑代码中掺杂事务管理的代码，只需在配置文件中做相关的事务规则声明（或通过等价的基于标注的方式），便可以将事务规则应用到业务逻辑中。

##### 基于XML配置声明式事务

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- 希望进行事务管理的业务类对象 -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- the transactional advice 配置Advice对象 -->
    <tx:advice id="txAdvice" transaction-manager="txManager">
        <tx:attributes>
            <!-- 需要事务控制的方法 -->
            <tx:method name="get*" read-only="true"/>
            <!-- other methods use the default transaction settings (see below) -->
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- 切点和advice配置 -->
    <aop:config>
        <aop:pointcut id="fooServiceOperation" expression="execution(* x.y.service.FooService.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceOperation"/>
    </aop:config>

    <!-- 配置数据源 -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"/>
        <property name="url" value="jdbc:oracle:thin:@rj-t42:1521:elvis"/>
        <property name="username" value="scott"/>
        <property name="password" value="tiger"/>
    </bean>

    <!-- Spring提供的事务管理对象 -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- 引入配置的数据源 -->
        <property name="dataSource" ref="dataSource"/>
    </bean>

    ......

</beans>
```

`<tx:method/>`标签可以配置的属性如下：

| 属性              | 需要？ | 默认       | 描述                                                         |
| :---------------- | :----- | :--------- | :----------------------------------------------------------- |
| `name`            | 是     |            | 与事务属性关联的方法名称。通配符（*）字符可以被用于相同的事务属性的设置有许多方法相关联（例如，`get*`，`handle*`，`on*Event`，等等）。 |
| `propagation`     | 没有   | `REQUIRED` | 事务传播行为。                                               |
| `isolation`       | 没有   | `DEFAULT`  | 事务隔离级别。仅适用于`REQUIRED`或的传播设置`REQUIRES_NEW`。 |
| `timeout`         | 没有   | -1         | 事务超时（秒）。仅适用于传播`REQUIRED`或传播`REQUIRES_NEW`。 |
| `read-only`       | 没有   | 假         | 读写与只读事务。仅适用于`REQUIRED`或`REQUIRES_NEW`。         |
| `rollback-for`    | 没有   |            | 逗号分隔的`Exception`触发回滚的实例列表。例如， `com.foo.MyBusinessException,ServletException`。 |
| `no-rollback-for` | 没有   |            | `Exception`不触发回滚的实例的逗号分隔列表。例如， `com.foo.MyBusinessException,ServletException`。 |

##### 基于`@Transactional`注解配置声明式事务

**`@Transactional` 的作用范围**：

- **方法** ：推荐将注解使用于方法上
- **类** ：如果这个注解使用在类上的话，表明该注解对该类中所有的 public 方法都生效。
- **接口** ：不推荐在接口上使用。

**使用注意**：

- `@Transactional` 注解只有作用到 public 方法上事务才生效，不推荐在接口上使用；
- 避免同一个类中调用 `@Transactional` 注解的方法，这样会导致事务失效；
- 正确的设置 `@Transactional` 的 rollbackFor 和 propagation 属性，否则事务可能会回滚失败

`@Transactional`代码如下：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
	//配置事务管理器
	@AliasFor("transactionManager")
	String value() default "";
	//配置事务管理器
	@AliasFor("value")
	String transactionManager() default "";

	String[] label() default {};
	//事务的传播行为，默认值为 REQUIRED
	Propagation propagation() default Propagation.REQUIRED;
	//事务的隔离级别，默认值采用 DEFAULT
	Isolation isolation() default Isolation.DEFAULT;
	//事务的超时时间，默认值为-1（不会超时）。如果超过该时间限制但事务还没有完成，则自动回滚事务。
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

	String timeoutString() default "";
	//指定事务是否为只读事务，默认值为 false。
	boolean readOnly() default false;
	//用于指定能够触发事务回滚的异常类型，可以指定多个异常类型。
	Class<? extends Throwable>[] rollbackFor() default {};

	String[] rollbackForClassName() default {};

	Class<? extends Throwable>[] noRollbackFor() default {};

	String[] noRollbackForClassName() default {};
}
```

**`@Transactional` 的工作机制是基于 AOP 实现的，AOP 又是使用动态代理实现的。如果目标对象实现了接口，默认情况下会采用 JDK 的动态代理，如果目标对象没有实现了接口,会使用 CGLIB 动态代理。**

**使用示例**：

```java
@Transactional
public class DefaultFooService implements FooService {
    Foo getFoo(String fooName) {
        // ...
    }
    Foo getFoo(String fooName, String barName) {
        // ...
    }
    void insertFoo(Foo foo) {
        // ...
    }
    void updateFoo(Foo foo) {
        // ...
    }
}
```

并在xml文件中配置启用注解事务：

```xml
<tx:annotation-driven transaction-manager="txManager"/>
```

**Spring AOP 自调用问题**

若同一类中的其他没有 `@Transactional` 注解的方法内部调用有 `@Transactional` 注解的方法，有`@Transactional` 注解的方法的事务会失效。

这是由于`Spring AOP`代理的原因造成的，因为只有当 `@Transactional` 注解的方法在类以外被调用的时候，Spring 事务管理才生效。

**AspectJ配置事务**：

```xml
<tx:annotation-driven mode="aspectj" transaction-manager="transactionManager"/>
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource"ref="dataSource"/>
</bean>
```

