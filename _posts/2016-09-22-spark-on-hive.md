---
layout: post
title: "Apache Spark on Apache Hive"
---

這篇主要是用Spark去連接現存的Hive

可能有人會先好奇說為什麼不用Spark本身的Thrift Server

我稍微看了一下，Spark的Thrift Server只能跑Local

也就是說你的資料只能在一台電腦上跑，因此，這樣是有風險的

只是現在Hive的設定只在一台Mysql上，也是很有風險

但是Hive可以把Metastore移到Mysql Cluster上，這樣就可以避開這個風險了

不過這不是本篇的重點，本篇會專注在怎麼用Spark去連接現有的Hive

配置很簡單，只需要使用下面四個指令，以及修改一下Spark的`spark-default.conf`即可

``` bash
cp $HADOOP_CONF_DIR/core-site.xml $SPARK_HOME/conf/
cp $HADOOP_CONF_DIR/hdfs-site.xml $SPARK_HOME/conf/
cp $HIVE_HOME/conf/hive-site.xml $SPARK_HOME/conf/
cp $HIVE_HOME/lib/mysql-connector-java-5.1.39-bin.jar $SPARK_HOME/extraClass/
```

`spark-default.conf`增加下面的東西：

``` bash
spark.driver.extraClassPath /usr/local/bigdata/spark/extraClass/mysql-connector-java-5.1.39-bin.jar
spark.executor.extraClassPath /usr/local/bigdata/spark/extraClass/mysql-connector-java-5.1.39-bin.jar
```

如果已經設定了，就用`,`去append即可


接下來就可以直接執行spark-shell了

執行`spark-shell mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos`

script如下：

``` scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder().appName("spark on hive").
  config("spark.sql.warehouse.dir", "hdfs://hc1/spark").
  enableHiveSupport().getOrCreate()

// 之前丟上去hive的test_df，沒有資料的話可以往前翻兩篇...
spark.sql("select count(*) from vddata").show()
/*
+--------+
|count(1)|
+--------+
|      41|
+--------+
*/

spark.sql("select v1,v2,sum(v3) from test_df group by v1,v2").show()
/*
+---+---+-------------------+
| v1| v2|            sum(v3)|
+---+---+-------------------+
|  b|  e| 1.1515786935289156|
|  a|  c| 2.1048099797195103|
|  b|  d| 0.5861563687201317|
|  a|  d|-0.4549045928741139|
|  b|  c| 2.0411431978258983|
|  a|  e| -4.942821584114654|
| v1| v2|               null|
+---+---+-------------------+
*/
```
