---
layout: post
title: "Apache Spark and Apache Hive"
---

關於Spark操作Hive資料庫的一些心得

1. 動態分割表

建議在Hive setup的時候修改這個參數：

``` xml
<property>
  <name>hive.exec.dynamic.partition.mode</name>
  <value>nonstrict</value>
  <description>
    In strict mode, the user must specify at least one static partition
    in case the user accidentally overwrites all partitions.
    In nonstrict mode all partitions are allowed to be dynamic.
  </description>
</property>
```

2. 從Spark寫入Hive

`spark-shell mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos`開啟spark-shell

``` scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder().appName("spark on hive").
  config("spark.sql.warehouse.dir", "hdfs://hc1/spark").
  enableHiveSupport().getOrCreate()
  
spark.sql("CREATE TABLE test_df (v1 STRING, v2 STRING, v3 DOUBLE)" + 
  "ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE")
spark.sql("LOAD DATA LOCAL INPATH '/home/tester/test_df.csv' OVERWRITE INTO TABLE test_df")

val hiveDF = spark.sql("select * from test_df")
hiveDF.show(2)
# +-------------------+---+----+
# |                 v1| v2|  v3|
# +-------------------+---+----+
# |-0.0170655635959667|  e|null|
# |  0.441907768470412|  d|null|
# +-------------------+---+----+
# only showing top 2 rows

# 試著換欄位位置
val hiveDF2 = hiveDF.select($"v3", $"v2", $"v1")
hiveDF2.show(2)
# +----+---+-------------------+
# |  v3| v2|                 v1|
# +----+---+-------------------+
# |null|  e|-0.0170655635959667|
# |null|  d|  0.441907768470412|
# +----+---+-------------------+
# only showing top 2 rows

# 寫進test_df
hiveDF2.write.insertInto("test_df")

spark.sql("select * from test_df").show(2)
#+-------------------+---+----+
#|                 v1| v2|  v3|
#+-------------------+---+----+
#|-0.0170655635959667|  e|null|
#|  0.441907768470412|  d|null|
#+-------------------+---+----+
#only showing top 2 rows
```

所以在寫入的時候要注意，欄位順序避免發生這種問題

利用`select`這個函數去調換column的位置以避免寫入問題

``` SQL
spark.sql("drop table test_df")
spark.sql("CREATE TABLE test_df (v1 STRING, v2 STRING, v3 DOUBLE)" + 
  "ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE")
spark.sql("LOAD DATA LOCAL INPATH '/home/tester/test_df.csv' OVERWRITE INTO TABLE test_df")

val hiveDF = spark.sql("select * from test_df")
hiveDF.show(2)
# +---+---+-------------------+
# | v1| v2|                 v3|
# +---+---+-------------------+
# |  b|  e|-0.0170655635959667|
# |  a|  d|  0.441907768470412|
# +---+---+-------------------+
# only showing top 2 rows

val cols_order = hiveDF.columns
val hiveDF2 = hiveDF.select($"v3", $"v2", $"v1")
hiveDF2.select(cols_order.head, cols_order.tail: _*).write.insertInto("test_df")
spark.sql("select * from test_df").show(2)
# +---+---+-------------------+
# | v1| v2|                 v3|
# +---+---+-------------------+
# |  b|  e|-0.0170655635959667|
# |  a|  d|  0.441907768470412|
# +---+---+-------------------+
# only showing top 2 rows
```
