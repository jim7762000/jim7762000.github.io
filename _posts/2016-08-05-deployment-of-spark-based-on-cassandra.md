---
layout: post
title: "基於cassandra的spark環境部署"
---

之前的部屬是base on hadoop所建立的spark環境

這一篇會從頭建立基於cassandra的spark環境

一樣是透過VM安裝minimal system of centos

再部署好其中一台做為spark的master，將該台的映像檔做clone，變成slaves

Cassandra則部署到全部的節點上

1. 準備工作

    1. 安裝好VMware，然後新增一台VM (網路連接方式使用bridged即可)，引進centos 7.2安裝映像檔
    1. 安裝centos的時候先將一些設定configure好，之後就不需要用小黑框慢慢config了
    1. 先設置網路，第一個分頁的自動連線打勾、第四頁的IPv4選Manual，填上IP、Net mask、Gateway跟DNS，接著點進硬碟確定配置，然後設定時區(你可以不設定，就用美國紐約)
    1. 按下一步安裝，此時可以新增使用者跟設置root密碼，我這裡新增一個使用者tester
    
安裝後重開機後，先幫你自己的使用者帳號開權限：
          
``` bash
su # 切換到root
visudo # 打開設定檔
# 打/root\tALL找到這行 root ALL=(ALL) ALL
# 在下面新增 tester ALL=(ALL) ALL
```

而ssh已經有了，就不用安裝了，直接產生key：
    
``` bash
# 產生SSH Key
ssh-keygen -t rsa -P ""
# 授權SSH Key
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
 
# 設定.ssh的權限
sudo chmod 700 /home/tester
sudo chmod 700 /home/tester/.ssh
sudo chmod 644 /home/tester/.ssh/authorized_keys
sudo chmod 600 /home/tester/.ssh/id_rsa
sudo service sshd restart
```
  
再來是讓電腦認得自己，並讓其他電腦認得自己

`echo "cassSpark1" | sudo tee /etc/hostname` (每台電腦用不同名字)

``` bash
sudo tee -a /etc/hosts << "EOF"
192.168.0.121 cassSpark1
192.168.0.122 cassSpark2
192.168.0.123 cassSpark3
EOF
```

最後是斷掉防火牆
    
``` bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld  
```

然後用`sudo reboot`重開機

2. 開始部署

Note: spark-cassandra-connector 1.6是用spark 1.6.1, scala 2.10.5編譯出來的

所以我這版本就用一樣的，避免版本不同，沒有align情況下出現methodNotFound的exception

i. 下載檔案並移到適當位置
    
``` bash
sudo mkdir /usr/local/bigdata
sudo chown -R tester /usr/local/bigdata

# 下載並安裝java
curl -v -j -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.rpm -o jdk-8u101-linux-x64.rpm
sudo yum install -y jdk-8u101-linux-x64.rpm
# 下載並部署scala
curl -v -j -k -L http://downloads.lightbend.com/scala/2.10.5/scala-2.10.5.tgz -o scala-2.10.5.tgz
tar -zxvf scala-2.10.5.tgz
sudo mkdir /usr/local/scala
sudo mv scala-2.10.5 /usr/local/scala/scala-2.10
# 下載並部署spark
curl -v -j -k -L http://d3kbcqa49mib13.cloudfront.net/spark-1.6.1-bin-hadoop2.6.tgz -o spark-1.6.1-bin-hadoop2.6.tgz
tar -zxvf spark-1.6.1-bin-hadoop2.6.tgz
mv spark-1.6.1-bin-hadoop2.6 /usr/local/bigdata/spark
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
export SCALA_HOME=/usr/local/scala/scala-2.10
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
# 複製hadoop的slaves
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
SPARK_DRIVER_MEMORY=3G
SPARK_WORKER_CORES=2
SPARK_WORKER_MEMORY=3g
EOF

# download dependencies of spark-cassandra-connector
mkdir $SPARK_HOME/extraClass

curl -v -j -k -L http://search.maven.org/remotecontent?filepath=org/apache/ivy/ivy/2.4.0/ivy-2.4.0.jar -o ivy-2.4.0.jar
curl -v -j -k -L http://search.maven.org/remotecontent?filepath=org/apache/ivy/ivy/2.4.0/ivy-2.4.0.jar -o ivy-2.4.0.jar
curl -v -j -k -L http://search.maven.org/remotecontent?filepath=com/datastax/spark/spark-cassandra-connector_2.10/1.6.0/spark-cassandra-connector_2.10-1.6.0.jar  -o spark-cassandra-connector_2.10-1.6.0.jar
# remove spark-related jar
rm spark-catalyst_2.10-1.6.1.jar
rm spark-core_2.10-1.6.1.jar
rm spark-hive_2.10-1.6.1.jar
rm spark-launcher_2.10-1.6.1.jar
rm spark-network-common_2.10-1.6.1.jar
rm spark-network-shuffle_2.10-1.6.1.jar
rm spark-sql_2.10-1.6.1.jar
rm spark-streaming_2.10-1.6.1.jar
rm spark-unsafe_2.10-1.6.1.jar
rm datanucleus-*

java -jar ivy-2.3.0.jar -dependency com.datastax.cassandra spark-cassandra-connector_2.10 1.6.0 -retrieve "[artifact]-[revision](-[classifier]).[ext]"
rm -f *-{sources,javadoc}.jar

# add jars to spark-defaults.conf
jarsToImport=$(echo $SPARK_HOME/extraClass/*.jar | sed 's/ /:/g')
echo "spark.executor.extraClassPath $jarsToImport" >> $SPARK_HOME/conf/spark-defaults.conf
echo "spark.driver.extraClassPath $jarsToImport" >> $SPARK_HOME/conf/spark-defaults.conf
echo "spark.driver.extraLibraryPath $SPARK_HOME/extraClass" >> $SPARK_HOME/conf/spark-defaults.conf
echo "spark.executor.extraLibraryPath $SPARK_HOME/extraClass" >> $SPARK_HOME/conf/spark-defaults.conf
jarsToImport2=$(echo $SPARK_HOME/extraClass/*.jar | sed 's/ /,/g')
echo "spark.jars $jarsToImport2" >> $SPARK_HOME/conf/spark-defaults.conf

# 複製到各台
scp -r /usr/local/bigdata/spark tester@cassSpark2:/usr/local/bigdata
scp -r /usr/local/bigdata/spark tester@cassSpark3:/usr/local/bigdata
```

