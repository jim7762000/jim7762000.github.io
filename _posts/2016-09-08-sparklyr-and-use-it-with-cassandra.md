---
layout: post
title: "sparklyr初探，並連接Cassandra使用"
---

RStudio推出了一個感覺很厲害的套件`sparklyr`

可以讓dplyr變得lazy，然後去即時的操作Spark中的dataFrame

先從安裝R, RStudio Server開始：

``` bash
curl -v -j -k -L https://mran.microsoft.com/install/mro/3.3.1/microsoft-r-open-3.3.1.tar.gz -o microsoft-r-open-3.3.1.tar.gz
tar -zxvf microsoft-r-open-3.3.1.tar.gz
cd microsoft-r-open/rpm
su -c 'rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm'
# for offline installation
sudo yum install R R-devel R-java --downloadonly --downloaddir=r_deps
sudo yum install openssl-devel libcurl-devel --downloadonly --downloaddir=r_deps
sudo yum install *.rpm --downloadonly --downloaddir=r_deps
# install R, R-devel and R-java and deps
sudo yum install -y r_deps/*.rpm
# remove R
sudo rm -rf /usr/lib64/R
sudo rm -rf /usr/bin/R
sudo rm -rf /usr/bin/Rscript
# install MRO and MKL
sudo yum install -y *.rpm

sudo chown -R tester:tester /usr/lib64/microsoft-r/3.3/lib64/R
sudo chmod -R 775 /usr/lib64/microsoft-r/3.3/lib64/R

# for offline installation
sudo chown -R tester:tester r_deps
mv r_deps ~/

# 這裡下載preview版本，是為了可以直接有UI操作spark
curl -v -j -k -L https://s3.amazonaws.com/rstudio-dailybuilds/rstudio-server-rhel-1.0.8-x86_64.rpm -o rstudio-server-rhel-1.0.8-x86_64.rpm
sudo yum install --nogpgcheck rstudio-server-rhel-1.0.8-x86_64.rpm
## start the rstudio-server
sudo rstudio-server start
## start rstudio-server on boot
sudo cp /usr/lib/rstudio-server/extras/init.d/redhat/rstudio-server /etc/init.d/
sudo chmod 755 /etc/init.d/rstudio-server
sudo chkconfig --add rstudio-server
```

為了測試，我先用scala在spark-shell上塞了一些資料進去

``` scala
spark.stop()

import java.net.InetAddress._
import com.datastax.spark.connector._
import org.apache.spark.sql.{SparkSession, SaveMode}
import com.datastax.spark.connector.cql.{CassandraConnector, CassandraConnectorConf}
import org.apache.spark.sql.functions._

val cassIps = Set("192.168.0.121", "192.168.0.122", "192.168.0.123")

val spark = SparkSession.builder().appName("test")
  .config("spark.cassandra.connection.host", cassIps.mkString(","))
  .config("spark.sql.warehouse.dir", "file:///home/tester").getOrCreate()
  
import spark.implicits._

val cassConf = new CassandraConnectorConf(cassIps.map(getByName(_)))

val exeRes = new CassandraConnector(cassConf).withSessionDo { session =>
  session.execute("CREATE KEYSPACE IF NOT EXISTS test " +
    "WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 3 }")
  session.execute("DROP TABLE IF EXISTS test.kv2")
  session.execute("CREATE TABLE test.kv2 (var0 text, var1 text, var2 double, PRIMARY KEY(var0, var1))")
  session.execute("INSERT INTO test.kv2(var0, var1, var2) VALUES ('T', 'A', 23.1)")
  session.execute("INSERT INTO test.kv2(var0, var1, var2) VALUES ('T', 'B', 17.5)")
  session.execute("INSERT INTO test.kv2(var0, var1, var2) VALUES ('U', 'B', 11.9)")
  session.execute("INSERT INTO test.kv2(var0, var1, var2) VALUES ('U', 'A', 25.3)")
}

val collection = spark.sparkContext.parallelize(Seq(("T", "C", 17.0), ("U", "C", 5.0)))
collection.saveToCassandra("test", "kv2", SomeColumns("var0", "var1", "var2"))

val cass_tbl = spark.read.format("org.apache.spark.sql.cassandra")
  .option("keyspace", "test").option("table", "kv2").load()

val concatMap = udf((maps: Seq[Map[String, Double]]) => maps.flatten.toMap)
val cass_tbl_agg = cass_tbl.withColumn("var2_map", map($"var1", $"var2")).groupBy($"var0").agg(concatMap(collect_list($"var2_map")).alias("var2"))

try {
  cass_tbl_agg.createCassandraTable("test", "kv2_trans")
} catch {
  case e1: com.datastax.driver.core.exceptions.AlreadyExistsException => None
  case e2: Exception => throw e2
}

cass_tbl_agg.write.format("org.apache.spark.sql.cassandra").options(Map("table" -> "kv2_trans", "keyspace" -> "test")).mode(SaveMode.Append).save()
```

