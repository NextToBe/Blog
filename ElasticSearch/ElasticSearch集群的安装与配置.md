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
