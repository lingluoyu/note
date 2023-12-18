## 简介

RabbitMQ是流行的开源消息队列系统，是AMQP（Advanced Message Queuing Protocol高级消息队列协议）的标准实现，用erlang语言开发。

### 安装

#### 安装Erlang

安装依赖

```shell
yum install -y make gcc gcc-c++ m4 openssl openssl-devel ncurses-devel unixODBC unixODBC-devel java java-devel socat
```

下载erlang的rpm包

```shell
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v23.1.5/erlang-23.1.5-1.el8.x86_64.rpm
```

安装erlang

```shell
yum localinstall erlang-23.1.5-1.el8.x86_64.rpm
```

安装成功后，输入一下命令查看版本

```shell
erl -version
```

#### 安装RabbitMQ

下载RabbitMQ的rpm包

```shell
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.9/rabbitmq-server-3.8.9-1.el7.noarch.rpm
```

安装RabbitMQ

```shell
yum localinstall rabbitmq-server-3.8.8.el7.noarch.rpm
```

安装完毕，启动RabbitMQ

```shell
systemctl start rabbitmq-server
```

其他命令

```shell
# 添加web管理插件
rabbitmq-plugins enable rabbitmq_management
#添加新用户，用户名为"root"，密码为"root"
rabbitmqctl add_user root root
#为root用户设置所有权限
rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
#设置用户为管理员角色
rabbitmqctl set_user_tags root administrator
```

**RabbitMQ默认端口**

`4369`：erlang发现口

`5672`：client端通信口，客户端要连接RabbitMQ服务时要用到

`15672`：后台管理界面ui端口，进入管理后台时访问url如：http://localhost:15672/

`25672`：server间内部通信口

### 集群搭建

准备三台虚拟机，并配置安装好RabbitMQ

#### 修改hostname

```shell
192.168.138.130 $ hostname mqnode1
192.168.138.131 $ hostname mqnode2
192.168.138.132 $ hostname mqnode3
```

#### 修改 hosts

```shell
$ vim /etc/hosts
#内容如下
192.168.138.130 mqnode1
192.168.138.131 mqnode2
192.168.138.132 mqnode3
```

确保三台虚拟机上的RabbitMQ都可以正常启动，执行如下步骤：

#### 停止RabbitMQ

```shell
systemctl stop rabbitmq-server
```

#### 配置Erlang Cookie

RabbitMQ利用erlang的分布式特性组建集群，erlang集群通过magic cookie实现，此cookie保存在$home/.erlang.cookie，这里即：/var/lib/rabbitmq/.erlang.cookie，需要保证集群各节点的此cookie一致，可以选取一个节点的cookie，采用scp同步到其余节点。

```shell
[root@mqnode1 ~]# scp /var/lib/rabbitmq/.erlang.cookie root@192.168.138.131:/var/lib/rabbitmq/
[root@mqnode1 ~]# scp /var/lib/rabbitmq/.erlang.cookie root@192.168.138.132:/var/lib/rabbitmq/
```

#### 使用-detached参数启动节点

```bash
#mqnode2/3因更换了.erlang.cookie，使用此命令会无效并报错，可以依次采用“systemctl stop rabbitmq-server”停止服务，“systemctl start rabbitmq-server”启动服务，最后再“rabbitmqctl stop”
[root@mqnode1 ~]# rabbitmqctl stop
[root@mqnode1 ~]# rabbitmq-server -detached
```

#### 组建集群

*在RabbitMQ集群里，必须至少有一个磁盘节点存在。*

```shell
#“rabbitmqctl join_cluster rabbit@mqnode1”中的“rabbit@mqnode1”，rabbit代表集群名，mqnode1代表集群节点；mqnode2与mqnode3均连接到mqnode1，它们之间也会自动建立连接。
#如果需要使用内存节点，增加一个”--ram“的参数即可，如“rabbitmqctl join_cluster --ram rabbit@mqnode1”，一个集群中至少需要一个”disk”节点 
[root@mqnode2 ~]# rabbitmqctl stop_app
[root@mqnode2 ~]# rabbitmqctl join_cluster rabbit@mqnode1
[root@mqnode2 ~]# rabbitmqctl start_app

[root@mqnode3 ~]# rabbitmqctl stop_app
[root@mqnode3 ~]# rabbitmqctl join_cluster rabbit@mqnode1
[root@mqnode3 ~]# rabbitmqctl start_app
```

```shell
#如果节点已是"disk"节点，可以修改为内存节点
[root@mqnode3 ~]# rabbitmqctl stop_app
[root@mqnode3 ~]# rabbitmqctl change_cluster_node_type ram
[root@mqnode3 ~]# rabbitmqctl start_app
```

#### 查看集群状态

```shell
[root@rmq-node1 ~]# rabbitmqctl cluster_status
```

#### 设置镜像队列高可用

到目前为止，集群虽然搭建成功，但只是默认的普通集群，exchange，binding等数据可以复制到集群各节点。