iv. slaves的部署

因為是VM，所以剩下的就是把映像檔clone複製成各個nodes，然後針對需要個別配置的地方做配置：

``` bash
# 改hostname
sudo vi /etc/hostname
# 改網路設定
sudo vi /etc/sysconfig/network-scripts/ifcfg-eno16777736
# 配置玩各台電腦，並透過`sudo service network restart`重啟網路服務後
# 生成新的SSH key
ssh-keygen -t rsa -P ""
# 在sparkServer0，把他的ssh key傳到各台電腦去
tee run.sh << "EOF"
#!/bin/bash
for hostname in `cat $SPARK_HOME/conf/slaves`; do
  ssh-copy-id -i ~/.ssh/id_rsa.pub $hostname
done
EOF
bash ./run.sh
```

v. 設置Cassandra

使用`vi $CASSANDRA_HOME/conf/cassandra.yaml`去改設定檔，改的部分如下：

``` yaml
# first place:
cluster_name: 'cassSparkServer'

# second place:
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          - seeds: "192.168.0.121,192.168.0.122,192.168.0.123"
          
# third place:
listen_address: 192.168.0.121

# fourth place:
rpc_address: 192.168.0.121

# fifth place:
endpoint_snitch: GossipingPropertyFileSnitch
```

設置多台的做法：

```
scp -rp $CASSANDRA_HOME/* tester@cassSpark2:/usr/local/bigdata/cassandra
scp -rp $CASSANDRA_HOME/* tester@cassSpark3:/usr/local/bigdata/cassandra
ssh tester@cassSpark2 "sed -i -e 's/: 192.168.0.121/: 192.168.0.122/g' /usr/local/bigdata/cassandra/conf/cassandra.yaml"
ssh tester@cassSpark3 "sed -i -e 's/: 192.168.0.121/: 192.168.0.123/g' /usr/local/bigdata/cassandra/conf/cassandra.yaml"
```

vi. 設置Cassandra自動開機啟動

開機自動啟動Cassandra的script(用`sudo vi /etc/init.d/cassandra`去create)：

