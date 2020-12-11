#### RabbitMQ

RabbitMQ是一个实现了AMQP（Advanced Message Queuing Protocol）高级消息队列协议的消息队列服务，用Erlang语言。

##### RabbitMQ原理

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

##### rabbitmq 消息转发流程

![v2-ae6966318989e41001ef11d9f3724f46_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084355.jpg)

1. The producer publishes a message to the exchange.
2. The exchange receives the message and is now responsible for the routing of the message.
3. A binding has to be set up between the queue and the exchange. In this case, we have bindings to two different queues from the exchange. The exchange routes the message in to the queues.
4. The messages stay in the queue until they are handled by a consumer
5. The consumer handles the message.

ps：重点说下路由转发。生产者Producer在发送消息时，都需要指定一个RoutingKey和Exchange，Exchange收到消息后可以看到消息中指定的RoutingKey，再根据当前Exchange的ExchangeType,按一定的规则将消息转发到相应的queue中去。

##### rabbitmq中exchange type

- Direct exchange 直接转发路由

原理是通过消息中的routing key，与binding 中的binding-key 进行比对，若二者匹配，则将消息发送到这个消息队列。

如下图：消息生成者生成一个message(payload是1，routing key为苹果)，两个binding(binding key分别为苹果、香蕉);exchange比对消息的routing key和binding key后，将消息发给了queue1，消息消费者1获得queue1的消息，got msg: 1

![v2-7f766cdbe85f1d46090b3c0aa24bc527_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084416.jpg)

- Fanout exchange 复制分发路由

原理是不需要routkey，当exchange收到消息后，将消息复制多份转发给与自己绑定的消息队列

如下图：消息生成者生成一个message(payload是1，routing key为苹果)，z两个binding(binding key分别为苹果、香蕉)；exchange将消息分发给两个queue，两个消费者获得queue的消息，got msg: 1

![img](D:\MySpace\GitHub\note\picture\v2-e2b7bc4ab9e0598aff01c3855ab2f383_720w.jpg)

- topic exchange 通配路由

是direct exchange的通配符模式

如下图：消息生成者生成一个message(payload是1，routing key为quick.orange.rabbit)，两个binding(binding key分别为*.orange.**、***.*.rabbit)；exchange比对消息的routing key和binding key后,exchange将消息分发给两个queue，两个消费者获得queue的消息，got msg: 1

![v2-f16152982f3770224d18630b18b8d21b_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084434.jpg)

再如下图：消息生成者生成一个message(payload是1，routing key为lazy.pink.rabbit)，两个binding(binding key分别为*.orange.**、***.*.rabbit)；exchange比对消息的routing key和binding key后,exchange将消息分发给queue2，消费者2获得queue的消息，got msg: 1

![v2-a649dee77629316ba1e8974bf94d8892_720w](https://gitee.com/LoopSup/image/raw/master/img/20201205084451.jpg)

##### rabbitmq消息的可靠性

**1、Message durability**

将保存在内存中的数据都写入磁盘，防止服务器重启后数据丢失；有哪些数据需要持久化保存呢？

元数据、消息需要持久化到磁盘；

磁盘节点：持久化的消息在到达队列时就被写入到磁盘，并且如果可以，持久化的消息也会在内存中保存一份备份，这样可以提高一定的性能，只有在内存吃紧的时候才会从内存中清除；

内存节点：非持久化的消息一般只保存在内存中，在内存吃紧的时候会被换入到磁盘中，以节省内存空间；

**2、Message acknowledgment**

在实际应用中，可能会发生消费者收到Queue中的消息，但没有处理完成就宕机（或出现其他意外）的情况，这种情况下就可能会导致消息丢失。为了避免这种情况发生，我们可以要求消费者在消费完消息后发送一个回执给RabbitMQ，RabbitMQ收到消息回执（Message acknowledgment）后才将该消息从Queue中移除。

如果一个Queue没被任何的Consumer Subscribe（订阅），当有数据到达时，这个数据会被cache，不会被丢弃。当有Consumer时，这个数据会被立即发送到这个Consumer。这个数据被Consumer正确收到时，这个数据就被从Queue中删除。

那么什么是正确收到呢？通过ACK。每个Message都要被acknowledged（确认，ACK）。我们可以显示的在程序中去ACK，也可以自动的ACK。如果有数据没有被ACK，那么RabbitMQ Server会把这个信息发送到下一个Consumer。

**3、生产者消息确认机制**

如何知道消息有没有正确到达exchange呢？

1、通过AMQP提供的事务机制实现：

[RabbitMQ系列（四）RabbitMQ事务和Confirm发送方消息确认--深入解读 - 掘金juejin.im](https://juejin.im/post/5b54681bf265da0f82023014)

2、通过生产者消息确认机制（publisher confirm）实现：

[RabbitMQ系列（四）RabbitMQ事务和Confirm发送方消息确认--深入解读 - 掘金juejin.im](https://juejin.im/post/5b54681bf265da0f82023014)

#### 部署rabbitmq

**单节点环境：**

1、下载rpm包，yum 安装

2、配置

/etc/rabbitmq/rabbitmq-env.conf 服务环境配置

/etc/rabbitmq/rabbitmq.config 服务配置

3、启动

rabbitmq-server -detached

4、启动web组件

rabbitmq-plugins enable rabbitmq_management

5、创建用户并授予root用户为管理员

rabbitmqctl add_user root root

rabbitmqctl set_user_tags root administrator

**集群环境：**

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

#### **rabbitmq常见故障**

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

#### **常用命令**

启动rabbit服务：service rabbitmq-server start

停止rabbit服务：service rabbitmq-server stop

后台启动：rabbitmq-server -detached

运行状态：rabbitmqctl status

用户管理

查看所有用户：rabbitmqctl list_users

添加用户：rabbitmqctl add_user username password

删除用户：rabbitmqctl delete_user username

修改密码：rabbitmqctl change_password username newpassword

开启rabbit网页控制台

进入rabbit安装目录：cd /usr/lib/rabbitmq

查看已经安装的插件：rabbitmq-plugins list

开启网页版控制台：rabbitmq-plugins enable rabbitmq_management

重启rabbitmq服务

输入网页访问地址：http://localhost:15672/ 使用默认账号：guest/guest登录

#### **参考：**

[史上最透彻的 RabbitMQ 可靠消息传输实战juejin.im](https://juejin.im/entry/5bbc22135188255c6a0456ef)

[RabbitMQ集群原理与部署objcoding.com](http://objcoding.com/2018/10/19/rabbitmq-cluster/)

[消息队列之 RabbitMQwww.jianshu.com](https://www.jianshu.com/p/79ca08116d57)

[rabbitmq 原理、集群、基本运维操作、常见故障处理cloud.tencent.com](https://cloud.tencent.com/developer/article/1391426)

#### 参考

1. [^](https://zhuanlan.zhihu.com/p/63700605#ref_1_0)RabbitMQ 网络分区问题 https://88250.b3log.org/rabbitmq-network-partition
2. [^](https://zhuanlan.zhihu.com/p/63700605#ref_2_0)RabbitMQ脑裂 https://blog.csdn.net/u013256816/article/details/53291907