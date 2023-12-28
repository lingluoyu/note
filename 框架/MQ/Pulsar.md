#### 主要特点

- Pulsar 的单个实例原生支持多个集群，可跨机房在集群间无缝地完成消息复制。
- 非常低的发布延迟和端到端延迟。
- 可无缝扩展到超过一百万个 topic。
- 简单的客户端 API，支持 Java、Go、Python 和 C++。
- 主题的多种订阅模式（独占、共享和故障转移）。
- 通过 Apache BookKeeper 提供的持久化消息存储机制保证消息传递 。
- 由轻量级的 serverless 计算框架 Pulsar Functions 实现流原生的数据处理。
- 基于 Pulsar Functions 的 serverless connector 框架 Pulsar IO 使得数据更易移入、移出 Apache Pulsar。
- 分层式存储可在数据陈旧时，将数据从热存储卸载到冷/长期存储（如S3、GCS）中。

#### 核心概念

##### Message 消息

Pulsar 建立在发布-订阅模式（通常缩写为 pub-sub）之上。在此模式中，生产者将消息发布到主题;使用者订阅这些主题，处理传入的消息，并在处理完成后向代理发送确认。

创建订阅时，Pulsar 会保留所有消息，即使消费者断开连接也是如此。仅当使用者确认所有这些消息都已成功处理时，才会丢弃保留的消息。

如果消息消费失败，并且希望再次消费此消息，可以启用消息重投机制以请求代理重新发送此消息。

###### 消息组成