裡面的`test.kv2`會長下面這樣：

| var0 | var1 | var2 |
|------|------|------|
|    T |    A | 23.1 |
|    T |    B | 17.5 |
|    T |    C |   17 |
|    U |    A | 25.3 |
|    U |    B | 11.9 |
|    U |    C |    5 |

裡面的`test.kv2_trans`會長下面這樣：

| var0 | var2                            |
|------+---------------------------------|
|    T | {'A': 23.1, 'B': 17.5, 'C': 17} |
|    U |  {'A': 25.3, 'B': 11.9, 'C': 5} |

可以看到`test.kv2`是一般長相的table

`test.kv2_trans`則是包含了Map這個資料格式


接下來安裝`sparklyr`

``` R
install.packages("devtools")
devtools::install_github("rstudio/sparklyr")
```

如果照我之前文章跟我設定一樣Spark的路徑的話

下面的R script就可以直接執行看到結果了：

``` R
# 設定spark home的位置
Sys.setenv(SPARK_HOME = "/usr/local/bigdata/spark")
# library sparklyr跟dplyr
library(sparklyr)
library(dplyr)

# library pipeR跟stringr
library(pipeR)
library(stringr)

# 讀取spark的設定
spark_settings <- readLines(paste0(Sys.getenv("SPARK_HOME"), "/conf/spark-defaults.conf")) %>>%
  `[`(!str_detect(., "^#")) %>>% `[`(nchar(.) > 0) %>>% str_split("\\s")
# 複製sparklyr的預設config檔案
file.copy(system.file(file.path("conf", "config-template.yml"), 
          package = "sparklyr"), "config.yml", TRUE)
# 把spark的設定寫進config去
sink("config.yml", TRUE)
sapply(spark_settings %>>% sapply(paste, collapse = ": "), 
       function(x) cat(" ", x, "\n")) %>>% invisible
sink()

# 開啟spark connection
spark_master <- "mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos"
sc <- spark_connect(master = spark_master, config = spark_config("config.yml", FALSE))

# 開啟spark session
sparkSession <- invoke_static(sc2, "org.apache.spark.sql.SparkSession", "builder") %>>%
  invoke("config", "spark.cassandra.connection.host", 
         "192.168.0.121,192.168.0.122,192.168.0.123") %>>%
  invoke("getOrCreate")

# 讀取Cassandra table，可以看到讀進來是Dataset，還是一個java的物件
cass_df <- invoke(sparkSession, "read") %>>% 
  invoke("format", "org.apache.spark.sql.cassandra") %>>%
  invoke("option", "keyspace", "test") %>>% invoke("option", "table", "kv2") %>>%
  invoke("load") %>>% (~ print(cass_df))
# <jobj[32]>
#   class org.apache.spark.sql.Dataset
#   [var0: string, var1: string ... 1 more field]
 
# register Spark dataframe，註冊後，就可以在R裡面看到部分資料，並且可以使用dplyr
cass_df_tbl <- sparklyr:::spark_partition_register_df(sc2, cass_df, "test_kv2", 0, TRUE)
print(cass_df_tbl)
# Source:   query [?? x 3]
# Database: spark connection master=mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos app=sparklyr local=FALSE
# 
#    var0  var1  var2
#   <chr> <chr> <dbl>
# 1     T     A  23.1
# 2     T     B  17.5
# 3     T     C  17.0
# 4     U     A  25.3
# 5     U     B  11.9
# 6     U     C   5.0

# 測試一下filter，可以發現結果還 ?? x 3，因為dplyr這裡還是lazy，還沒做回收的動作
cass_df_tbl %>>% filter(var2 > 5)
# Source:   query [?? x 3]
# Database: spark connection master=mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos app=sparklyr local=FALSE
# 
#    var0  var1  var2
#   <chr> <chr> <dbl>
# 1     T     A  23.1
# 2     T     B  17.5
# 3     T     C  17.0
# 4     U     A  25.3
# 5     U     B  11.9

# 使用as.data.frame就會變成local了
cass_df_tbl %>>% filter(var2 > 5) %>>% as.data.frame
#   var0 var1 var2
# 1    T    A 23.1
# 2    T    B 17.5
# 3    T    C 17.0
# 4    U    A 25.3
# 5    U    B 11.9

# 這裡也可以使用很多org.apache.spark.sql.functions中的函數
# 最簡單的如abs, min, max, sin, cos, sqrt等
# 到比較少見的如map, collect_list, unix_timestamp等
# 下面示範把column在spark裡面bind成map
cass_df_tbl_agg <- cass_df_tbl %>>% mutate(var2_map = map(var1, var2)) %>>% 
  group_by(var0) %>>% summarise(var2 = collect_list(var2_map)) %>>% as.data.frame
  
# print data.frame，看不到map的key
print(cass_df_tbl_agg)
#   var0             var2
# 1    T 23.1, 17.5, 17.0
# 2    U  25.3, 11.9, 5.0

# 用str看，可以看到key還是有被留下來
# 只是要在R裡面再做一點處理而已，這部分就越過
str(cass_df_tbl_agg)
'data.frame':	2 obs. of  2 variables:
 $ var0: chr  "T" "U"
 $ var2:List of 2
  ..$ :List of 3
  .. ..$ :List of 1
  .. .. ..$ A: num 23.1
  .. ..$ :List of 1
  .. .. ..$ B: num 17.5
  .. ..$ :List of 1
  .. .. ..$ C: num 17
  ..$ :List of 3
  .. ..$ :List of 1
  .. .. ..$ A: num 25.3
  .. ..$ :List of 1
  .. .. ..$ B: num 11.9
  .. ..$ :List of 1
  .. .. ..$ C: num 5
  
# 再來是直接從Cassandra讀入那張含有map欄位的表，一樣是Dataset
cass_df_2 <- invoke(sparkSession, "read") %>>% 
  invoke("format", "org.apache.spark.sql.cassandra") %>>%
  invoke("option", "keyspace", "test") %>>% invoke("option", "table", "kv2_trans") %>>%
  invoke("load") %>>% (~print(.))
# <jobj[51]>
#   class org.apache.spark.sql.Dataset
#   [var0: string, var2: map<string,double>]

# 一樣做register的動作
cass_df_tbl_2 <- sparklyr:::spark_partition_register_df(sc2, cass_df_2, "test_kv2_trans", 0, TRUE)

# print出來看看，是變成list了，只是不知道有沒有抓到key
print(cass_df_tbl_2)
# Source:   query [?? x 2]
# Database: spark connection master=mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos app=sparklyr local=FALSE
# 
#    var0       var2
#   <chr>     <list>
# 1     T <list [3]>
# 2     U <list [3]>

# 拉回local瞧瞧，可以看到變成很漂亮的format
as.data.frame(cass_df_tbl_2) %>>% str
# 'data.frame':	2 obs. of  2 variables:
#  $ var0: chr  "T" "U"
#  $ var2:List of 2
#   ..$ :List of 3
#   .. ..$ A: num 23.1
#   .. ..$ B: num 17.5
#   .. ..$ C: num 17
#   ..$ :List of 3
#   .. ..$ A: num 25.3
#   .. ..$ B: num 11.9
#   .. ..$ C: num 5

# 這裡拉回local，用data.table稍微整一下就可以回到test.kv2的樣子了
cass_df_tbl_2_r <- as.data.frame(cass_df_tbl_2) %>>% data.table %>>%
  `[`( , `:=`(var2_v = lapply(var2, unlist), var2_k = lapply(var2, names))) %>>%
  `[`( , .(var2_value = unlist(var2_v), var2_key = unlist(var2_k)), by = "var0") %>>%
  (~ print(.))
#    var0 var2_value var2_key
# 1:    T       23.1        A
# 2:    T       17.5        B
# 3:    T       17.0        C
# 4:    U       25.3        A
# 5:    U       11.9        B
# 6:    U        5.0        C
```

以上是sparklyr的初探，並嘗試Cassandra的結果


一些小心得：

沒有辦法直接udf，無法直接套用R函數是致命傷

幾乎都要去翻spark.sql.functions的document

能用的操作少之又少，又懶得去寫extension

希望這部分能趕快有enhancement，滿期待這個出現[Github issue](https://github.com/rstudio/sparklyr/issues/191)

