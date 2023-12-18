## RabbitMQ

RabbitMQ是一个实现了AMQP（Advanced Message Queuing Protocol）高级消息队列协议的消息队列服务，用Erlang语言。

优点：

- 由于erlang语言的特性，mq性能较好，高并发；
- 健壮、稳定、易用、跨平台、支持多种语言、文档齐全；
- 有消息确认机制和持久化机制，可靠性高；
- 高度可定制的路由；
- 管理界面较丰富，在互联网公司也有较大规模的应用；
- 社区活跃度高；

缺点：

- 尽管结合erlang语言本身的并发优势，性能较好，但是不利于做二次开发和维护；
- 实现了代理架构，意味着消息在发送到客户端之前可以在中央节点上排队。此特性使得RabbitMQ易于使用和部署，但是使得其运行速度较慢，因为中央节点增加了延迟，消息封装后也比较大；
- 需要学习比较复杂的接口和协议，学习和维护成本较高；

### AMQP

#### AMQP基本概念

![20160310091724939](https://gitee.com/LoopSup/image/raw/master/img/20201205084210.jpg)

* <font color='red'>Broker</font>:接收和分发消息的应用，RabbitMQ Server就是Message Broker。
* <font color='red'>Virtual host</font>:出于多租户和安全因素设计的，把AMQP的基本组件划分到一个虚拟的分组中，类似于网络中的namespace概念。*当多个不同的用户使用同一个RabbitMQ Server提供的服务时，可以划分出多个vhost，每个用户在自己的vhost创建exchange/queue等*。
* <font color='red'>Connection</font>:publisher/consumer和broker之间的TCP连接。*断开连接的操作只会在client端进行，Broker不会断开连接*，除非出现网络故障或broker服务出险问题。
* <font color='red'>Channel</font>:如果每一次访问RabbitMQ都建立一个Connection，在消息量大的时候建立TCP Connection的开销将是巨大的，效率也较低。Channel是在connection内部建立的逻辑连接，如果应用程序支持多线程，通常每个thread创建单独的channel进行通讯，AMQP method包含了channel id帮助客户端和message broker识别channel，所以channel之间是完全隔离的。Channel作为轻量级的Connection极大减少了操作系统建立TCP Connection的开销。
* <font color='red'>Exchange</font>:message到达broker的第一站，根据分发规则，匹配查询表中的routing key，分发消息到queue中去。常用的类型有：direct（point-to-point），topic（publish-subscribe）and fanout（multicast）。
* <font color='red'>Queue</font>:消息最终被送到这里等待consumer取走。一个message可以被同时拷贝到多个queue中。
* <font color='red'>Binding</font>:exchange和queue之间的虚拟连接，binding中可以包含routing key。Binding信息被保存到exchange中的查询表中，用于message的分发依据。

#### 典型的“生产/消费”消息模型

![20160310091838945](https://gitee.com/LoopSup/image/raw/master/img/20201205084225.jpg)

生产者发送消息到broker server（RabbitMQ）。在Broker内部，用户创建Exchange/Queue，通过Binding规则将两者联系在一起。Exchange分发消息，根据类型/binding的不同分发策略有区别。消息最后来到Queue中，等待消费者取走。

#### Exchange类型

Exchange有多种类型，最常用的是Direct/Fanout/Topic三种类型。

Direct

![20160310091854457](https://gitee.com/LoopSup/image/raw/master/img/20201205084239.jpg)

Message中的“routing key”如果和Binding中的“binding key”一致，Direct exchange则将message发到对应的queue中。

Fanout

![20160310091909055](https://gitee.com/LoopSup/image/raw/master/img/20201205084249.jpg)

每个发到Fanout类型Exchange的message都会分到所有绑定的queue上去。

Topic

![20160310091924023](https://gitee.com/LoopSup/image/raw/master/img/20201205084259.jpg)

Routing key中可以包含两种通配符，类似于正则表达式：

```
"#”通配任何零个或多个word 

“*”通配任何单个word
```

### RabbitMQ原理

![v2-cf2ff62088efcca10d15162142015e82_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084624.jpg)

1. Producer：数据发送方

> Create messages and publish (send) them to a Broker Server

一般一个Message有两个部分：payload（有效载荷）和label（标签），payload顾名思义就是传输的数据，label是exchange的名字或者说是一个tag，它描述了payload，而且RabbitMQ也是通过这个label来决定把这个Message发给哪个Consumer

2. Exchange：**rabbitmq**内的消息交换器

> Exchanges are message routing agents， An exchange accepts messages from the producer application and routes them to message queues with help of header attributes, bindings, and routing keys.

exchange从生产者那收到消息后，一般会指定一个Routing Key，来指定这个消息的路由规则，当然Routing Key需要与Exchange Type及Binding key联合使用才能最终生效，根据路由规则，匹配查询表中的routing key，分发消息到queue中。

3. binding：绑定

> A **binding** is a "link" that you set up to bind a queue to an exchange

在绑定（Binding）Exchange与Queue的同时，一般会指定一个Binding key;

但Binding key并不是在所有情况下都生效，它依赖于Exchange Type

4. Queue：队列

是rabbitmq内部对象，用于存储消息；

消息最终被送到这里等待consumer取走。一个message可以被同时拷贝到多个queue中。

那么谁应该负责创建这个queue呢？是Consumer，还是Producer？

> 如果queue不存在，当然Consumer不会得到任何的Message。那么Producer Publish的Message会被丢弃。所以，还是为了数据不丢失，Consumer和Producer都try to create the queue！反正不管怎么样，这个接口都不会出问题。
> queue对load balance的处理是完美的。对于多个Consumer来说，RabbitMQ 使用循环的方式（round-robin）的方式均衡的发送给不同的Consumer。

5. Connection与Channel：两者都是RabbitMQ对外提供的API中最基本的对象

> A Connection represents a real TCP connection to the message broker, whereas a Channel is a virtual connection (AMPQ connection) inside it

**Connection**就是一个TCP的连接，Producer和Consumer都是通过TCP连接到RabbitMQ Server的

**Channel**是建立在上述的TCP连接中，因为建立TCP Connection的开销将是巨大的，所以是节省开销；

Channel是我们与RabbitMQ打交道的最重要的一个接口，我们大部分的业务操作是在Channel这个接口中完成的，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等

6. Consumer：即数据的接收方

> Consumers attach to a Broker Server (RabbitMQ) and subscribe to a queue。

如果有多个消费者同时订阅同一个Queue中的消息，Queue中的消息会被平摊给多个消费者

7. Broker: 即RabbitMQ Server

> RabbitMQ isn't a food truck, it's a delivery service

其作用是维护一条从Producer到Consumer的路线，保证数据能够按照指定的方式进行传输

8. Virtual host：即虚拟主机

当多个不同的用户使用同一个RabbitMQ server提供的服务时，可以划分出多个vhost，每个用户在自己的vhost创建exchange／queue

#### rabbitmq 消息转发流程

![v2-ae6966318989e41001ef11d9f3724f46_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084355.jpg)

1. The producer publishes a message to the exchange.
2. The exchange receives the message and is now responsible for the routing of the message.
3. A binding has to be set up between the queue and the exchange. In this case, we have bindings to two different queues from the exchange. The exchange routes the message in to the queues.
4. The messages stay in the queue until they are handled by a consumer
5. The consumer handles the message.

ps：重点说下路由转发。生产者Producer在发送消息时，都需要指定一个RoutingKey和Exchange，Exchange收到消息后可以看到消息中指定的RoutingKey，再根据当前Exchange的ExchangeType,按一定的规则将消息转发到相应的queue中去。

#### rabbitmq中exchange type

- Direct exchange 直接转发路由

原理是通过消息中的routing key，与binding 中的binding-key 进行比对，若二者匹配，则将消息发送到这个消息队列。

如下图：消息生成者生成一个message(payload是1，routing key为苹果)，两个binding(binding key分别为苹果、香蕉);exchange比对消息的routing key和binding key后，将消息发给了queue1，消息消费者1获得queue1的消息，got msg: 1

![v2-7f766cdbe85f1d46090b3c0aa24bc527_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084416.jpg)

- Fanout exchange 复制分发路由

原理是不需要routkey，当exchange收到消息后，将消息复制多份转发给与自己绑定的消息队列

如下图：消息生成者生成一个message(payload是1，routing key为苹果)，z两个binding(binding key分别为苹果、香蕉)；exchange将消息分发给两个queue，两个消费者获得queue的消息，got msg: 1

![rabbitmq-1](https://gitee.com/LoopSup/image/raw/master/img/rabbitmq-1.jpg)

- topic exchange 通配路由

是direct exchange的通配符模式

如下图：消息生成者生成一个message(payload是1，routing key为quick.orange.rabbit)，两个binding(binding key分别为*.orange.**、***.*.rabbit)；exchange比对消息的routing key和binding key后,exchange将消息分发给两个queue，两个消费者获得queue的消息，got msg: 1

![v2-f16152982f3770224d18630b18b8d21b_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084434.jpg)

再如下图：消息生成者生成一个message(payload是1，routing key为lazy.pink.rabbit)，两个binding(binding key分别为*.orange.**、***.*.rabbit)；exchange比对消息的routing key和binding key后,exchange将消息分发给queue2，消费者2获得queue的消息，got msg: 1

![v2-a649dee77629316ba1e8974bf94d8892_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084451.jpg)

### rabbitmq消息的可靠性

#### 消息持久化（Message durability）

将保存在内存中的数据都写入磁盘，防止服务器重启后数据丢失；有哪些数据需要持久化保存呢？

元数据、消息需要持久化到磁盘；

磁盘节点：持久化的消息在到达队列时就被写入到磁盘，并且如果可以，持久化的消息也会在内存中保存一份备份，这样可以提高一定的性能，只有在内存吃紧的时候才会从内存中清除；

内存节点：非持久化的消息一般只保存在内存中，在内存吃紧的时候会被换入到磁盘中，以节省内存空间；

消息持久化：

```java
//deliveryMode=2表示消息持久化
AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder().deliveryMode(2).build();
channel.basicPublish("exchangeName","routingKey",properties,msg.getBytes());
```

交换机和队列的持久化可以在声明交换机和队列时，将`durable` 参数设置为 `true`。

#### 生产者消息确认机制

如何知道消息有没有正确到达exchange呢？

##### Transaction（事务） 模式

通过AMQP提供的事务机制实现（存在性能问题）：

事务的实现主要是对信道（Channel）的设置，主要的方法有三个：

1. channel.txSelect()声明启动事务模式；
2. channel.txComment()提交事务；
3. channel.txRollback()回滚事务；

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);	
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(_queueName, true, false, false, null);
String message = String.format("时间 => %s", new Date().getTime());
try {
	channel.txSelect(); // 声明事务
	// 发送消息
	channel.basicPublish("", _queueName, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes("UTF-8"));
	channel.txCommit(); // 提交事务
} catch (Exception e) {
	channel.txRollback();
} finally {
	channel.close();
	conn.close();
}
```

##### Confirm（确认）模式

通过生产者消息确认机制（publisher confirm）实现：

Confirm发送方确认模式使用和事务类似，也是通过设置Channel进行发送方确认的。

消息确认模式又可以分为三种（**事务模式和确认模式无法同时开启**）：

- 单条确认模式：发送一条消息，确认一条消息。此种确认模式的效率也不高。
- 批量确认模式：发送一批消息，然后同时确认。批量发送有一个缺点就是同一批消息一旦有一条消息发送失败，就会收到失败的通知，需要将这一批消息全部重发。
- 异步确认模式：一边发送一边确认，消息可能被单条确认也可能会被批量确认。

方式一：channel.waitForConfirms()普通发送方确认模式；

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
String message = String.format("时间 => %s", new Date().getTime());
channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
if (channel.waitForConfirms()) {
	System.out.println("消息发送成功" );
}
```

只需要在推送消息之前，channel.confirmSelect()声明开启发送方确认模式，再使用channel.waitForConfirms()等待消息被服务器确认即可。

方式二：channel.waitForConfirmsOrDie()批量确认模式；

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
for (int i = 0; i < 10; i++) {
	String message = String.format("时间 => %s", new Date().getTime());
	channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
}
channel.waitForConfirmsOrDie(); //直到所有信息都发布，只要有一个未确认就会IOException
System.out.println("全部执行完成");
```

channel.waitForConfirmsOrDie()，使用同步方式等所有的消息发送之后才会执行后面代码，只要有一个消息未被确认就会抛出IOException异常。

方式三：channel.addConfirmListener()异步监听发送方确认模式；

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
for (int i = 0; i < 10; i++) {
	String message = String.format("时间 => %s", new Date().getTime());
	channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
}
//异步监听确认和未确认的消息
channel.addConfirmListener(new ConfirmListener() {
	@Override
	public void handleNack(long deliveryTag, boolean multiple) throws IOException {
		System.out.println("未确认消息，标识：" + deliveryTag);
	}
	@Override
	public void handleAck(long deliveryTag, boolean multiple) throws IOException {
		System.out.println(String.format("已确认消息，标识：%d，多个消息：%b", deliveryTag, multiple));
	}
});
```

异步模式执行效率高，不需要等待消息执行完，只需要监听消息即可

代码是异步执行的，消息确认有可能是批量确认的，是否批量确认在于返回的multiple的参数，此参数为bool值，如果true表示批量执行了deliveryTag这个值以前的所有消息，如果为false的话表示单条确认。

#### 消息无法从交换机路由到正确的队列

`RabbitMQ` 中提供了 `2` 种方式来确保消息可以正确路由到队列：开启监听模式或者通过新增备份交换机模式来备份数据。

##### 监听回调

```java
channel.addReturnListener(new ReturnListener() {
     @Override
     public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, AMQP.BasicProperties properties, byte[] body) throws IOException {
         System.out.println("收到未路由到队列的回调消息：" + new String(body));
     }
 });
//注意这里的第三个参数，mandatory需要设置为true（发送一个错误的路由，即可收到回调）
channel.basicPublish(EXCHANGE_NAME,"ERROR_ROUTING_KEY",true,null,msg.getBytes());
```

##### 备份交换机

当原交换机无法正确路由到队列时，则会进入备份交换机，再由备份交换机路由到正确队列。

```java
 //声明交换机且指定备份交换机
Map<String,Object> argMap = new HashMap<String,Object>();
argMap.put("alternate-exchange","TEST_ALTERNATE_EXCHANGE");
channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT,false,false,argMap);
//队列和交换机进行绑定
channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,ROUTEING_KEY);

//声明备份交换机和备份队列，并绑定（为了防止收不到消息，备份交换机一般建议设置为Fanout类型）
channel.queueDeclare("BAK_QUEUE", false, false, false, null);
channel.exchangeDeclare("TEST_ALTERNATE_EXCHANGE", BuiltinExchangeType.TOPIC);
channel.queueBind("BAK_QUEUE","TEST_ALTERNATE_EXCHANGE","ERROR.#");

String msg = "I'm a bak exchange msg";
channel.basicPublish(EXCHANGE_NAME,"ERROR.ROUTING_KEY",null,msg.getBytes());
```

#### 消费者消息确认机制

在实际应用中，可能会发生消费者收到Queue中的消息，但没有处理完成就宕机（或出现其他意外）的情况，这种情况下就可能会导致消息丢失。为了避免这种情况发生，我们可以要求消费者在消费完消息后发送一个回执给RabbitMQ，RabbitMQ收到消息回执（Message acknowledgment）后才将该消息从Queue中移除。

如果一个Queue没被任何的Consumer Subscribe（订阅），当有数据到达时，这个数据会被cache，不会被丢弃。当有Consumer时，这个数据会被立即发送到这个Consumer。这个数据被Consumer正确收到时，这个数据就被从Queue中删除。

那么什么是正确收到呢？通过ACK。每个Message都要被acknowledged（确认，ACK）。我们可以显示的在程序中去ACK，也可以自动的ACK。如果有数据没有被ACK，那么RabbitMQ Server会把这个信息发送到下一个Consumer。

消费者确认：

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);	
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
//声明队列（默认交换机AMQP default，Direct）
channel.queueDeclare(QUEUE_NAME, false, false, false, null);
System.out.println(" 等待接收消息...");

// 创建消费者
Consumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                               byte[] body) throws IOException {
        System.out.println("收到消息: " + new String(body, "UTF-8"));
        Map<String,Object> map = properties.getHeaders();//获取头部消息
        String ackType = map.get("ackType").toString();
        if (ackType.equals("ack")){//手动应答
            channel.basicAck(envelope.getDeliveryTag(),true);
        }else if(ackType.equals("reject-single")){//拒绝单条消息
            //拒绝消息。requeue参数表示消息是否重新入队
            channel.basicReject(envelope.getDeliveryTag(),false);
            //channel.basicNack(envelope.getDeliveryTag(),false,false);
        }else if (ackType.equals("reject-multiple")){//拒绝多条消息
            //拒绝消息。multiple参数表示是否批量拒绝，为true则表示<deliveryTag的消息都被拒绝
            channel.basicNack(envelope.getDeliveryTag(),true,false);
        }
    }
};

//开始获取消息,第二个参数 autoAck表示是否开启自动应答
channel.basicConsume(QUEUE_NAME, false, consumer);
```

生产者代码：

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);	
Connection conn = factory.newConnection();
// 创建消息通道
Channel channel = conn.createChannel();
Map<String, Object> headers = new HashMap<String, Object>(1);
headers.put("ackType", "ack");//请应答
//headers.put("ackType", "reject-single");//请单条拒绝
//headers.put("ackType", "reject-multiple");//请多条拒绝

AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
    .contentEncoding("UTF-8")  // 编码
    .headers(headers) // 自定义属性
    .messageId(String.valueOf(UUID.randomUUID()))
    .build();

String msg = "I'm a ack message";
//声明队列
channel.queueDeclare(QUEUE_NAME, false, false, false, null);
//声明交换机
channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT,false);
//队列和交换机进行绑定
channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,ROUTEING_KEY);
// 发送消息
channel.basicPublish(EXCHANGE_NAME, ROUTEING_KEY, properties, msg.getBytes());

channel.close();
conn.close();
```

#### 死信队列

DLX，Dead Letter Exchange 的缩写，又死信邮箱、死信交换机。DLX就是一个普通的交换机，和一般的交换机没有任何区别。 当消息在一个队列中变成死信（dead message）时，通过这个交换机将死信发送到死信队列中（指定好相关参数，rabbitmq会自动发送）

**以下情况会形成死信**：

- 消息被拒绝（basic.reject或basic.nack）并且requeue=false.
- 消息TTL过期
- 队列达到最大长度（队列满了，无法再添加数据到mq中）

**应用场景分析：** 在定义业务队列的时候，可以考虑指定一个死信交换机，并绑定一个死信队列，当消息变成死信时，该消息就会被发送到该死信队列上，这样就方便我们查看消息失败的原因了 

**使用死信交换机**

定义业务（普通）队列的时候指定参数：

- x-dead-letter-exchange: 用来设置死信后发送的交换机
- x-dead-letter-routing-key：用来设置死信的routingKey

```java
@Bean
public Queue helloQueue() {
    //将普通队列绑定到私信交换机上
    Map<String, Object> args = new HashMap<>(2);
    args.put(DEAD_LETTER_QUEUE_KEY, deadExchangeName);
    args.put(DEAD_LETTER_ROUTING_KEY, deadRoutingKey);
    Queue queue = new Queue(queueName, true, false, false, args);
    return queue;
}
```

### 集群

集群模式下RabbitMQ 默认会将消息冗余到所有节点上吗？

> 答案是不会

![v2-1a8c0f9dec63787f021889873bf5cf49_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084512.jpg)

三个节点组成了一个RabbitMQ的集群，Exchange A的元数据信息在所有节点上是一致的，而Queue（存放消息的队列）的完整数据则只会存在于它所创建的那个节点上，其他节点只知道这个queue的metadata信息和一个指向queue的owner node的指针。

**集群元数据的同步**

rabbitmq考虑存储空间、性能的原因，所以集群内部仅同步元数据

RabbitMQ 内部有各种基础构件，包括队列、交换器、绑定、虚拟主机等，这些构件以元数据的形式存在

> Queue元数据：队列的名称和声明队列时设置的属性(是否持久化、是否自动删除、队列所属的节点)
> Exchange元数据：交换机的名称、类型、属性(是否持久化等)
> Bindding元数据：一张简单的表格展示了如何将消息路由到队列。包含的列有 Exchange名称、Exchange类型、routing_key、queue_name等
> vhost元数据：为vhost内队列、交换机和绑定提供命名空间和安全属性

**集群消息转发**

例如当消息进入A节点的Queue中后，consumer却从B节点拉取时，集群内部做了什么？

> RabbitMQ会临时在A、B间进行消息传输，把A中的消息实体取出并经过B发送给consumer，所以consumer应平均连接每一个节点，从中取消息

**集群节点故障**

如果特定队列的所有者节点发生了故障，那么该节点上的队列和关联的绑定都会消失吗？

> 分两种情况，看集群内节点是内存模式还是磁盘模式：
> 如果是内存节点，那么附加在该节点上的队列和其关联的绑定都会丢失，并且消费者可以重新连接集群并重新创建队列；
> 如果是磁盘节点，重新恢复故障后，该队列又可以进行传输数据了，并且在恢复故障磁盘节点之前，不能在其它节点上让消费者重新连到集群并重新创建队列，如果消费者继续在其它节点上声明该队列，会得到一个 404 NOT_FOUND 错误，这样确保了当故障节点恢复后加入集群，该节点上的队列消息不回丢失，也避免了队列会在一个节点以上出现冗余的问题。

**注意点**：

在单节点 RabbitMQ 上，仅允许该节点是磁盘节点，这样确保了节点发生故障或重启节点之后，所有关于系统的配置与元数据信息都会重磁盘上恢复；

而在 RabbitMQ 集群上，要求集群里至少有一个磁盘节点，所有其他节点可以是内存节点，当节点加入或者离开集群时，必须要将该变更通知到至少一个磁盘节点。

如果是唯一的磁盘节点也发生故障了，集群可以继续路由消息，但是不可以做以下操作了：

- 创建队列
- 创建交换器
- 创建绑定
- 添加用户
- 更改权限
- 添加或删除集群节点

因为上述操作都需要持久化到磁盘节点上，以便内存节点恢复故障可以从磁盘节点上恢复元数据。

所以好的方案就是在集群添加 2 台以上的磁盘节点，这样其中一台发生故障了，集群仍然可以保持运行，且能够在任何时候保存元数据变更。

### rabbitmq常见故障

- 集群状态异常

1. `rabbitmqctl cluster_status`检查集群健康状态，不正常节点重新加入集群
2. 分析是否节点挂掉，手动启动节点。
3. 保证网络连通正常

- 队列阻塞、数据堆积

1. 保证网络连通正常
2. 保证消费者正常消费，消费速度大于生产速度
3. 保证服务器TCP连接限制合理

- 脑裂[[1\]](https://zhuanlan.zhihu.com/p/63700605#ref_1)[[2\]](https://zhuanlan.zhihu.com/p/63700605#ref_2)

1. 按正确顺序重启集群
2. 保证网络连通正常
3. 保证磁盘空间、cpu、内存足够

#### 参考

1. [^](https://zhuanlan.zhihu.com/p/63700605#ref_1_0)RabbitMQ 网络分区问题 https://88250.b3log.org/rabbitmq-network-partition
2. [^](https://zhuanlan.zhihu.com/p/63700605#ref_2_0)RabbitMQ脑裂 https://blog.csdn.net/u013256816/article/details/53291907