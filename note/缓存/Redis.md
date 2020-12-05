## Redis核心对象

![redisObject对象](https://raw.githubusercontent.com/lingluoyu/image/master/img/20191210155928.png)

redis内部使用一个redisObject对象来表示所有的key和value，redisObject最主要的信息如上图所示：type表示一个value对象具体是何种数据类型，encoding是不同数据类型在redis内部的存储方式。比如：type=string表示value存储的是一个普通字符串，那么encoding可以是raw或者int。

## 基础数据结构

> ### String 
>
> 最大value值为512MB
>
> *  **缓存功能**：**String**字符串是最常用的数据类型，不仅仅是**Redis**，各个语言都是最基本类型，因此，利用**Redis**作为缓存，配合其它数据库作为存储层，利用**Redis**支持高并发的特点，可以大大加快系统的读写速度、以及降低后端数据库的压力。 
> *  **计数器**：许多系统都会使用Redis作为系统的实时计数器，可以快速实现计数和查询的功能。而且最终的数据结果可以按照特定的时间落地到数据库或者其它存储介质当中进行永久保存。
> * **共享用户session**：用户重新刷新一次界面，可能需要访问一下数据进行重新登录，或者访问页面缓存Cookie，但是可以利用Redis将用户的Session集中管理，在这种模式只需要保证Redis的高可用，每次用户Session的更新和获取都可以快速完成。大大提高效率。

* String命令

```shell
> set mykey somevalue #set值
OK
> get mykey	#获取已存入的值
"somevalue"
```

```shell
> set mykey newval nx	#mykey不存在时set
(nil)
> set mykey newval xx	#mykey存在时set
OK
```

```shell
> set counter 100	
OK
> incr counter	#incr：counter加1
(integer) 101
> incr counter	#incr：counter加1
(integer) 102
> incrby counter 50	#incrby：counter增加指定值
(integer) 152
```

```shell
> mset a 10 b 20 c 30	#一次set多个key-value
OK
> mget a b c	#一次获取多个key的值
1) "10"
2) "20"
3) "30"
```

```
> set mykey hello
OK
> exists mykey	#是否存在key
(integer) 1
> del mykey
(integer) 1
> exists mykey
(integer) 0
```

```shell
> set mykey x
OK
> type mykey	#key的类型
string
> del mykey	#删除key
(integer) 1
> type mykey
none
```

```shell
> set key some-value
OK
> expire key 5	#设置过期时间，秒
(integer) 1
> get key (immediately)
"some-value"
> get key (after some time)
(nil)
```

```shell
> set key 100 ex 10	#set值同时设置过期时间
OK
> ttl key	#查看key剩余过期时间
(integer) 9
```

> ### Hash
>
> 这个是类似 Map 的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是这个对象没嵌套其他的对象）给缓存在 Redis 里，然后每次读写缓存的时候，可以就操作 Hash 里的某个字段。
> 但是这个的场景其实还是多少单一了一些，因为现在很多对象都是比较复杂的，比如你的商品对象可能里面就包含了很多属性，其中也有对象。

* Hash命令

```
> hmset user:1000 username antirez birthyear 1977 verified 1
OK
> hget user:1000 username
"antirez"
> hget user:1000 birthyear
"1977"
> hgetall user:1000
1) "username"
2) "antirez"
3) "birthyear"
4) "1977"
5) "verified"
6) "1"
> hmget user:1000 username birthyear no-such-field
1) "antirez"
2) "1977"
3) (nil)
> hincrby user:1000 birthyear 10
(integer) 1987
> hincrby user:1000 birthyear 10
(integer) 1997
```

> ### List
>
> * 消息队列：Redis的链表结构，可以轻松实现阻塞队列，可以使用左进右出的命令组成来完成队列的设计。比如：数据的生产者可以通过Lpush命令从左边插入数据，多个数据消费者，可以使用BRpop命令阻塞的“抢”列表尾部的数据。
>
>   
>
> * 文章列表或者数据分页展示的应用。
>
>   比如，我们常用的博客网站的文章列表，当用户量越来越多时，而且每一个用户都有自己的文章列表，而且当文章多时，都需要分页展示，这时可以考虑使用Redis的列表，列表不但有序同时还支持按照范围内获取元素，可以完美解决分页查询功能。大大提高查询效率。

* List命令

```
> rpush mylist A	#尾部插入数据
(integer) 1
> rpush mylist B
(integer) 2
> lpush mylist first	#头部插入数据
(integer) 3
> lrange mylist 0 -1	#从list中取出元素	-1为最后一个元素
1) "first"
2) "A"
3) "B"
```

```
> rpush mylist a b c
(integer) 3
> rpop mylist	#从list中取出列尾元素，并清除
"c"
> rpop mylist
"b"
> lpop mylist	#从list中取出列头元素，并清除
"a"
```

```
> brpop tasks 5	#移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
1) "tasks"		#假如在指定时间内没有任何元素被弹出，则返回一个 nil 和等待时长。 反之，返回一个含有两个元素的列表，第一个元素是被弹出元素所属的 key ，第二个元素是被弹出元素的值。
2) "do_something"
```

> ### Set
>
> Set 是无序集合，会自动去重的那种。
>
> 直接基于 Set 将系统里需要去重的数据扔进去，自动就给去重了，如果你需要对一些数据进行快速的全局去重，你当然也可以基于 JVM 内存里的 HashSet 进行去重，但是如果你的某个系统部署在多台机器上呢？得基于Redis进行全局的 Set 去重。
>
> 可以基于 Set 玩儿交集、并集、差集的操作，比如交集吧，我们可以把两个人的好友列表整一个交集，看看俩人的共同好友是谁。

* set命令

```
> sadd myset 1 2 3
(integer) 3
> smembers myset	#查看元素，每次都是无序的
1. 3
2. 1
3. 2
> sismember myset 3	#验证元素是否存在
(integer) 1
> sismember myset 30
(integer) 0
```

```
> sadd news:1000:tags 1 2 5 77	#关联tag
(integer) 4
> sadd tag:1:news 1000	#反向关联，tag关联news
(integer) 1
> sadd tag:2:news 1000
(integer) 1
> sadd tag:5:news 1000
(integer) 1
> sadd tag:77:news 1000
(integer) 1
> smembers news:1000:tags	#获取所有tags
1. 5
2. 1
3. 77
4. 2
```



> ### Sorted Set
>
> 有序集合的使用场景与集合类似，但是set集合不是自动有序的，而Sorted set可以利用分数进行成员间的排序，而且是插入时就排序好。所以当你需要一个有序且不重复的集合列表时，就可以选择Sorted set数据结构作为选择方案。
>
> * 排行榜：有序集合经典使用场景。例如视频网站需要对用户上传的视频做排行榜，榜单维护可能是多方面：按照时间、按照播放量、按照获得的赞数等。
> * 用Sorted Sets来做带权重的队列，比如普通消息的score为1，重要消息的score为2，然后工作线程可以选择按score的倒序来获取工作任务。让重要的任务优先执行。

* Sorted Set命令

```
> zadd hackers 1940 "Alan Kay"
(integer) 1
> zadd hackers 1957 "Sophie Wilson"
(integer) 1
> zadd hackers 1953 "Richard Stallman"
(integer) 1
> zadd hackers 1949 "Anita Borg"
(integer) 1
> zadd hackers 1965 "Yukihiro Matsumoto"
(integer) 1
> zadd hackers 1914 "Hedy Lamarr"
(integer) 1
> zadd hackers 1916 "Claude Shannon"
(integer) 1
> zadd hackers 1969 "Linus Torvalds"
(integer) 1
> zadd hackers 1912 "Alan Turing"
(integer) 1
```

```
> zrange hackers 0 -1	#返回所有元素，有序（插入顺序）
1) "Alan Turing"
2) "Hedy Lamarr"
3) "Claude Shannon"
4) "Alan Kay"
5) "Anita Borg"
6) "Richard Stallman"
7) "Sophie Wilson"
8) "Yukihiro Matsumoto"
9) "Linus Torvalds"
```

```
> zrevrange hackers 0 -1	#返回所有元素，反向排序
1) "Linus Torvalds"
2) "Yukihiro Matsumoto"
3) "Sophie Wilson"
4) "Richard Stallman"
5) "Anita Borg"
6) "Alan Kay"
7) "Claude Shannon"
8) "Hedy Lamarr"
9) "Alan Turing"
```

```
> zrange hackers 0 -1 withscores	#带有权值
1) "Alan Turing"
2) "1912"
3) "Hedy Lamarr"
4) "1914"
5) "Claude Shannon"
6) "1916"
7) "Alan Kay"
8) "1940"
9) "Anita Borg"
10) "1949"
11) "Richard Stallman"
12) "1953"
13) "Sophie Wilson"
14) "1957"
15) "Yukihiro Matsumoto"
16) "1965"
17) "Linus Torvalds"
18) "1969"
```

```
> zrangebyscore hackers -inf 1950	#返回score小于1950的元素
1) "Alan Turing"
2) "Hedy Lamarr"
3) "Claude Shannon"
4) "Alan Kay"
5) "Anita Borg"
```

```
> zremrangebyscore hackers 1940 1960	#移除score在1940-1960的元素
(integer) 4
```

```
> zrank hackers "Anita Borg"	#返回元素在有序集合中的位置
(integer) 4
```



> ### Bitmap
> 位图是支持按 bit 位来存储信息，可以用来实现 布隆过滤器（BloomFilter）；



> ### HyperLogLog
> 供不精确的去重计数功能，比较适合用来做大规模数据的去重统计，例如统计 UV；

> ### Geospatial
> 可以用来保存地理位置，并作位置距离计算或者根据半径计算位置等。有没有想过用Redis来> > 实现附近的人？或者计算最优地图路径？
> 这三个其实也可以算作一种数据结构，不知道还有多少朋友记得，我在梦开始的地方，Redis> > 基础中提到过，你如果只知道五种基础类型那只能拿60分，如果你能讲出高级用法，那就觉得> 你有点东西。

> ### pub/sub
> 功能是订阅发布功能，可以用作简单的消息队列。 

> ### Pipeline
> 可以批量执行一组指令，一次性返回全部结果，可以减少频繁的请求应答。

> ### Lua
> Redis 支持提交 Lua 脚本来执行一系列的功能。

### 缓存雪崩

* 数据缓存，同时失效
* 所有数据查询请求落在数据库

![缓存雪崩](https://raw.githubusercontent.com/lingluoyu/image/master/img/20191210143147.png)

### 缓存穿透

* 数据库和缓存中都不存在的数据

![缓存穿透](https://raw.githubusercontent.com/lingluoyu/image/master/img/20191210143331.png)

### 缓存击穿

* 一个Key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个Key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个完好无损的桶上凿开了一个洞。

#### 解决方案

* 在接口层增加校验，比如用户鉴权校验，参数做校验，不合法的参数直接代码Return，比如：id 做基础校验，id <=0的直接拦截等。 

#### 布隆过滤器（Bloom Filter）

* Bloom filter 是由 Howard Bloom 在 1970 年提出的二进制向量数据结构，它具有很好的空间和时间效率，被用来检测一个元素是不是集合中的一个成员。如果检测结果为是，该元素不一定在集合中；但如果检测结果为否，该元素一定不在集合中。因此Bloom filter具有100%的召回率。这样每个检测请求返回有“在集合内（可能错误）”和“不在集合内（绝对不在集合内）”两种情况，可见 Bloom filter 是牺牲了正确率和时间以节省空间。

* 海量数据查重

![海量数据查重](https://raw.githubusercontent.com/lingluoyu/image/master/img/20191210143246.png)

* 避免缓存穿透

![避免缓存穿透](https://raw.githubusercontent.com/lingluoyu/image/master/img/20191210143218.png)