# ES-Hadoop 的使用

一直以来，ElasticSearch 与 Hadoop 的连接都是一个头疼的问题，直到 *ES-Hadoop* 的出现。

*ES-Hadoop* 支持 MapReduce 计算框架，可以更加快速的从 ES 中读写数据。

本次的任务是将 CSV 文件的各个字段拆分出来，并导入 ES，本来想要先把 CSV 文件转化成 Json 文件再导入，后来发现不用，在 ES 的官方文档中，有这么一句话：

>EsOutputFormat expects a Map[Writable, Writable] representing a document value that is converted internally into a JSON document and indexed in Elasticsearch. Hadoop OutputFormat requires implementations to expect a key and a value however, since for Elasticsearch only the document (that is the value) is necessary, EsOutputFormat ignores the key.

就是说，在 Hfile 中写入 MapWritable 类型的，在写入 ES 的时刻，它会自动转成 Json。


首先，在 ES 中，index 的定义如下：

```json
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  },
  "mappings": {
    "events": {
      "properties": {
        "GlobalEventID": {
          "type": "long"
        },
        "ActorCode": {
          "type": "keyword"
        },
        "ActorName": {
          "type": "keyword"
        },
       "EventCode": {
          "type": "keyword"
        },
        "NumMentions": {
          "type": "integer"
        },
        "NumSources": {
          "type": "integer"
        },
        "NumArticles": {
          "type": "integer"
        },
        "DateAdded": {
          "type": "date"
        },
        "SourceURL": {
          "type": "keyword"
        }
      }
    }
  }
}

```

这次的任务本质上也是一个 MapReduce 任务，主类代码如下：

```scala
import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.fs.Path
import org.apache.hadoop.io.{MapWritable, NullWritable}
import org.apache.hadoop.mapreduce.Job
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat
import org.elasticsearch.hadoop.mr.EsOutputFormat

object H2ES {

    def main(args: Array[String]): Unit = {
        val conf = new Configuration()
        conf.setBoolean("mapreduce.map.speculative", false)
        conf.setBoolean("mapreduce.reduce.speculative", false)
        conf.set("es.resource", "gdelt_test/events")

        val job = Job.getInstance(conf, "H2ES")
        job.setMapperClass(classOf[GdeltMapper])
        job.setMapOutputKeyClass(classOf[NullWritable])
        job.setMapOutputValueClass(classOf[MapWritable])
        job.setOutputFormatClass(classOf[EsOutputFormat])
        job.setJarByClass(H2ES.getClass)

        FileInputFormat.addInputPath(job, new Path(args(1)))
        job.waitForCompletion(true)
    }
}

```

自定义的 Mapper 类如下：

```scala
import org.apache.hadoop.io.{LongWritable, NullWritable, Text, MapWritable, FloatWritable, IntWritable}
import org.apache.hadoop.mapreduce.Mapper
import java.text.SimpleDateFormat
import java.util.Date

class GdeltMapper extends Mapper[LongWritable, Text, NullWritable, MapWritable] {

    override def map(key: LongWritable, value: Text, context: Mapper[LongWritable, Text, NullWritable, MapWritable]#Context): Unit = {
        val data = value.toString.split("\t")
        val mp = new MapWritable()
        val size = data.length - 1
        for (i <- 0 to size) {
            if (GdeltMapper.properties(i) == "DateAdded") {
                val format = new SimpleDateFormat("yyyyMMddHHmmss")
                val millsec: Long = format.parse(data(i)).getTime
                mp.put(new Text(GdeltMapper.properties(i)), new LongWritable(millsec))
            } else if (GdeltMapper.ints.contains(GdeltMapper.properties(i))) {
                if (data(i) == "") {
                    mp.put(new Text(GdeltMapper.properties(i)), new IntWritable(0))
                } else {
                    mp.put(new Text(GdeltMapper.properties(i)), new IntWritable(data(i).toInt))
                }

            } else if (GdeltMapper.floats.contains(GdeltMapper.properties(i))) {
                if (data(i) == "") {
                    mp.put(new Text(GdeltMapper.properties(i)), new FloatWritable(0))
                } else {
                    mp.put(new Text(GdeltMapper.properties(i)), new FloatWritable(data(i).toFloat))
                }

            } else {
                mp.put(new Text(GdeltMapper.properties(i)), new Text(data(i)))
            }

        }

        context.write(NullWritable.get(), mp)
    }
}


object GdeltMapper {
    private val properties = List(
        "GlobalEventID", "SQLDate", "MonthYear", "Year", "FractonDate", "Actor1Code", "Actor1Name", "Actor1CountryCode", "Actor1KnownGroupCode",
        "Actor1EthnicCode", "Actor1Religion1Code", "Actor1Religion2Code", "Actor1Type1Code", "Actor1Type2Code", "Actor1Type3Code",
        "Actor2Code", "Actor2Name", "Actor2CountryCode", "Actor2KnownGroupCode", "Actor2EthnicCode", "Actor2Religion1Code",
        "Actor2Religion2Code", "Actor2Type1Code", "Actor2Type2Code", "Actor2Type3Code", "IsRootEvent", "EventCode", "EventBaseCode",
        "EventRootCode", "QuadClass", "GoldsteinScale", "NumMentions", "NumSources", "NumArticles", "AverageTone", "Actor1GeoType",
        "Actor1GeoFullName", "Actor1GeoCountryCode", "Actor1GeoADM1Code", "Actor1GeoADM2Code", "Actor1GeoLat", "Actor1GeoLong",
        "Actor1GeoFeatureID", "Actor2GeoType", "Actor2GeoFullName", "Actor2GeoCountryCode", "Actor2ADM1Code", "Actor2ADM2Code",
        "Actor2GeoLat", "Actor2GeoLong", "Actor2GeoFeatureID", "ActionGeoType", "ActionGeoFullName", "ActionGeoCountryCode",
        "ActionGeoADM1Code", "ActionGeoADM2Code", "ActionGeoLat", "ActionGeoLong", "ActionGeoFeatureID", "DateAdded", "SourceURL"
    )

    private val ints = Set("GlobalEventID", "SQLDate", "MonthYear", "Year", "IsRootEvent", "QuadClass", "NumMentions",
        "NumSources", "NumArticles", "Actor1GeoType", "Actor2GeoType", "ActionGeoType")

    private val floats = Set("FractonDate", "GoldsteinScale", "AverageTone", "Actor1GeoLat", "Actor1GeoLong", "Actor2GeoLat",
        "Actor2GeoLong", "ActionGeoLat", "ActionGeoLong")
}


```
这样一来，运行

	hadoop jar my.jar H2ES  my/input/file
	
在 Map 和 Reduce 任务完成之后，会有空白的等待，表示正在往 ES 中导入数据，等到本次任务报告打印完成，也会看到 ES-Hadoop 本次传输了多少文档。此时，在 Kibana 中已经可以看到，所有的数据都已经被索引过来。

