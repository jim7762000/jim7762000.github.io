---
layout: post
title: "Spark assembly"
---

之前一直配置失敗的Spark assembly

今天花了點時間GOOGLE，終於可以成功assembly了

也可以擺脫在Spark設定時候那些Jars檔了

在Intellij開啟專案後，在`build.sbt`新增如下：

``` scala
// Spark相關的記得都加上 % "provided" 因為cluster上已經會有對應的jar檔了
libraryDependencies ++= Seq(
  "com.datastax.spark" %% "spark-cassandra-connector" % "2.0.0-M1",
  "org.apache.spark" %% "spark-core" % "2.0.0" % "provided",
  "org.apache.spark" %% "spark-sql" % "2.0.0" % "provided",
  "org.apache.spark" %% "spark-mllib" % "2.0.0" % "provided",
  "com.databricks" %% "spark-csv" % "1.5.0",
  "com.github.nscala-time" % "nscala-time_2.11" % "2.14.0"
)

// 避免重複納入，只merge第一個出現的檔案
assemblyMergeStrategy in assembly := {
  case PathList(ps @ _*) if ps.last endsWith ".properties" => MergeStrategy.first
  case x =>
    val oldStrategy = (assemblyMergeStrategy in assembly).value
    oldStrategy(x)
}

// 避免出現com.google.guava版本低於1.6.1的錯誤
assemblyShadeRules in assembly := Seq(
  ShadeRule.rename("com.google.**" -> "shadeio.@1").inAll
)
```

並在project下方新增一個檔案`assembly.sbt`，其內容是：

``` scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.3")
```

接著在src/main/scala-2.11裡面新增檔案`cassSpark.scala`

程式內容如下：

``` scala
import java.net.InetAddress._
import com.datastax.spark.connector._
import org.apache.spark.sql.{SparkSession, SaveMode}
import com.datastax.spark.connector.cql.{CassandraConnector, CassandraConnectorConf}
import org.apache.spark.sql.functions._

object cassSpark {
  def main(args: Array[String]) {
    val cassIps = Set("192.168.0.121", "192.168.0.122", "192.168.0.123")

    val spark = SparkSession.builder().appName("test")
      .config("spark.cassandra.connection.host", cassIps.mkString(","))
      .config("spark.sql.warehouse.dir", "file:///home/tester").getOrCreate()

    import spark.implicits._

    val cassConf = new CassandraConnectorConf(cassIps.map(getByName(_)))

    val exeRes = new CassandraConnector(cassConf).withSessionDo { session =>
      session.execute("CREATE KEYSPACE IF NOT EXISTS test " +
        "WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 3 }")
      session.execute("DROP TABLE IF EXISTS test.testtbl")
      session.execute("CREATE TABLE test.testtbl (var0 text, var1 text, var2 double, PRIMARY KEY(var0, var1))")
      session.execute("INSERT INTO test.testtbl(var0, var1, var2) VALUES ('T', 'A', 23.1)")
      session.execute("INSERT INTO test.testtbl(var0, var1, var2) VALUES ('T', 'B', 17.5)")
      session.execute("INSERT INTO test.testtbl(var0, var1, var2) VALUES ('U', 'B', 11.9)")
      session.execute("INSERT INTO test.testtbl(var0, var1, var2) VALUES ('U', 'A', 25.3)")
    }

    val collection = spark.sparkContext.parallelize(Seq(("T", "C", 17.0), ("U", "C", 5.0)))
    collection.saveToCassandra("test", "testtbl", SomeColumns("var0", "var1", "var2"))

    val cass_tbl = spark.read.format("org.apache.spark.sql.cassandra")
      .option("keyspace", "test").option("table", "testtbl").load()

    cass_tbl.write.format("com.databricks.spark.csv").option("header", "true").save("file:///home/tester/test.csv")

    val concatMap = udf((maps: Seq[Map[String, Double]]) => maps.flatten.toMap)
    val cass_tbl_agg = cass_tbl.withColumn("var2_map", map($"var1", $"var2")).groupBy($"var0").agg(concatMap(collect_list($"var2_map")).alias("var2"))

    try {
      cass_tbl_agg.createCassandraTable("test", "testtbl_trans")
    } catch {
      case e1: com.datastax.driver.core.exceptions.AlreadyExistsException => None
      case e2: Exception => throw e2
    }

    cass_tbl_agg.write.format("org.apache.spark.sql.cassandra").options(Map("table" -> "testtbl_trans", "keyspace" -> "test")).mode(SaveMode.Append).save()
  }
}

```

最後，新增一個SBT Task，打assembly，然後執行就可以在`target/scala-2.11`下看到assembly的檔案了


再來是`$SPARK_HOME/conf/spark-default.conf`的設定

原本設定了`spark.jars`、`spark.executor.extraClassPath`、`spark.executor.extraLibraryPath`

`spark.driver.extraClassPath`以及`spark.driver.extraLibraryPath`就可以通通先拿掉

然後上傳`test_cassSpark-assembly-1.0.jar`，在console輸入下面的指令就可以執行了

``` bash
spark-submit --master mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos --class cassSpark test_cassSpark-assembly-1.0.jar
```

最後就可以在`/home/tester/test.csv`看到產生的csv檔案了
