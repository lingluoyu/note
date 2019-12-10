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