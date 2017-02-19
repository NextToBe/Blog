#深入理解 ElasticSearch
节选并译自 [How to monitor Elasticsearch performance](https://www.datadoghq.com/blog/monitor-elasticsearch-performance-metrics/)，原作者 Emily Chang

##什么是 ElasticSearch ？

ElasticSearch（以下简称 ES） 是一款开源的分布式的存储搜索引擎，它可以近乎实时的存储和查询数据。ES 最早在2010 年由 [Shay Banon](https://github.com/kimchy)开源出来，它重度依赖于 Apache Lucene，一款使用 Java 开发的全文搜索引擎。

ES 将数据组织成 Json 形式的文档，并且可以通过 RESTful API 和各个语言的 Web 客户端（像 PHP，Python 和 Ruby）来进行全文搜索。之所以被称作 Elastic(有弹性的)是因为它可以很轻松的水平扩展它的节点。如今，许多公司包括*维基百科*， *Github* 和 *DataDog* 都在使用 ES 做大规模数据的存储、检索和分析。

####ElasticSearch 的组成元素

在我们开始探索性能标准(Performance metrics)之前，首先解释一下 ES 是如何工作的，在 ES 中，一个集群由以下部分组成：

![ES 集群](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/elasticsearch-diagram1a.png?ch=Width,DPR&w=660&auto=format)


每一个节点(node)都是一个单独运行 ES 实例，配置文件 *elasticsearch.yml* 指定了这个节点属于哪一个集群(*cluster.name*) 和这个节点的类型。在这个文件中的每一个设置项都可以通过命令行来完成。上图中的集群包括一个 master 节点和五个 data 节点。

ES 中最常见的三种节点类型：

* Master-Eligible 节点：默认情况下，除非指定了其他节点，否则每个节点都有资格做 master 节点。每一个 ES 集群会自动的从所有有资格的节点中选举出一个 master 节点。如果当前的 master 节点不幸遭遇了异常，比如断电，硬件问题或者是内存溢出错误，集群会再次选举出一个新的 master 节点。master 节点负责诸如***跨节点分配分片(shard)***和***创建或者删除索引***之类的集群作业的调度。每一个 Master-Eligible 节点都会想一个 Data 节点那样运作。然而，在一些大型的集群中，用户可能会通过配置 *node.data:false* 来上线一个不存储任何数据的节点。在高使用率的环境中，为了提高集群的稳定性，这样做可以保证总是会有足够的资源被分配给 master 节点处理那些只能由他来做的任务。
* Data 节点：默认情况下，每一个节点都会以分片的形式存储数据并且做一些和索引、搜索、聚合数据相关的事情。在一些大型的集群中，你可以在配置文件中设置 *node.master:false* 来创建专用数据节点，来保证这些节点拥有足够的资源去分配给与数据相关的请求中而不是给和集群相关的“行政”任务中。
* Client 节点：如果你同时设置了 *node.master:false* 和 *node.data:false*，你将会得到一个用来负责负载均衡和辅助索引检索数据的 client 节点。Client 节点帮助分担了一部分搜索的工作量，这样一来，data 节点和 master 节点可以专注于他们的核心工作。根据你的具体情况，Client 节点可能是没有必要的因为 data 节点可以自己处理请求路由。当然，如果你的工作量已经大到设置一个专用的 client 节点会使得你获得性能上的好处，这个时候一个 client 节点是有意义的。

####ElasticSearch 是如何组织数据的 ?

在 ES 中，相关的数据一般会存储在相同的**索引**中，每一个索引包含一个 Json 形式的文档的集合。ES 对于全文检索的秘密武器是 Lucene 的 [倒排索引](https://lucene.apache.org/core/3_0_3/fileformats.html#InvertedIndexing)(Inverted index，其实我觉得翻译成 *翻转索引* 更加恰当，但由于各大出版社都是倒排索引，故沿旧用)。当一个文档被索引的同时，ES 会自动为每一个字段创建倒排索引。这些倒排索引映射了每个条目到包含这些条目的文档中去。

每一个索引被存储在不同的主分片和不同的备份分片上。每一个分片都是一个完全的 Lucene 的实体，就像一个 mini 的搜索引擎。

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/elasticsearch-diagram1bb.png?ch=Width,DPR&w=660&auto=format)

在创建索引的时候，你可以指定主分片的数量(默认是 5)和每个主分片备份的数量(默认是 1)。主分片的数量一旦索引被创建就无法改变，所以请谨慎选择。 或者你可以选择后来重建索引。当然，备份的分片的数量是可以随意更改的。为了保护数据防止丢失，master 节点会保证每一份备份都不会被和它的主分片被分配在同一个节点上。

## ElasticSearch 的关键概念

####查询请求

查询请求是 ES 中两个主要请求之一，另一个是*索引请求*。这两个请求就像是传统数据库系统中的读写请求。ES 提供了与处理搜索请求的两个阶段相符的指标：查询和获取(query and fetch)。下图说明了一个搜索请求从头到尾的具体发生的事情：

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/search-diagram1.png?ch=Width,DPR&w=660&auto=format)

> 集群向 Node2 发送请求

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/search-diagram2.png?ch=Width,DPR&w=660&auto=format)
> Node2 向这个索引的所有分片(包括备份分片)发送查询的拷贝

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/search-diagram3.png?ch=Width,DPR&w=660&auto=format)
> 每个分片在本地执行查询并且把结果发送给 Node2。 Node2 排序并且把结果放进全局的一个优先队列。

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/search-diagram4.png?ch=Width,DPR&w=660&auto=format)
> Node2 找出所有的需要被取回的文档然后给所有相关的分片发送 GET 请求。

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/search-diagram5.png?ch=Width,DPR&w=660&auto=format)
> 每个分片加载文档并返回给 Node2

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/search-diagram6.png?ch=Width,DPR&w=660&auto=format)
> Node2 发送查询结果给客户端。

