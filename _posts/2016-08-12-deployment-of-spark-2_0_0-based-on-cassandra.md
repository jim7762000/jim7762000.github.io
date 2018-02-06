---
layout: post
title: "基於cassandra的spark 2.0.0環境部署 (scala 2.11)"
---

spark升級到2.0.0，等了幾天

spark-cassandra-connector終於升級到2.0.0-M1

就直接來嘗試重新裝一次

1. 準備工作

這裡基本上跟[前篇](http://chingchuan-chen.github.io/cassandra/2016/08/05/deployment-of-spark-based-on-cassandra.html)一樣，就不贅述了

2. 開始部署

i. 下載檔案並移到適當位置
    
``` bash
sudo mkdir /usr/local/bigdata
sudo chown -R tester /usr/local/bigdata

# 下載並安裝java
curl -v -j -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.rpm -o jdk-8u101-linux-x64.rpm
sudo yum install -y jdk-8u101-linux-x64.rpm
# 下載並部署scala
curl -v -j -k -L http://downloads.lightbend.com/scala/2.11.8/scala-2.11.8.tgz -o scala-2.11.8.tgz
tar -zxvf scala-2.11.8.tgz
sudo mkdir /usr/local/scala
sudo mv scala-2.11.8 /usr/local/scala/scala-2.11
# 下載並部署spark
curl -v -j -k -L http://d3kbcqa49mib13.cloudfront.net/spark-2.0.0-bin-hadoop2.6.tgz -o spark-2.0.0-bin-hadoop2.6.tgz
tar -zxvf spark-2.0.0-bin-hadoop2.6.tgz
mv spark-2.0.0-bin-hadoop2.6 /usr/local/bigdata/spark
# 下載並部署cassandra
curl -v -j -k -L http://apache.stu.edu.tw/cassandra/2.2.7/apache-cassandra-2.2.7-bin.tar.gz -o apache-cassandra-2.2.7-bin.tar.gz
tar -zxvf apache-cassandra-2.2.7-bin.tar.gz
mv apache-cassandra-2.2.7 /usr/local/bigdata/cassandra

# 如果不是VM，實體可以一台下載之後用scp
# scp -r /usr/local/bigdata/* tester@cassSpark2:/usr/local/bigdata
# scp -r /usr/local/bigdata/* tester@cassSpark3:/usr/local/bigdata
```
   
ii. 環境變數設置

``` bash
sudo tee -a /etc/bashrc << "EOF"
# JAVA
export JAVA_HOME=/usr/java/jdk1.8.0_101
# SCALA
export SCALA_HOME=/usr/local/scala/scala-2.11
# SPARK
export SPARK_HOME=/usr/local/bigdata/spark
# CASSANDRA
export CASSANDRA_HOME=/usr/local/bigdata/cassandra
# PATH
export PATH=$PATH:$JAVA_HOME:$SPARK_HOME/bin:$SPARK_HOME/sbin:$CASSANDRA_HOME/bin
EOF
source /etc/bashrc
```

iii. 配置scala and spark
      
``` bash
tee $SPARK_HOME/conf/slaves << "EOF"
cassSpark1
cassSpark2
cassSpark3
EOF

# 複製檔案
cp $SPARK_HOME/conf/spark-env.sh.template $SPARK_HOME/conf/spark-env.sh
cp $SPARK_HOME/conf/log4j.properties.template $SPARK_HOME/conf/log4j.properties
cp $SPARK_HOME/conf/spark-defaults.conf.template $SPARK_HOME/conf/spark-defaults.conf

# 傳入設定
tee -a $SPARK_HOME/conf/spark-env.sh << "EOF"
SPARK_MASTER_IP=cassSpark1
SPARK_LOCAL_DIRS=/usr/local/bigdata/spark
SPARK_SCALA_VERSION=2.11
SPARK_DRIVER_MEMORY=3G
SPARK_WORKER_CORES=2
SPARK_WORKER_MEMORY=3g
EOF

# install sbt and git 
sudo yum install sbt git-core

# clone spark-cassandra-connector
git clone git@github.com:datastax/spark-cassandra-connector.git

# compile assembly jar
cd spark-cassandra-connector
rm -r spark-cassandra-connector-perf
sbt -Dscala-2.11=true assembly

# copy jar to spark
mkdir $SPARK_HOME/extraClass
cp spark-cassandra-connector/target/scala-2.11/spark-cassandra-connector-assembly-2.0.0-M1-2-g70018a6.jar $SPARK_HOME/extraClass

tee -a $SPARK_HOME/conf/spark-defaults.conf << "EOF"
spark.driver.extraClassPath /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar
spark.driver.extraLibraryPath /usr/local/bigdata/spark/extraClass
spark.executor.extraClassPath /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar
spark.executor.extraLibraryPath /usr/local/bigdata/spark/extraClass
spark.jars /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar
EOF

# 複製到各台
scp -r /usr/local/bigdata/spark tester@cassSpark2:/usr/local/bigdata
scp -r /usr/local/bigdata/spark tester@cassSpark3:/usr/local/bigdata
```

slaves的部署、cassandra的設置跟自動啟動部分就都一樣，此處也跳過，直接進測試

3. 測試

用`cqlsh cassSpark1`登入CQL介面

用下面指令去創測試資料：

``` SQL
CREATE KEYSPACE test WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3 };
CREATE TABLE test.kv(key text PRIMARY KEY, value int);
INSERT INTO test.kv(key, value) VALUES ('key1', 1);
INSERT INTO test.kv(key, value) VALUES ('key2', 2);
INSERT INTO test.kv(key, value) VALUES ('key3', 11);
INSERT INTO test.kv(key, value) VALUES ('key4', 25);
```

然後用`spark-shell`打開spark的shell

``` scala
sc.stop()
import com.datastax.spark.connector._, org.apache.spark.SparkContext, org.apache.spark.SparkContext._, org.apache.spark.SparkConf
val conf = new SparkConf(true).set("spark.cassandra.connection.host", "192.168.0.121")
val sc = new SparkContext(conf)
val rdd = sc.cassandraTable("test", "kv")
println(rdd.first)
// CassandraRow{key: key4, value: 25}
println(rdd.map(_.getInt("value")).sum) // 39.0
val collection = sc.parallelize(Seq(("key5", 38), ("key6", 5)))
// save new data into cassandra
collection.saveToCassandra("test", "kv", SomeColumns("key", "value"))
sc.stop()
:quit
```

用`cqlsh cassSpark1`登入CQL，去看剛剛塞進去的資料

``` SQL
USE test;
select * from kv;
#  key   | value
# -------+-------
#   key5 |    38
#   key6 |     5
#   key1 |     1
#   key4 |    25
#   key3 |    11
#   key2 |     2
```

根據我自己的測試，要用spark-submit才能在remote cluster上run

Intellij的application不能直接跑，所以先用intellij的SBT Task: package (scala SDK用2.11.8)

會在專案目錄/target/scala-2.11下面產生一個jar檔，我這裡是`test_cassspark_2.11-1.0.jar`

把這個jar上傳到cluster上，然後用下面指令就可以成功運行：

``` bash
spark-submit --class cassSpark test_cassspark_2.11-1.0.jar
```

4. Ref
    1. https://github.com/datastax/spark-cassandra-connector/blob/master/doc/0_quick_start.md
    2. https://tobert.github.io/post/2014-07-15-installing-cassandra-spark-stack.html


