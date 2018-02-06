---
layout: post
title: "使用scala透過sparkSQL去搬移Oracle DB的資料到Cassandra上"
---

這篇主要有兩個目的：

1. 幫ROracle澄清其實沒那麼難用，只是要把table name跟column name都轉成大寫，就不會有double quote了

2. 在scala用sparkSQL連ojdbc7，把Oracle資料拉出來，再透過spark-cassandra-connector把資料倒進Cassandra

ROracle安裝部分請參考[這篇](http://chingchuan-chen.github.io/oracle/2016/07/25/use-ROracle-to-manipulate-oracle-database-in-R.html)

在server使用`$ORACLE_HOME/bin/sqlplus system/password@oracleServer:1521/orcl`登入

透過sqlplus在Oracle server上創新的user，其SQL如下：

``` sql
CREATE TABLESPACE testuser
  DATAFILE 'testuser.dat'
  SIZE 40M REUSE AUTOEXTEND ON;

CREATE TEMPORARY TABLESPACE testuser_tmp
  TEMPFILE 'testuser_tmp.dbf'
  SIZE 10M REUSE AUTOEXTEND ON;
  
CREATE USER C##testuser
IDENTIFIED BY testuser
DEFAULT TABLESPACE testuser
TEMPORARY TABLESPACE testuser_tmp
quota unlimited on testuser;

GRANT CREATE SESSION,
      CREATE TABLE,
      CREATE VIEW,
      CREATE PROCEDURE
TO C##testuser; 
```

我們再透過R去塞一個夠大的資料到Oracle上去，其R code如下：

``` R
library(rpace)  # my package (there is a big data.frame)
library(fasttime)
library(ROracle)
library(pipeR)
library(data.table)
Sys.setenv(TZ = "GMT", ORA_SDTZ = "GMT")

tmpData <- TaiwanFreeway5vdData %>>% data.table %>>%
  `[`( , time := fastPOSIXct(sprintf("%i-%02i-%02i %02i:%02i:00", 
                                     year, month, day, hour, minute))) %>>%
  setnames(toupper(names(.)))

print(c(nrow(tmpData), ncol(tmpData)))
# [1] 998177     12

# 連線資訊
host <- "192.168.0.120"
port <- 1521
sid <- "orcl"
connectString <- paste(
  "(DESCRIPTION=",
  "(ADDRESS=(PROTOCOL=TCP)(HOST=", host, ")(PORT=", port, "))",
  "(CONNECT_DATA=(SID=", sid, ")))", sep = "")

# 先用system權限登入查看user
con <- dbConnect(dbDriver("Oracle"), username = "system", password = "qscf12356",
                 dbname = connectString)

# query all_users
userList <- dbSendQuery(con, "select * from ALL_USERS") %>>% 
  fetch(n = -1) %>>% data.table
print(userList)
# 印出表格，user按照創造時間排列
# 可以看到已經創造了C##TESTUSER這個user

# system帳號斷線
dbDisconnect(con)

# 用C##TESTUSER去上傳，這樣才會傳到C##TESTUSER的schema下
con <- dbConnect(dbDriver("Oracle"), username = "c##testuser", 
                 password = "testuser", dbname = connectString)

# 把上傳的table
dbWriteTable(con, "VDDATA", as.data.frame(tmpData), row.names = FALSE)

# 帳號斷線
dbDisconnect(con)
```

在server上query看看結果：

![](/images/roracle_upload_test.PNG)

接下來就是用scala寫一個程式去拉資料

`build.sbt`的部分：

``` bash
name := "oracle2cassandra_sparksql"

version := "1.0"

scalaVersion := "2.11.8"

libraryDependencies += "org.apache.spark" %% "spark-core" % "2.0.0"

libraryDependencies += "org.apache.spark" %% "spark-sql" % "2.0.0"

libraryDependencies += "com.datastax.spark" %% "spark-cassandra-connector" % "2.0.0-M1"
```

scala code:

``` scala
import java.net.InetAddress
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.cassandra._
import com.datastax.spark.connector._
import com.datastax.spark.connector.cql.{CassandraConnector, CassandraConnectorConf}

object ora2cass {
  def main(args: Array[String]) {
    /* test args
      val args = List("C##TESTUSER", "VDDATA", "testuser", "vddata")
    */

    // connection information of Oracle DB
    val oraConnInfo = "jdbc:oracle:thin:system/qscf12356@//192.168.0.120:1521/orcl"
    val oraSchema = args(0).toUpperCase()
    val oraTable = args(1).toUpperCase()
    val cassKeyspace = args(2).toLowerCase()
    val cassTable = args(3).toLowerCase()
    // create SQL to grap data from Oracle DB
    val oraSQL = s"(SELECT ROWIDTOCHAR(t.ROWID) ID,t.ORA_ROWSCN,t.* FROM $oraSchema.$oraTable t) tmp"

    // connection information of Cassandra
    val cassIps = List("192.168.0.121", "192.168.0.122", "192.168.0.123")
      .map(InetAddress.getByName(_)).toSet

    // spark session
    val spark = SparkSession.builder().master("local")
      .appName("oracle to cassandra")
      .config("spark.cassandra.connection.host", "192.168.0.121")
      .config("spark.sql.warehouse.dir", "file:///d://")
      .getOrCreate()

    // read data from Oracle DB
    val jdbcDF = spark.read.format("jdbc")
      .options(Map("url" -> oraConnInfo, "dbtable" -> oraSQL))
      .load()

    // create connection to Cassandra
    val cassConf = new CassandraConnectorConf(cassIps)
    // create keyspace and table
    // get all ks: val allcassKS = session.getCluster().getMetadata().getKeyspaces()
    val cassCreate = new CassandraConnector(cassConf).withSessionDo { session =>
      session.execute(s"CREATE KEYSPACE IF NOT EXISTS $cassKeyspace " +
        "WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 2 }")
      session.execute(s"DROP TABLE IF EXISTS $cassKeyspace.$cassTable")
    }

    // create table in Cassandra
    jdbcDF.createCassandraTable(cassKeyspace, cassTable)
    // save the RDD in cassandra
    jdbcDF.write.cassandraFormat(cassTable, cassKeyspace).save()
  }
}
```

然後在Intellij用Application執行，記得Program arguments要給參數，本地端就可以運行成功了

如果要在遠端伺服器跑的話，`master`跟`spark.sql.warehouse.dir`記得修改成相對應的位置

然後把ojdbc7.jar放進server，用

``` bash
cp ~/ojdbc7.jar $SPARK_HOME/extraClass
cp $SPARK_HOME/conf/spark-defaults.conf.template $SPARK_HOME/conf/spark-defaults.conf

tee -a $SPARK_HOME/conf/spark-defaults.conf << "EOF"
spark.driver.extraClassPath /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar:/usr/local/bigdata/spark/extraClass/ojdbc7.jar
spark.driver.extraLibraryPath /usr/local/bigdata/spark/extraClass
spark.executor.extraClassPath /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar:/usr/local/bigdata/spark/extraClass/ojdbc7.jar
spark.executor.extraLibraryPath /usr/local/bigdata/spark/extraClass
spark.jars /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar,/usr/local/bigdata/spark/extraClass/ojdbc7.jar
EOF
```

重開spark，再用`spark-submit`：

``` bash
spark-submit --class ora2cass oracle2cassandra_sparksql_2.11-1.0.jar C##TESTUSER VDDATA testuser vddata
```