###索引请求

索引请求与传统数据库的写入请求十分类似。如果你的 ES 的工作写入占有较大比重的话，那么如何高效的更新索引是十分重要的。让我们来探索一下 ES 是如何更新一个索引的。每当新的文档被索引到一个索引，或者一个存在的文档被更新或者删除，每个索引中的分片通过两个步骤完成更新操作：***refresh*** 和 ***flush***

####索引的 Refresh

最新被索引的文档是不会立刻可以被搜索到的。首先，它们会被写入到内存中的一个缓存区，之后会等待下一次索引 refresh 的到来。refresh 默认是每秒一次。这个过程会用缓存区中的内容在内存中创建一个 ***段(Segment)***, 这样一来，最新被索引的文档就可以被搜索到了。之后，会清空缓存区，如下图：

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/elasticsearch-diagram2a.png?ch=Width,DPR&w=660&auto=format)
> 索引 refresh 过程

####段(Segments)上特殊的段

一个索引的分片是由多个段组成的，段是 Lucene 的核心数据结构，它本质上是对一个索引的所有改变的集合。 这些段会在每次索引的 refresh 过程中被创建，随着时间的推移，这些段会在后台被合并以便资源的有效利用。

段是 mini 的倒排索引，每当一个搜索一个索引, 一个主分片或者备份分片的所有段都会被搜索。

段是一个不可变的集合，所以每次更新一个文档意味着两件事情：

* 在一次索引刷新过程中将信息写入新的段中
* 把旧的数据标记为已删除

当一个过时的段被合并到另一个段的时候，老旧的信息将被永久删除。

####索引 Flush

在新的文档被添加到内存中的缓存区的同时，这些文档也被追加到了分片的事务日志(translog)中。事务日志是一个记录所有操作的持久的，预写的日志。每隔 30 分钟，或者每当事务日志达到了最大的尺寸(默认是 512MB)就会触发 flush 操作。在 flush 过程中，任何在缓存区中的文档都会被刷新(存储到新的段中)，之后所有的段都会被提交到磁盘，事务日志随后被清空。

