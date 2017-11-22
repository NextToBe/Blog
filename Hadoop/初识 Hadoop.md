#初识 Hadoop
Hadoop 是我一直想要学习的技术，恰巧毕设要用到它，也让我打开了前往新世界的大门。

## Hadoop 的核心
 Hadoop的核心就是HDFS和MapReduce，而两者只是理论基础，不是具体可使用的高级应用，Hadoop旗下有很多经典子项目，比如HBase、Hive等，这些都是基于HDFS和MapReduce发展出来的。要想了解Hadoop，就必须知道HDFS和MapReduce是什么。
 
### HDFS
HDFS（Hadoop Distributed File System，Hadoop分布式文件系统），它是一个高度容错性的系统，适合部署在廉价的机器上。HDFS能提供高吞吐量的数据访问，适合那些有着超大数据集（large data set）的应用程序。

HDFS 设计的特点是：

* 大数据文件，非常适合上T级别的大文件或者一堆大数据文件的存储，如果文件只有几个G甚至更小就没啥意思了。
* 文件分块存储，HDFS会将一个完整的大文件平均分块存储到不同计算器上，它的意义在于读取文件时可以同时从多个主机取不同区块的文件，多主机读取比单主机读取效率要高得多得都。
* 流式数据访问，一次写入多次读写，这种模式跟传统文件不同，它不支持动态改变文件内容，而是要求让文件一次写入就不做变化，要变化也只能在文件末添加内容。
* 廉价硬件，HDFS可以应用在普通PC机上，这种机制能够让给一些公司用几十台廉价的计算机就可以撑起一个大数据集群。
* 硬件故障，HDFS认为所有计算机都可能会出问题，为了防止某个主机失效读取不到该主机的块文件，它将同一个文件块副本分配到其它某几个主机上，如果其中一台主机失效，可以迅速找另一块副本取文件

HDFS 的关键点有：

* Block：将一个文件进行分块，通常是64M。
* NameNode：保存整个文件系统的目录信息、文件信息及分块信息，这是由唯一一台主机专门保存，当然这台主机如果出错，NameNode 就失效了。在 Hadoop2.* 开始支持 activity-standy 模式----如果主NameNode失效，启动备用主机运行 NameNode。 
* DataNode：分布在廉价的计算机上，用于存储Block块文件


### MapReduce
Hadoop MapReduce是一个使用简易的软件框架，基于它写出来的应用程序能够运行在由上千个商用机器组成的大型集群上，并以一种可靠容错的方式并行处理上T级别的数据集。

一个Map/Reduce 作业（job） 通常会把输入的数据集切分为若干独立的数据块，由 map任务（task）以完全并行的方式处理它们。框架会对map的输出先进行排序， 然后把结果输入给reduce任务。通常作业的输入和输出都会被存储在文件系统中。 整个框架负责任务的调度和监控，以及重新执行已经失败的任务。

通常，Map/Reduce框架和分布式文件系统是运行在一组相同的节点上的，也就是说，计算节点和存储节点通常在一起。这种配置允许框架在那些已经存好数据的节点上高效地调度任务，这可以使整个集群的网络带宽被非常高效地利用。

Map/Reduce框架由一个单独的master JobTracker 和每个集群节点一个slave TaskTracker共同组成。master负责调度构成一个作业的所有任务，这些任务分布在不同的slave上，master监控它们的执行，重新执行已经失败的任务。而slave仅负责执行由master指派的任务。

应用程序至少应该指明输入/输出的位置（路径），并通过实现合适的接口或抽象类提供map和reduce函数。再加上其他作业的参数，就构成了作业配置（job configuration）。然后，Hadoop的 job client提交作业（jar包/可执行程序等）和配置信息给JobTracker，后者负责分发这些软件和配置信息给slave、调度任务并监控它们的执行，同时提供状态和诊断信息给job-client。

在本次毕设中，需要同时使用 Hadoop 和 ElasticSearch 两种技术，第一步的工作当然是转换数据格式，ES 能够识别并索引的只有 Json 文件，而本次用到的 Hfile 都是 CSV 文件，所以，需要转换为 Json 才能导入 ES。

下面是代码示例(Scala):

```scala

object H2ES {
    
    def main(args: Array[String]): Unit = {
        val conf = new Configuration()
        conf.setBoolean("mapreduce.map.speculative", false)
        conf.setBoolean("mapreduce.reduce.speculative", false)
       
        val job = Job.getInstance(conf, "H2ES")
        job.setMapperClass(classOf[GdeltMapper]) //设置自定义 Mapper
        job.setMapOutputKeyClass(classOf[NullWritable])
        job.setMapOutputValueClass(classOf[Text])
        job.setJarByClass(H2ES.getClass) // 没有的话找不到自定义类

        FileInputFormat.addInputPath(job, new Path(args(1)))
        val outputPath = new Path(args(2))
        val hdfs = FileSystem.get(conf) 
        if (hdfs.exists(outputDir) {
        	hdfs.delete(outputDir, true);
        }
        
        job.waitForCompletion(true)
    }
}

class GdeltMapper extends Mapper[LongWritable, Text, NullWritable, Text] {

    override def map(key: LongWritable, value: Text, context: Mapper[LongWritable, Text, NullWritable, Text]#Context): Unit = {
    	// 此时，key 是文件的 index(行号)，value 是每一行的值
        val data = value.toString.split("\t")
        val gson = new Gson() // 使用 Gson 包序列化
        val mp = new Java.util.HashMap<String, String>()
        val size = data.length - 1
        for (i <- 0 to size) {
            mp.put(new Text(GdeltMapper.properties(i)), new Text(data(i)))
        }
		
        context.write(NullWritable.get(), gson.toJson(mp))
    }
}

```

使用 sbt 命令打包，在集群上执行以下命令：

	hadoop jar H2ES  my.jar /your/input/path  /your/output/path
	
耐心等待程序运行完成，就会在 /your/output/path 看到 Reduce 完成的 Hfile。

