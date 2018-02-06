---
layout: post
title: "部署Spark on Mesos and Cassandra環境"
---

本篇主要在部署Spark on Mesos的環境

目標是Spark跟Mesos的master配上兩台Mesos standby(同時為Zookeeper)

cassSpark1為Mesos master跟Spark master，cassSpark2以及cassSpark3為mesos standby

這三台同時也是Mesos slaves跟Spark slaves (實際用途中，會是其他電腦，這裡用VM就都放一起了)

1. 準備工作

這裡基本上跟[前篇](http://chingchuan-chen.github.io/cassandra/2016/08/05/deployment-of-spark-based-on-cassandra.html)一樣，就不贅述了

2. 開始部署

i. 下載檔案並移到適當位置

``` bash
# 建立放置資料夾
sudo mkdir /usr/local/bigdata
sudo chown -R tester /usr/local/bigdata

# 下載並安裝java
curl -v -j -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.rpm -o jdk-8u101-linux-x64.rpm
sudo yum install -y jdk-8u101-linux-x64.rpm
# 下載並部署zookeeper
curl -v -j -k -L http://apache.stu.edu.tw/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz -o zookeeper-3.4.8.tar.gz
tar -zxvf zookeeper-3.4.8.tar.gz
sudo mv zookeeper-3.4.8 /usr/local/bigdata/zookeeper
sudo chown -R tester /usr/local/bigdata/zookeeper
# 下載並部署mesos
curl -v -j -k -L http://repos.mesosphere.com/el/7/x86_64/RPMS/mesos-1.0.0-2.0.89.centos701406.x86_64.rpm -o mesos-1.0.0-2.0.89.centos701406.x86_64.rpm
sudo yum install mesos-1.0.0-2.0.89.centos701406.x86_64.rpm
# 下載並部署scala
curl -v -j -k -L http://downloads.lightbend.com/scala/2.11.8/scala-2.11.8.tgz -o scala-2.11.8.tgz
tar -zxvf scala-2.11.8.tgz
mv scala-2.11.8 /usr/local/bigdata/scala
# 下載並部署spark
curl -v -j -k -L http://d3kbcqa49mib13.cloudfront.net/spark-2.0.0-bin-hadoop2.6.tgz -o spark-2.0.0-bin-hadoop2.6.tgz
tar -zxvf spark-2.0.0-bin-hadoop2.6.tgz
mv spark-2.0.0-bin-hadoop2.6 /usr/local/bigdata/spark
# 下載並部署cassandra
curl -v -j -k -L http://apache.stu.edu.tw/cassandra/2.2.7/apache-cassandra-2.2.7-bin.tar.gz -o apache-cassandra-2.2.7-bin.tar.gz
tar -zxvf apache-cassandra-2.2.7-bin.tar.gz
mv apache-cassandra-2.2.7 /usr/local/bigdata/cassandra
```

ii. 環境變數設置

``` bash
sudo tee -a /etc/bashrc << "EOF"
# JAVA
export JAVA_HOME=/usr/java/jdk1.8.0_101
# ZOOKEEPER
export ZOOKEEPER_HOME=/usr/local/bigdata/zookeeper
# SCALA
export SCALA_HOME=/usr/local/bigdata/scala
# SPARK
export SPARK_HOME=/usr/local/bigdata/spark
# CASSANDRA
export CASSANDRA_HOME=/usr/local/bigdata/cassandra
# PATH
export PATH=$PATH:$JAVA_HOME:$ZOOKEEPER_HOME/bin:$SPARK_HOME/bin:$CASSANDRA_HOME/bin
```

iv. 配置Zookeeper

``` bash
# 複製zoo.cfg
cp $ZOOKEEPER_HOME/conf/zoo_sample.cfg $ZOOKEEPER_HOME/conf/zoo.cfg
# 傳入設定
tee $ZOOKEEPER_HOME/conf/zoo.cfg << "EOF"
dataDir=/usr/local/bigdata/zookeeper/data
server.1=cassSpark1:2888:3888
server.2=cassSpark2:2888:3888
server.3=cassSpark3:2888:3888
EOF

# 接著創立需要的資料夾，並新增檔案
mkdir $ZOOKEEPER_HOME/data
tee $ZOOKEEPER_HOME/data/myid << "EOF"
1
EOF
```

在cassSpark2跟cassSpark3分別設定為2跟3。

啟動zookeeper: 

``` bash
# 啟動zookeeper server
zkServer.sh start
ssh tester@cassSpark2 "zkServer.sh start"
ssh tester@cassSpark3 "zkServer.sh start"
```

再來是測試看看是否有部署成功，先輸入`zkCli.sh -server cassSpark1:2181,cassSpark2:2181,cassSpark3:2181`可以登錄到zookeeper的server上，如果是正常運作會看到下面的訊息：

``` bash
[zk: cassSpark1:2181,cassSpark2:2181,cassSpark3:2181(CONNECTED) 0]
```

此時試著輸入看看`create /test01 abcd`，然後輸入`ls /`看看是否會出現`[test01, zookeeper]`

如果是，zookeeper就是設定成功，如果中間有出現任何錯誤，則否

最後用`delete /test01`做刪除即可，然後用`quit`離開。

最後是設定開機自動啟動zookeeper server:

``` bash
sudo tee /etc/init.d/zookeeper << "EOF"
#!/bin/bash
#
# ZooKeeper
# 
# chkconfig: - 80 99
# description: zookeeper

# ZooKeeper install path (where you extracted the tarball)
ZOOKEEPER=/usr/local/bigdata/zookeeper

source /etc/rc.d/init.d/functions
source $ZOOKEEPER/bin/zkEnv.sh

RETVAL=0
PIDFILE=/var/run/zookeeper_server.pid
desc="ZooKeeper daemon"

start() {
  echo -n $"Starting $desc (zookeeper): "
  daemon $ZOOKEEPER/bin/zkServer.sh start
  RETVAL=$?
  echo
  [ $RETVAL -eq 0 ] && touch /var/lock/subsys/zookeeper
  return $RETVAL
}

stop() {
  echo -n $"Stopping $desc (zookeeper): "
  daemon $ZOOKEEPER/bin/zkServer.sh stop
  RETVAL=$?
  sleep 5
  echo
  [ $RETVAL -eq 0 ] && rm -f /var/lock/subsys/zookeeper $PIDFILE
}

restart() {
  stop
  start
}

get_pid() {
  cat "$PIDFILE"
}

checkstatus(){
  status -p $PIDFILE ${JAVA_HOME}/bin/java
  RETVAL=$?
}

condrestart(){
  [ -e /var/lock/subsys/zookeeper ] && restart || :
}

case "$1" in
  start)
    start
    ;;
  stop)
    stop
    ;;
  status)
    checkstatus
    ;;
  restart)
    restart
    ;;
  condrestart)
    condrestart
    ;;
  *)
    echo $"Usage: $0 {start|stop|status|restart|condrestart}"
    exit 1
esac

exit $RETVAL
EOF

# 然後使用下面指令讓這個script能夠自動跑：
sudo chmod +x /etc/init.d/zookeeper
sudo chkconfig --add zookeeper
sudo service zookeeper start
```

v. 配置mesos

下面的動作在cassSpark1, cassSpark2, cassSpark3這三台都要配置

``` bash
# 修改zookeeper
sudo tee /etc/mesos/zk << "EOF"
zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos
EOF

# 配置quorum
sudo tee /etc/mesos-master/quorum << "EOF"
2
EOF
```

ssh-copy-id -i ~/.ssh/id_rsa.pub cassSpark3

再來就是啟動了

``` bash
# 記得先啟動zookeeper server，再啟動mesos
# master要記得關掉slave (這裡的配置下不用關，指令供人參考用)
sudo systemctl disable mesos-master
sudo systemctl stop mesos-master
sudo service mesos-master restart
# 在cassSpark2,cassSpark3上啟動slave
# slave要記得關掉master (這裡的配置下不用關，指令供人參考用)
sudo systemctl disable mesos-slave
sudo systemctl stop mesos-slave
sudo service mesos-slave restart
```

iii. 配置scala and spark
      
``` bash
# 複製檔案
cp $SPARK_HOME/conf/spark-env.sh.template $SPARK_HOME/conf/spark-env.sh
cp $SPARK_HOME/conf/log4j.properties.template $SPARK_HOME/conf/log4j.properties
cp $SPARK_HOME/conf/spark-defaults.conf.template $SPARK_HOME/conf/spark-defaults.conf

# 傳入設定
tee -a $SPARK_HOME/conf/spark-env.sh << "EOF"
SPARK_LOCAL_DIRS=/usr/local/bigdata/spark
SPARK_SCALA_VERSION=2.11
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
spark.cores.max 3
spark.driver.memory 4g
spark.executor.memory 4g
EOF
```

Spark的master, slave就不用啟動了，直接用Mesos即可

如果之前用我的方式配置過自動啟動的話，請用下面的指令移除：

``` bash
# 移除master的服務
sudo systemctl stop spark-master.service
sudo rm /etc/systemd/system/multi-user.target.wants/spark-master.service
# 移除slave的服務
sudo systemctl stop spark-slave.service
sudo rm /etc/systemd/system/multi-user.target.wants/spark-slave.service
# 重新讀取開機的服務
sudo systemctl daemon-reload
```

至於cassandra的設置就都一樣，此處就不贅述，直接進測試

執行下面兩行，成功執行就是Mesos有裝成功

``` bash
MASTER=$(mesos-resolve `cat /etc/mesos/zk`)
mesos-execute --master=$MASTER --name="cluster-test" --command="sleep 5"
```

Mesos也可以透過去連接5050 port到目前的master上去，有出現網頁就是正常


再來測試Spark，用`spark-shell --master mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos`開啟spark-shell確定功能正常

``` scala
// 單純測試spark
val NUM_SAMPLES = 1000000
val count = sc.parallelize(1 to NUM_SAMPLES).map{i =>
  val x = Math.random()
  val y = Math.random()
  if (x*x + y*y < 1) 1 else 0
}.reduce(_ + _)
println("Pi is roughly " + 4.0 * count / NUM_SAMPLES)

// 測試與cassandra的connector
// 重啟sc
sc.stop()
// imprt需要套件
import com.datastax.spark.connector._, org.apache.spark.SparkContext, org.apache.spark.SparkContext._, org.apache.spark.SparkConf, com.datastax.spark.connector.cql.CassandraConnector
// 設定cassandra host
val conf = new SparkConf(true).set("spark.cassandra.connection.host", "192.168.0.121")
val sc = new SparkContext(conf)

// 創立keyspace, table，然後塞值
CassandraConnector(conf).withSessionDo { session =>
  session.execute("CREATE KEYSPACE IF NOT EXISTS test WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 2 }")
  session.execute("DROP TABLE IF EXISTS test.kv")
  session.execute("CREATE TABLE test.kv (key text PRIMARY KEY, value DOUBLE)")
  session.execute("INSERT INTO test.kv(key, value) VALUES ('key1', 1.0)")
  session.execute("INSERT INTO test.kv(key, value) VALUES ('key2', 2.5)")
}

// 印出cassandra的元素
val rdd = sc.cassandraTable("test", "kv")
println(rdd.first)
// 會出現CassandraRow{key: key1, value: 1.0}
val collection = sc.parallelize(Seq(("key3", 1.7), ("key4", 3.5)))
// save new data into cassandra
collection.saveToCassandra("test", "kv", SomeColumns("key", "value"))
// 印出塞完的結果
rdd.collect().foreach(row => println(s"Existing Data: $row"))
// Existing Data: CassandraRow{key: key3, value: 1.7}
// Existing Data: CassandraRow{key: key1, value: 1.0}
// Existing Data: CassandraRow{key: key4, value: 3.5}
// Existing Data: CassandraRow{key: key2, value: 2.5}

// 停止spark-shell，然後離開
sc.stop()
:quit
```

備註：

如果cluster不能對外連線的話，curl可以取得的檔案都先經由能夠連線的電腦取得

至於Mesos的dependencies，先在能夠對外連線的centos電腦上下

`sudo yum install --downloadonly --downloaddir=pkgs mesos-1.0.0-2.0.89.centos701406.x86_64.rpm`

這樣就會把要下載rpm檔案全部都載下來到`pkgs`的資料夾，這些在打包傳到cluster上

然後用`sudo yum install *.rpm`安裝，在安裝Mesos即可