但对于队列来说，各节点只有相同的元数据，即队列结构，但队列实体只存在于创建改队列的节点，即队列内容不会复制（从其余节点读取，可以建立临时的通信传输）。

这样此节点宕机后，其余节点无法从宕机节点获取还未消费的消息实体。如果做了持久化，则需要等待宕机节点恢复，期间其余节点不能创建宕机节点已创建过的持久化队列；如果未做持久化，则消息丢失。

```shell
#任意节点执行均可如下命令：将所有队列设置为镜像队列，即队列会被复制到各个节点，各个节点状态保持一直；
#可通过命令查看：rabbitmqctl list_policies；
[root@mqnode1 ~]# rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

#### 注意事项

1. cookie在所有节点上必须完全一样，同步时注意；
2. erlang是通过主机名来连接服务，必须保证各个主机名之间可以ping通，可以通过编辑/etc/hosts来手工添加主机名和IP对应关系，如果主机名ping不通，rabbitmq服务启动会失败；
3. 如果queue是非持久化queue，则如果创建queue的那个节点失败，发送方和接收方可以创建同样的queue继续运作；如果是持久化queue，则只能等创建queue的那个节点恢复后才能继续服务。
4. 在集群元数据有变动的时候需要有disk node在线，但是在节点加入或退出时，所有的disk node必须全部在线；如果没有正确退出disk node，集群会认为这个节点宕掉了，在这个节点恢复之前不要加入其它节点。

#### 网络分区问题

**原因**

如果其他节点无法连接该节点的时间达到1分钟以上（net_ticktime设定），则Mnesia判定某个其他节点失效。当这两个连接失效的节点恢复连接状态时，都会认为对端已down 掉，此时Mnesia将会判定发生了网络分区，这种情况会被记录进 RabbitMQ 的日志文件中。

导致网络分区的原因有很多，常见如下：

1. **网络本身的原因；**
2. **挂起与恢复节点也会导致网络分区，最常见与节点本身是vm，而虚拟化操作系统的监控程序便有挂起vm的功能；**
3. **vm迁移（飘移）也会导致虚拟机挂起。**

发生网络分区时，各分区均能够独立运行，同时认为其余节点（分区）已处于不可用状态。其中 queue、binding、exchange 均能够在各个分区中创建和删除；当由于网络分区而被割裂的镜像队列，最后会演变成每个分区中产生一个 master ，并且每一个分区均能独立进行工作，其他未定义和奇怪的行为也可能发生。

**恢复**

*手工处理*

1. 选择一个可信分区；
2. 对于其余非可信分区的节点，停止服务，重新加入集群，此操作会导致非信任分区内的操作丢失；
3. 重启信任分区内的节点，消除告警（此点是官网上明确的，但**经实际验证似乎不需要**）。

\#更简单直接的方式是：重启集群所有节点，但需要保证重启的第一个节点是可信任的节点。

\#同时也可以观察3个节点的日志。

*自动处理*

RabbitMQ提供3种自动处理方式：pause-minority mode, pause-if-all-down mode and autoheal mode；默认的行为是ignore mode，不做处理，此模式适合网络非常稳定的场景。

1. pause-minority：此模式下，如果发现有节点失效，RabbitMQ将会自动停止**少数派集群（即少于或等于半数节点数）**中的所有节点。这种策略选择了CAP 理论中的分区容错性（P），而放弃了可用性（A），保证了在发生网络分区时，**最多只有一个分区中的节点会继续工作**；而处于少数派集群中的节点将在分区发生的开始就被停止，在分区恢复后重新启动（继续运行但不监听任何端口或做其他工作，其将每秒检测一次集群中的其他节点是否可见，若可见则从pause状态唤醒）。对于节点挂起引起的网络分区，此模式无效，因为挂起节点不能看到其余节点是否恢复"可见"，因而不能触发从cluster中断开。
2. pause-if-all-down：此模式下，RabbitMQ会自动停止集群中的节点，只要某节点与列举出来的任何节点之间无法通信。这与pause-minority模式比较接近，但该模式**允许管理员来决定采用哪些节点做判定**；可能存在列举出来的多个节点本身就处于无法通信的不同分区中，此时不会有任何节点被停掉。
3. autoheal：此模式下，RabbitMQ将在发生网络分区时，**自动决定出一个胜出分区**（获胜分区是获得最多客户端连接的那个分区；如果产生平局，则选择拥有最多节点的分区；如果仍是平局，则随机选择），并重启不在该分区中的所有节点。与pause_minority模式不同的是，autoheal 模式是**在分区结束阶段（已经形成稳定的分区）时起作用**，而不是在分区开始阶段（刚刚开始分区）。此模式**适合更看重服务的可持续行胜于数据完整性的场景**。

自动处理配置文件：/etc/rabbitmq/rabbitmq.conf，"cluster_partition_handling"配置项，可配置参数如下：

**pause_minority**

**{pause_if_all_down, [nodes], ignore | autoheal}**

**autoheal**

### 常用命令

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