``` bash 
#!/bin/bash
# chkconfig: - 79 01
# description: Cassandra

. /etc/rc.d/init.d/functions

CASSANDRA_HOME=/usr/local/bigdata/cassandra
CASSANDRA_BIN=$CASSANDRA_HOME/bin/cassandra
CASSANDRA_NODETOOL=$CASSANDRA_HOME/bin/nodetool
CASSANDRA_LOG=$CASSANDRA_HOME/logs/cassandra.log
CASSANDRA_PID=/var/run/cassandra.pid
CASSANDRA_LOCK=/var/lock/subsys/cassandra
PROGRAM="cassandra"

if [ ! -f $CASSANDRA_BIN ]; then
  echo "File not found: $CASSANDRA_BIN"
  exit 1
fi

RETVAL=0

start() {
  if [ -f $CASSANDRA_PID ] && checkpid `cat $CASSANDRA_PID`; then
    echo "Cassandra is already running."
    exit 0
  fi
  echo -n $"Starting $PROGRAM: "
  daemon $CASSANDRA_BIN -p $CASSANDRA_PID >> $CASSANDRA_LOG 2>&1
  usleep 500000
  RETVAL=$?
  if [ $RETVAL -eq 0 ]; then
    touch $CASSANDRA_LOCK
    echo_success
  else
    echo_failure
  fi
  echo
  return $RETVAL
}

stop() {
  if [ ! -f $CASSANDRA_PID ]; then
    echo "Cassandra is already stopped."
    exit 0
  fi
  echo -n $"Stopping $PROGRAM: "
  $CASSANDRA_NODETOOL -h 127.0.0.1 decommission
  if kill `cat $CASSANDRA_PID`; then
    RETVAL=0
    rm -f $CASSANDRA_LOCK
    echo_success
  else
    RETVAL=1
    echo_failure
  fi
  echo
  [ $RETVAL = 0 ]
}

status_fn() {
  if [ -f $CASSANDRA_PID ] && checkpid `cat $CASSANDRA_PID`; then
    echo "Cassandra is running."
    exit 0
  else
    echo "Cassandra is stopped."
    exit 1
  fi
}

case "$1" in
  start)
    start
    ;;
  stop)
    stop
    ;;
  status)
    status_fn
    ;;
  restart)
    stop
    start
    ;;
  *)
    echo $"Usage: $PROGRAM {start|stop|restart|status}"
    RETVAL=3
esac

exit $RETVAL
```

然後使用下面指令讓這個script能夠自動跑：

``` bash 
sudo chmod +x /etc/init.d/cassandra
sudo chkconfig --add cassandra
sudo service cassandra start
```

接著用`nodetool status`可以確定一下是不是都有跑起來，顯示資訊如下：

``` bash
nodetool status
# Datacenter: dc1
# ===============
# Status=Up/Down
# |/ State=Normal/Leaving/Joining/Moving
# --  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
# UN  192.168.0.121  202.17 KB  256          66.7%             265daa7e-8039-46f4-842e-4255ec18be19  rack1
# UN  192.168.0.122  185.65 KB  256          66.4%             0fed9978-0097-417c-b0a8-dcd7be2b2c10  rack1
# UN  192.168.0.123  207.79 KB  256          66.9%             cf7b47b3-5d50-444e-99b4-843f9a5dadb2  rack1
```

vii. 設置spark自動開機

在spark的master (你的User跟Group那裏記得要修改):

```
sudo tee -a /etc/systemd/system/multi-user.target.wants/spark-master.service << "EOF"
[Unit]
Description=Spark Master
After=network.target
 
[Service]
Type=forking
User=tester
Group=tester
ExecStart=/usr/local/bigdata/spark/sbin/start-master.sh
StandardOutput=journal
StandardError=journal
LimitNOFILE=infinity
LimitMEMLOCK=infinity
LimitNPROC=infinity
LimitAS=infinity
 
[Install]
WantedBy=multi-user.target
EOF
```

在spark的master跟slaves:

```
sudo tee -a /etc/systemd/system/multi-user.target.wants/spark-slave.service << "EOF"
[Unit]
Description=Spark Slave
After=network.target
 
[Service]
Type=forking
User=tester
Group=tester
ExecStart=/usr/local/bigdata/spark/sbin/start-slave.sh spark://cassSpark1:7077
StandardOutput=journal
StandardError=journal
LimitNOFILE=infinity
LimitMEMLOCK=infinity
LimitNPROC=infinity
LimitAS=infinity
CPUAccounting=true
CPUShares=100

[Install]
WantedBy=multi-user.target
EOF
```

然後全部node執行`sudo systemctl daemon-reload`

master執行`sudo systemctl start spark-master.service`

master跟slave都執行`sudo systemctl start spark-slave.service`

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

Intellij的application不能直接跑，所以先用intellij的SBT Task: package

會在專案目錄/target/scala-2.10下面產生一個jar檔，我這裡是`test_cassspark_2.10-1.0.jar`

把這個jar上傳到cluster上，然後用下面指令就可以成功運行：

``` bash
spark-submit --class cassSpark test_cassspark_2.10-1.0.jar --master spark://192.168.0.121:7077
```

4. Ref
    1. https://github.com/datastax/spark-cassandra-connector/blob/master/doc/0_quick_start.md
    2. https://tobert.github.io/post/2014-07-15-installing-cassandra-spark-stack.html


