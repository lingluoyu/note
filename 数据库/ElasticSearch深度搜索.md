#### ES三种分页方式

- from&size

​	顶部查询，查询10000以内的文档

​	eg：查询最新订单，信息流

- Scroll滚动游标

​	深度分页，非实时查询场景

​	eg：多出全部数据

- search after

​	深度分页，实时查询场景

##### from&size分页

```
POST movies/_search
{
  "from": 10000,
  "size": 10,
  "query": {
    "match_all": {

    }
  }
}
```

查询10000到10010文档

由于ES限制了最大获取文档为10000，此命令会报错

```
"root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Result window is too large, from + size must be less than or equal to: [10000] but was [10001]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting."
      }
    ],
    "type": "search_phase_execution_exception",
```

> 由于from&size方式在深度分页时性能不好，所以限制了分页深度，es目前支持的最大的 max_result_window = 10000

调整window

```
PUT movies/_settings
{ 
    "index" : { 
        "max_result_window" : 20000
    }
}
```

此种方式无法完全解决问题，随着数据量越来越大，分页越来越深，会出现OOM问题

**OOM 原因**

客户点进行查询**from+size**时，请求先进入到协调节点，油协调节点发送到分片

每个分片都查询出from+size个结果，返回给协调节点

协调节点进行结果合并，若分片为n，则最终查询的数据为 n * (from+size) 若from很大，会导致网络资源浪费甚至OOM

##### scroll 滚动游标分页(查询快照)

为了避免深度分页，ES不允许使用from+size查询10000条以后的数据

scroll滚动游标：

​	对一次查询生成一个**scroll_id**，后续查询只需要根据 **scroll_id** 游标取数据，直至返回的结果 hits 为空，表示所有符合条件的结果查询完毕。

scroll_id 生成时建立了一个临时快照，生成之后，原 doc 文件增删改查不会影响快照结果

```
# curl -XGET 200.200.107.232:9200/product/info/_search?pretty&scroll=2m -d 
'{"query":{"match_all":{}}, "size": 10, "sort": ["_doc"]}'

# 返回结果
{
  "_scroll_id": "cXVlcnlBbmRGZXRjaDsxOzg3OTA4NDpTQzRmWWkwQ1Q1bUlwMjc0WmdIX2ZnOzA7",
  "took": 1,
  "timed_out": false,
  "_shards": {
  "total": 1,
  "successful": 1,
  "failed": 0
  },
  "hits":{...}
}
```

scroll=2m：

> 表示 srcoll_id 的生存期为2分钟。scroll就是把一次的查询结果缓存一定的时间，如scroll=2m则把查询结果在下一次请求上来时暂存2分钟，response比传统的返回多了一个scroll_id，下次带上这个scroll_id即可找回这个缓存的结果。

后续翻页， 通过上一次查询返回的scroll_id 来不断的取下一页，请求指定的 scroll_id 时就不需要 /index/_type 等信息了。

如果srcoll_id 的生存期很长，那么每次返回的 scroll_id 都是一样的，直到该 scroll_id 过期，才会返回一个新的 scroll_id。

每读取一页都会重新设置 scroll_id 的生存时间，所以这个时间只需要满足读取当前页就可以，不需要满足读取所有的数据的时间，1 分钟足以。

**scroll__id 清理**

srcoll_id 的存在会耗费大量的资源来保存一份当前查询结果集映像，并且会占用文件描述符。

为了防止因打开太多scroll而导致的问题，不允许用户打开滚动超过某个限制。

默认情况下，打开的滚动的最大数量为500.可以使用search.max_open_scroll_context群集设置更新此限制。

##### search after 深度分页

scroll 方式查询，官方不建议用于实时文档查询

- 因为 scroll_id 生成的历史快照，对于数据的变更不会反应到快照上
- scroll 方式往往用于非实时处理大量数据的情况，比如要进行数据迁移或者索引变更之类的。

实时查询情况下需要深度分页？

es 5.0 版本之后提供了 **search after** 方式

search after 方式是根据上一页的最后一条数据来确定下一页的位置，同时在分页查询过程中，如果数据有增删改查，也会实时反应到游标上。

为了找到每一页最后一条数据，每个文档必须有一个全局唯一值，官方推荐使用 _id 做为全局唯一值，也可以使用业务 id 。

search after 使用

第一页查询，与正常请求一样

```
curl -XGET 127.0.0.1:9200/order/info/_search
{
    "size": 10,
    "query": {
        "term" : {
            "did" : 519390
        }
    },
    "sort": [
        {"date": "asc"},
        {"_id": "desc"}
    ]
}
```

第二页查询，使用第一页返回结果的最后一个数据的值，加上 searh after 字段

> 使用 search_after 的时候要将 from 置为 0 或 -1

```
curl -XGET 127.0.0.1:9200/order/info/_search
{
    "size": 10,
    "query": {
        "term" : {
            "did" : 519390
        }
    },
    "search_after": [1463538857, "tweet#654323"],
    "sort": [
        {"date": "asc"},
        {"_uid": "desc"}
    ]
}
```

> search_after 适用于深度分页+ 排序，因为每一页的数据依赖于上一页最后一条数据，所以无法跳页请求。
>
> 且返回的始终是最新的数据，在分页过程中数据的位置可能会有变更。

##### 总结

- 传统方式（from&size）

  需要实时获取顶部的部分文档。例如查询最新的订单。

- Scroll

  用于非实时查询

  需要全部文档，例如导出全部数据

- Search After

  用于实时查询

  需要做到深度分页

##### ES查询方案

1. 产品侧：在大量数据分页时，产品层面不允许用户自行输入页码跳页
2. 技术侧：ES数据有全局唯一且有序的ID，可以通过相关算法生成

**核心思想**

经分析，用户的点击跳页行为数据偏移量是有限的。且第1页和最后1页的跳转行为可以有效识别，那么可以将用户的点击查询行为分为几个场景：

用户点击第1页：可以按照正常的逻辑进行查询，根据数据要求的排序方向进行顺序查询或倒序查询；

用户点击最后1页：首先判断是否是最后1页；重新计算页数据展示大小；如果结果是按顺序展示，则先按倒序方向进行数据查询，再将结果进行翻转展示。如果结果按倒序展示反之操作即可；

用户点击其他页：此种情况较为复杂。因为要重新计算数据的偏移量，所以需要有用户上次点击的页码信息（例如上次点击页码及对应的第1条数据ID和最后1条数据ID等）进行参照计算。