flush 过程说明如下图：

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/elasticsearch-diagram2b.png?ch=Width,DPR&w=660&auto=format)

> flush 操作过程

### 内存使用和垃圾收集

#### JVM 堆内存

ES 强调 JVM堆大小的重要性，你不会希望把它设置的过大或者过小，原因下面会讲。一般情况下，ES 的经验法则是分配给 JVM堆不多于 50% 的机器物理内存大小，并且永远不要超过 32GB。

分配给ES越少的堆内存, Lucene 仍然可以使用内存就越多，这大大依赖于文件系统缓存来快速地提供请求。 当然，你也不会想把堆内存设置的太小因为你会遇到内存溢出错误或者会在应用因为垃圾回收持续的短暂停时会使得吞吐量降低。

ES 在 JVM堆大小这一项的默认设置是 1GB，这对于绝大多数使用场景来说不够的，你可以这样更改并重启 ES 使之生效：

	$ export ES_HEAP_SIZE=10g


#### 垃圾收集

ES 依赖垃圾收集机制来释放堆内存，当JVM 的堆内存使用率达到 75% 时就会触发垃圾收集。因为垃圾收集会消耗资源(虽然是为了释放资源)，所以必须注意垃圾收集的频率并且据此调整堆内存的大小。将堆内存设置的过大会导致垃圾收集的时间过长，这无疑是非常危险的因为这会导致你的集群当有节点掉线的时候对节点注册错误。

![](https://datadog-live.imgix.net/img/blog/monitor-elasticsearch-performance-metrics/pt1-1-jvm-heap.png?ch=Width,DPR&w=660&auto=format)

在 JVM 停止执行程序二区收集垃圾对象的这段时间里，节点无法完成任何任务。master 节点每30 秒检查一次其他节点，如果一个节点的垃圾回收时间开销超过30s，就会导致 master 节点判断出这个节点已经无法联通。

####内存使用

就像上面所提到的，ES 善于使用那些没有分配给 JVM 堆的内存。和 Kafka 一样，ES 天生就依赖于操作系统的文件系统缓存并因此服务地更加出色。

有那么几个变量能反映出 ES 是否成功地从文件系统缓存中读到了数据。如果 ES刚刚把段文件写入了磁盘，那么这个段就会出现在缓存里面。然而，如果有节点已经被关闭或是重新启动，当一个段首次被查询的时候，系统更有可能从磁盘中加载信息，这就是为什么要保持集群稳定不宕机的一个重要原因。

通常来说，监控节点的内存使用情况十分重要，并且需要分配给 ES 尽可能多的 RAM，从而提高 ES 的性能。

#### 集群健康与节点可用性

**集群状态**：当一个集群状态为**黄色**时，至少有一个备份分片没有被分配或者已经丢失了。查询结果依然回事完整的，但是如果有更多的分片丢失的话，将会丢失数据。

一个**红色**的集群状态表明了至少有一个主分片丢失，这意味着此时的查询结果不是完整的，只有其中的一部分数据。并且此时集群拒绝向丢失的分片中索引任何数据。

**初始化和未分配的分片**：当一个索引被首次创建，或者重启一个节点时，在master 节点试图分配集群中的分片的时候，这些分片在变为“已启动”或者“未分配”状态之前会短暂地拥有“正在初始化”的状态。如果你发现分片保持“正在初始化”状态时间过长，这说明集群可能处于不稳定状态。

####资源饱和与错误

ES 使用线程池来管理线程。自从线程池根据处理器的数量自动配置以来，通常情况下去微调它是没有任何意义的。但是，多留意队列里的请求和拒绝的请求从而判断当前集群的请求处理能力是否能跟上是非常必要的。

线程池的队列大小代表了当前正在等待被服务的请求个数。队列允许节点跟踪并最终处理这些请求而不是丢弃它们。线程池开始拒绝表示当当前线程池超过了队列大小。