| Component 元件                       | Description 描述                                             |
| ------------------------------------ | ------------------------------------------------------------ |
| Value / data payload 值/数据有效载荷 | The data carried by the message. All Pulsar messages contain raw bytes, although message data can also conform to data [schemas](https://pulsar.apache.org/docs/3.1.x/schema-get-started/). 消息携带的数据。所有 Pulsar 消息都包含原始字节，尽管消息数据也可以符合数据模式。 |
| Key                                  | The key (string type) of the message. It is a short name of message key or partition key. Messages are optionally tagged with keys, which is useful for features like [topic compaction](https://pulsar.apache.org/docs/3.1.x/concepts-topic-compaction/). 消息的键（字符串类型）。它是消息键或分区键的短名称。消息可以选择使用键进行标记，这对于主题压缩等功能非常有用。 |
| Properties 性能                      | An optional key/value map of user-defined properties. 用户定义属性的可选键/值映射。 |
| Producer name 生产者名称             | The name of the producer who produces the message. If you do not specify a producer name, the default name is used. 生成消息的创建者的名称。如果未指定生产者名称，则使用默认名称。 |
| Topic name 主题名称                  | The name of the topic that the message is published to. 将消息发布到的主题的名称。 |
| Schema version 架构版本              | The version number of the schema that the message is produced with. 用于生成消息的架构的版本号。 |
| Sequence ID 序列 ID                  | Each Pulsar message belongs to an ordered sequence on its topic. The sequence ID of a message is initially assigned by its producer, indicating its order in that sequence, and can also be customized. 每条 Pulsar 消息都属于其主题的一个有序序列。消息的序列 ID 最初由其生产者分配，指示其在该序列中的顺序，也可以自定义。 Sequence ID can be used for message deduplication. If `brokerDeduplicationEnabled` is set to `true`, the sequence ID of each message is unique within a producer of a topic (non-partitioned) or a partition. 序列 ID 可用于消息重复数据删除。如果 `brokerDeduplicationEnabled` 设置为 `true` ，则每条消息的序列 ID 在主题（未分区）或分区的创建者中是唯一的。 |
| Message ID 消息 ID                   | The message ID of a message is assigned by bookies as soon as the message is persistently stored. Message ID indicates a message's specific position in a ledger and is unique within a Pulsar cluster. 一旦消息被持久存储，消息的消息 ID 就会由 bookies 分配。消息 ID 表示消息在账本中的特定位置，在 Pulsar 集群中是唯一的。 |
| Publish time 发布时间                | The timestamp of when the message is published. The timestamp is automatically applied by the producer. 消息发布时的时间戳。时间戳由生产者自动应用。 |
| Event time 活动时间                  | An optional timestamp attached to a message by applications. For example, applications attach a timestamp on when the message is processed. If nothing is set to event time, the value is `0`. 应用程序附加到消息的可选时间戳。例如，应用程序在处理消息时附加时间戳。如果未将任何内容设置为事件时间，则该值为 `0` 。 |

##### 确认

消费者在成功使用消息后向代理发送确认请求。然后，此使用的消息将被永久存储，并且仅在所有订阅都确认后才删除。如果要存储消费者已确认的消息，则需要配置消息保留策略。

可以通过以下两种方式确认消息：

- 单条确认。通过单独确认，使用者确认每条消息并向代理发送确认请求。

```
consumer.acknowledge(msg);
```

- 累计确认。使用累积确认时，使用者仅确认其收到的最后一条消息。流中直到（包括）提供的消息的所有消息都不会重新传递到该使用者。

```
consumer.acknowledgeCumulative(msg);
```

> 累积确认不能在共享订阅类型中使用，因为共享订阅类型涉及有权访问同一订阅的多个使用者。在“共享订阅”类型中，将单独确认消息。

##### Topic 主题

Pulsar 中的主题是用于将消息从生产者传输到消费者的命名通道。主题名称是具有明确定义结构的 URL：

```
{persistent|non-persistent}://tenant/namespace/topic
```

| Topic name component 主题名称组件 | Description 描述                                             |
| --------------------------------- | ------------------------------------------------------------ |
| `persistent` / `non-persistent`   | This identifies the type of topic. Pulsar supports two kind of topics: [persistent](https://pulsar.apache.org/docs/3.1.x/concepts-architecture-overview/#persistent-storage) and [non-persistent](https://pulsar.apache.org/docs/3.1.x/concepts-messaging/#non-persistent-topics). The default is persistent, so if you do not specify a type, the topic is persistent. With persistent topics, all messages are durably persisted on disks (if the broker is not standalone, messages are durably persisted on multiple disks), whereas data for non-persistent topics is not persisted to storage disks. 这标识了主题的类型。Pulsar 支持两种主题：持久性和非持久性。默认值为持久性，因此，如果未指定类型，则主题是持久性的。对于持久性主题，所有消息都持久地保存在磁盘上（如果代理不是独立的，则消息持久地保存在多个磁盘上），而非持久性主题的数据不会持久保存到存储磁盘上。 |
| `tenant`                          | The topic tenant within the instance. Tenants are essential to multi-tenancy in Pulsar, and spread across clusters. 实例中的主题租户。租户对于 Pulsar 中的多租户至关重要，并且分布在集群中。 |
| `namespace`                       | The administrative unit of the topic, which acts as a grouping mechanism for related topics. Most topic configuration is performed at the [namespace](https://pulsar.apache.org/docs/3.1.x/concepts-messaging/#namespaces) level. Each tenant has one or more namespaces. 主题的管理单元，充当相关主题的分组机制。大多数主题配置都是在命名空间级别执行的。每个租户都有一个或多个命名空间。 |
| `topic`                           | The final part of the name. Topic names have no special meaning in a Pulsar instance. 名称的最后一部分。主题名称在 Pulsar 实例中没有特殊含义。 |

###### 命名空间

命名空间是租户中的逻辑命名法。租户通过管理 API 创建命名空间。例如，具有不同应用程序的租户可以为每个应用程序创建单独的命名空间。命名空间允许应用程序创建和管理主题层次结构。该主题 `my-tenant/app1` 是 的应用程序 `app1` `my-tenant` 的命名空间。您可以在命名空间下创建任意数量的主题。

###### 订阅

订阅是一个命名的配置规则，用于确定如何将消息传递给使用者。Pulsar 提供四种订阅类型：独占订阅、共享订阅、故障转移订阅和key_shared订阅。

**订阅类型**

1. Exclusive 独占

   在“独占”类型中，只允许将单个使用者附加到订阅。如果多个消费者使用同一个订阅订阅一个主题，则会发生错误。

   如果对主题进行分区，则所有分区都将由允许连接到订阅的单个使用者使用。

   **独占是默认订阅类型**

2. Failover 故障转移

   在故障转移类型中，多个使用者可以附加到同一订阅。

   为未分区的主题或分区主题的每个分区选择一个主使用者，并接收消息。

   当主使用者断开连接时，所有（未确认的和后续的）消息都将传递到下一个使用者。

3. Shared 共享

   在共享或循环类型中，多个使用者可以附加到同一订阅。消息在使用者之间以循环分布方式传递，任何给定的消息都只传递到一个使用者。当使用者断开连接时，所有已发送给它但未确认的消息都将被重新安排以发送给其余使用者。

4. Key_Shared Key 共享

   在Key_Shared类型中，多个使用者可以附加到同一订阅。消息在使用者之间分发，具有相同密钥或相同排序密钥的消息仅传递到一个使用者。无论消息重新传递多少次，它都会传递到同一个使用者。

#### 体系架构

一个 Pulsar 实例由一个或多个 Pulsar 集群组成。实例中的集群可以在它们之间复制数据。

- 一个或多个 broker 处理和负载均衡来自生产者的传入消息，将消息分派给消费者，与 Pulsar 配置存储通信以处理各种协调任务，将消息存储在 BookKeeper 实例（又名 bookies）中，依赖特定于集群的 ZooKeeper 集群来执行某些任务，等等。
- 一个或多个 Bookie 组成的 BookKeeper 集群处理消息的持久存储。
- ZooKeeper 集群负责处理 Pulsar 集群之间的协调任务。

##### Brokers

Pulsar message broker 是一个无状态组件，主要负责运行另外两个组件：

- 一个 HTTP 服务器，它为生产者和使用者的管理任务和主题查找公开 REST API。生产者连接到代理以发布消息，使用者连接到代理以使用消息。
- 调度程序，它是基于自定义二进制协议的异步 TCP 服务器，用于所有数据传输

##### Metadata store 元数据存储

Pulsar 元数据存储维护 Pulsar 集群的所有元数据，例如主题元数据、schema、broker load 数据等。Pulsar 使用 Apache ZooKeeper 进行元数据存储、集群配置和协调。Pulsar 元数据存储可以部署在单独的 ZooKeeper 集群上，也可以部署在现有的 ZooKeeper 集群上。您可以将一个 ZooKeeper 集群用于 Pulsar 元数据存储和 BookKeeper 元数据存储。如果你想部署连接到现有 BookKeeper 集群的 Pulsar broker，你需要分别为 Pulsar 元数据存储和 BookKeeper 元数据存储部署单独的 ZooKeeper 集群。

在 Pulsar 实例中：

- 配置存储仲裁存储租户、命名空间和其他需要全局一致的实体的配置。
- 每个集群都有自己的本地 ZooKeeper 集合，用于存储特定于集群的配置和协调，例如哪些代理负责哪些主题以及所有权元数据、代理负载报告、BookKeeper 账本元数据等。

##### Configuration store 配置存储

配置存储维护 Pulsar 实例的所有配置，例如集群、租户、命名空间、分区主题相关配置等。一个 Pulsar 实例可以有一个本地集群、多个本地集群或多个跨区域集群。因此，配置存储可以在 Pulsar 实例下的多个集群之间共享配置。配置存储可以部署在单独的 ZooKeeper 集群上，也可以部署在现有的 ZooKeeper 集群上。

##### Persistent storage 持久性存储

Pulsar 为应用程序提供有保证的消息传递。如果一条消息成功到达 Pulsar broker，它将被传递到其预期目标。

此保证要求持久存储未确认的消息，直到它们可以传递到使用者并由使用者确认为止。这种消息传递模式通常称为持久消息传递。在 Pulsar 中，所有消息的 N 个副本都存储在磁盘上并同步，例如，4 个副本跨两台服务器，每台服务器上都有镜像的 RAID 卷。

##### Apache BookKeeper

Pulsar 使用一个名为 Apache BookKeeper 的系统进行持久消息存储。BookKeeper 是一个分布式预写日志 （WAL） 系统，它为 Pulsar 提供了几个关键优势：

- 使 Pulsar 能够利用许多独立的日志，称为账本。随着时间推移，可以为主题创建多个账本。
- 为处理条目复制的顺序数据提供了非常高效的存储。
- 保证了在存在各种系统故障的情况下账本的读取一致性。
- 在 bookies 之间提供均匀的 I/O 分布。
- 在容量和吞吐量方面均可水平扩展。通过向集群添加更多 bookies，可以立即增加容量。
- bookies 旨在通过并发读取和写入来处理数千个账本。通过使用多个磁盘设备---一个用于日志，另一个用于常规存储），Bookies 可以将读取操作的影响与正在进行的写入操作的延迟隔离开来。

除了消息数据之外，游标还持久存储在 BookKeeper 中。游标是使用者的订阅位置。BookKeeper 使 Pulsar 能够以可扩展的方式存储消费者位置。

##### Pulsar proxy Pulsar 代理

Pulsar 客户端与 Pulsar 集互的一种方式是直接连接到 Pulsar 消息 brokers。但是，在某些情况下，这种直接连接是不可行或不可取的，因为客户端无法直接访问 brokers 地址。例如，如果你在云环境或 Kubernetes 或类似平台上运行 Pulsar，那么客户端可能无法直接连接到 brokers。

Pulsar 代理通过充当集群中所有代理的单个网关来解决这个问题。如果你运行 Pulsar 代理（同样，这是可选的），所有与 Pulsar 集群的客户端连接都将流经代理，而不是与 brokers 通信。

##### Service discovery 服务发现

连接到 Pulsar broker 的客户端需要能够使用单个 URL 与整个 Pulsar 实例进行通信。

##### Producer 生产者

生产者是附加到主题并将消息发布到 Pulsar broker 的进程。Pulsar broker 处理消息。

###### Send Mode 发送方式

生产者以同步或异步方式向 brokers 发送消息。

| Mode 模式           | Description 描述                                             |
| ------------------- | ------------------------------------------------------------ |
| Sync send 同步发送  | The producer waits for an acknowledgment from the broker after sending every message. If the acknowledgment is not received, the producer treats the sending operation as a failure. 生产者在发送每条消息后等待代理的确认。如果未收到确认，则生成者会将发送操作视为失败。 |
| Async send 异步发送 | The producer puts a message in a blocking queue and returns immediately. The client library sends the message to the broker in the background. If the queue is full (you can [configure](https://pulsar.apache.org/docs/3.1.x/reference-configuration/#broker) the maximum size), the producer is blocked or fails immediately when calling the API, depending on arguments passed to the producer. 生产者将消息放入阻塞队列中并立即返回。客户端库在后台将消息发送到代理。如果队列已满（您可以配置最大大小），则创建者在调用 API 时会被阻止或立即失败，具体取决于传递给创建者的参数。 |

###### Access Mode 访问模式

| Access mode 访问方式   | Description 描述                                             |
| ---------------------- | ------------------------------------------------------------ |
| `Shared`               | Multiple producers can publish on a topic. 多个生产者可以就一个主题发布。  This is the **default** setting. 这是默认设置。 |
| `Exclusive`            | Only one producer can publish on a topic. 一个主题只能由一个制作者发布。  If there is already a producer connected, other producers trying to publish on this topic get errors immediately. 如果已经连接了生产者，则尝试发布此主题的其他生产者会立即收到错误。  The "old" producer is evicted and a "new" producer is selected to be the next exclusive producer if the "old" producer experiences a network partition with the broker. 如果“旧”生产者遇到与代理的网络分区，则“旧”生产者将被逐出，并选择“新”生产者作为下一个独占生产者。 |
| `ExclusiveWithFencing` | Only one producer can publish on a topic. 一个主题只能由一个制作者发布。  If there is already a producer connected, it will be removed and invalidated immediately. 如果已经连接了生产者，它将被删除并立即失效。 |
| `WaitForExclusive`     | If there is already a producer connected, the producer creation is pending (rather than timing out) until the producer gets the `Exclusive` access. 如果已经连接了生产者，则生产者创建将处于挂起状态（而不是超时），直到生产者获得 `Exclusive` 访问权限。  The producer that succeeds in becoming the exclusive one is treated as the leader. Consequently, if you want to implement a leader election scheme for your application, you can use this access mode. Note that the leader pattern scheme mentioned refers to using Pulsar as a Write-Ahead Log (WAL) which means the leader writes its "decisions" to the topic. On error cases, the leader will get notified it is no longer the leader *only* when it tries to write a message and fails on appropriate error, by the broker. 成功成为独家生产者的生产者被视为领导者。因此，如果要为应用程序实现领导者选举方案，则可以使用此访问模式。请注意，提到的领导者模式方案是指使用 Pulsar 作为预写日志 （WAL），这意味着领导者将其“决策”写入主题。在错误情况下，只有当 leader 尝试写入消息并在适当的错误时失败时，代理才会通知它不再是 leader。 |

##### Consumer 消费者

消费者是通过订阅附加到主题，然后接收消息的进程。

消费者向代理发送流许可请求以获取消息。消费者端有一个队列，用于接收从代理推送的消息。您可以使用该 `receiverQueueSize` 参数配置队列大小。默认大小为 `1000` ）。 `consumer.receive()` 每次调用时，都会从缓冲区中取消一条消息。

###### Receive mode 接收模式

消息以同步或异步方式从 brokers 接收。

| Mode 模式              | Description 描述                                             |
| ---------------------- | ------------------------------------------------------------ |
| Sync receive 同步接收  | A sync receive is blocked until a message is available. 同步接收将被阻止，直到消息可用。 |
| Async receive 异步接收 | An async receive returns immediately with a future value—for example, a [`CompletableFuture`](http://www.baeldung.com/java-completablefuture) in Java—that completes once a new message is available. 异步接收会立即返回一个 future 值（例如，Java 中的 a `CompletableFuture` ），该值在新消息可用时完成。 |

###### Listener 监听器

客户端库为使用者提供侦听器实现。例如，Java 客户端提供了一个 MesssageListener 接口。在此接口中，每当收到新消息时都会调用该 `received` 方法。

##### Reader 

在 Pulsar 中，“标准”消费者接口涉及使用消费者来监听主题、处理传入的消息，并最终在处理这些消息时确认这些消息。每当创建新订阅时，它最初都会定位在主题的末尾（默认情况下），与该订阅关联的使用者会从之后创建的第一条消息开始阅读。每当使用者使用预先存在的订阅连接到主题时，它都会从该订阅中未确认的最早消息开始读取。总之，在消费者接口中，订阅游标由 Pulsar 自动管理，以响应消息确认。

Pulsar 的读取器界面使应用程序能够手动管理游标。当您使用阅读器连接到主题时---而不是使用者---您需要指定阅读器在连接到主题时从哪条消息开始阅读。连接到主题时，通过阅读器界面，您可以开始：

- 主题中最早可用的消息。

  ```java
  import org.apache.pulsar.client.api.Message;
  import org.apache.pulsar.client.api.MessageId;
  import org.apache.pulsar.client.api.Reader;
  
  // Create a reader on a topic and for a specific message (and onward)
  Reader<byte[]> reader = pulsarClient.newReader()
      .topic("reader-api-test")
      .startMessageId(MessageId.earliest)
      .create();
  
  while (true) {
      Message message = reader.readNext();
  
      // Process the message
  }
  ```

- 主题中的最新可用消息。

  ```java
  Reader<byte[]> reader = pulsarClient.newReader()
      .topic(topic)
      .startMessageId(MessageId.latest)
      .create();
  ```

- 最早和最晚之间的一些其他消息。如果选择此选项，则需要显式提供消息 ID。您的应用程序将负责提前“知道”此消息 ID，可能从持久性数据存储或缓存中获取它。

  ```java
  // 建一个读取器，该读取器从最早和最晚之间的某些消息中读取
  byte[] msgIdBytes = // Some byte array
  MessageId id = MessageId.fromByteArray(msgIdBytes);
  Reader<byte[]> reader = pulsarClient.newReader()
      .topic(topic)
      .startMessageId(id)
      .create();
  ```

  

