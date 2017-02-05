#ElasticSearch 集群的安装与配置
***

###ElasticSearch是一个开源搜索服务框架，它已经成为搜索解决方案领域的重要成员。ElasticSearch还经常被用作文档数据库，这主要得益于它的分布式特性和实时搜索能力，另外，ElasticSearch支持越来越多的聚合功能，而且和Yarn、Hadoop、Hive、Pig、Spark、Flume等大数据处理框架的兼容性越来越好。毕设需要使用到 ES 作为实时搜索的引擎，于是在这里记录一下安装与配置的过程。

##安装环境
	1、九台 linux 服务器（1.2.3.1, 1.2.3.2, 1.2.3.3 ...），内存32GB
	2、Java 版本：1.8.0_72
	3、ElasticSearch 版本：5.1.2
这里需要注意的是，在安装 ES 之前必须首先安装 Java，因为 ES 是用 Java 语言实现的，也跑在 JVM 之上。

##安装
* 各个节点分别[下载](https://www.elastic.co/downloads/elasticsearch)安装包，并解压到对应目录。
* 转到相应目录，运行 ```./bin/elasticsearch``` 进行安装，这里需要注意的是，不能用 root 账户或者 sudo 来执行，否则会出现如下异常：
```java.lang.RuntimeException: don't run elasticsearch as root.```
此时有两种选择：1，创建专有账号 es。2，将本文件夹归登录账号所有 (非 root 账号)

##遇到的问题
安装完毕之后，有可能会出现 ES 无法启动的情况，一般问题会有这么几种：

	ERROR: bootstrap checks failed
	max file descriptors [10240] for elasticsearch process likely too low, increase to at least [65536]
	max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]
	max number of threads [1024] for user [elsearch] likely too low, increase to at least [2048]

对第一条，这条限制了 ELasticSearch 的打开的最大文件数，解决方法：

```sudo vi /etc/security/limits.conf```

添加如下内容：

	* soft nofile 65536

	* hard nofile 131072

	* soft nproc 2048

	* hard nproc 4096
	
对第二条，限制了 JVM最大线程数量，解决方法：

```sudo sysctl -w vm.max_map_count=262144```

对第三条， 限制了用户的最大线程数量，解决方法如下：

```sudo vi /etc/security/limits.d/90-nproc.conf```

修改如下内容：

	#原内容
	* soft nproc 1024
	#修改为
	* soft nproc 2048
然后退出当前用户，重新登录之后改变方可生效。


##ElasticSearch 的配置

ElasticSearch/config/ 里有两个配置文件：elasticsearch.yml 和 jvm.options，其中，jvm.options 顾名思义，用来配置 ES 使用 jvm 资源的一些配置，而 elasticsearch.yml 则是 ES 本身的配置。
```vi jvm.options```

将 Xms 和 Xmx设置成相同的值，官方建议最大值为机器内存的一半。

	# Xms represents the initial size of total heap space
	# Xmx represents the maximum size of total heap space

	-Xms8g
	-Xmx8g
	

然后看 ES 的常规配置
```vi elasticsearch.yml```

一般来说我们要配置的有如下几项：

	# Use a descriptive name for your cluster:
	cluster.name: gDelt
因为 ES 会对加入其集群的节点自动发现，使用的便是这个集群名称，所以，一个集群的节点应当使用同一个名称。所以请不要再不同的环境中使用相同的集群名称，否则会使得节点加入了错误的集群中。

	# Use a descriptive name for the node:
	node.name: gDelt_1
修改加入集群中的节点的默认名称，否则 ES 会给当前节点随机起名。

	# Path to directory where to store the data (separate multiple locations by comma):
	path.data: /data/elasticsearch
	# Path to log files:
	path.logs: /var/log/elasticsearch
	
ES 保存数据的路径和日志的路径


	# Set the bind address to a specific IP (IPv4 or IPv6):
	network.host: 0.0.0.0
	# Set a custom port for HTTP:
	http.port: 9200
配置绑定的 IP 地址和端口号，默认的 IP 地址只能使用 localhost 访问，设置为 ```0.0.0.0```之后便可以通过 IP地址访问了。

	# Pass an initial list of hosts to perform discovery when new node is started:
	# The default list of hosts is ["127.0.0.1", "[::1]"]
	discovery.zen.ping.unicast.hosts: []
	
```zen discovery```是 ES 内置的自动发现功能，该模块主要负责集群中节点的自动发现和 Master 节点的选举, 将集群中的节点加入列表一遍 ES 自动建立集群。

***
在所有的节点上都完成上述步骤之后，测试一下！

```curl -XGET 1.2.3.4:9200/_cluster/health```
返回结果如下：
	
	{  
   		"cluster_name":"gDelt",
   		"status":"green",
   		"timed_out":false,
   		"number_of_nodes":9,
   		"number_of_data_nodes":9,
   		"active_primary_shards":11,
   		"active_shards":22,
   		"relocating_shards":0,
   		"initializing_shards":0,
   		"unassigned_shards":0,
   		"delayed_unassigned_shards":0,
   		"number_of_pending_tasks":0,
   		"number_of_in_flight_fetch":0,
   		"task_max_waiting_in_queue_millis":0,
   		"active_shards_percent_as_number":100.0
	}
	
大功告成~