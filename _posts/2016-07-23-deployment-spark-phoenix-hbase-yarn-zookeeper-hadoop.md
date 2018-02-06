---
layout: post
title: "基於hadoop的spark, phoenix, hbase, yarn, zookeeper部署"
---

之前有一系列文章是在mint 17上部署R-hadoop的環境

這篇主要是在centos 7.2最小安裝下去部署hadoop, yarn, zookeeper, hbase, phoenix以及spark

一樣是透過VM做部署，所以目標是部署好其中一台後

再把那一台的映像檔做clone，變成slaves

1. 準備工作

    i. 安裝好VMware，然後新增一台VM (網路連接方式使用bridged即可)，引進centos 7.2安裝映像檔
    
    ii. 選擇最小安裝，並新增使用者: tester
    
    iii. 安裝完後要先configure：
    
a. 給予使用者sudoer權限
      
``` bash
su # 切換到root
visudo # 打開設定檔
# 打/root\tALL找到這行 root ALL=(ALL) ALL
# 在下面新增 tester ALL=(ALL) ALL
```

b. 網路設定
        
先查看自己電腦的網段是哪一個(使用撥接就無法，要透過IP分享器)

在cmd上找`ipconfig`就可以找到，像是我的電腦是192.168.0.111

預設閘道192.168.0.1，沒有設定DNS

接著用`ip a`看VM網路卡的裝置名稱

我的VM網路卡名稱是eno16777736

然後就使用`sudo ifup eno16777736`去啟用網路

接著使用`sudo vi /etc/sysconfig/network-scripts/ifcfg-eno16777736`去修改網路設定

改成下方這樣：
      
``` bash
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=eno16777736
UUID=213f0a79-5d73-45c5-afce-16b48e6c9c65
DEVICE=eno16777736
ONBOOT=yes
IPADDR=192.168.0.161
PREFIX=24
GATEWAY=192.168.0.1
DNS1=192.168.0.1
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_PRIVACY=no
```

之後使用`sudo service network restart`重啟網路服務

這樣網路設定就完成了。測試方式為：`ping 192.168.0.1`(DNS)跟

`ping www.google.com`就可以知道網路有沒有設定成功了。
    
    
c. 安裝ssh跟設定ssh資料夾權限
    
``` bash
# 安裝SSH
sudo yum -y install rsync openssh-server-*
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
  
d. 編輯/etc/hosts

``` bash
sudo tee -a /etc/hosts << "EOF"
192.168.0.161 sparkServer0
192.168.0.162 sparkServer1
192.168.0.163 sparkServer2
192.168.0.164 sparkServer3
EOF
```

e. 編輯/etc/hostname
    
``` bash
sudo vi /etc/hostname
# 對應的電腦修改成對應的名稱
```

f. 斷掉防火牆
    
``` bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld  
```

2. 開始部署
i. 關掉ip v6
   
``` bash
sudo tee -a /etc/sysctl.conf << "EOF"
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
```

ii. 下載檔案並移到適當位置
    
``` bash
# 下載並安裝java
curl -v -j -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.rpm -o jdk-8u101-linux-x64.rpm
sudo yum install -y jdk-8u101-linux-x64.rpm
# 下載並部署Hadoop
curl -v -j -k -L http://apache.stu.edu.tw/hadoop/common/hadoop-2.6.4/hadoop-2.6.4.tar.gz -o hadoop-2.6.4.tar.gz
tar -zxvf hadoop-2.6.4.tar.gz
sudo mv hadoop-2.6.4 /usr/local/hadoop
sudo chown -R tester /usr/local/hadoop
# 下載並部署zookeeper
curl -v -j -k -L http://apache.stu.edu.tw/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz -o zookeeper-3.4.8.tar.gz
tar -zxvf zookeeper-3.4.8.tar.gz
sudo mv zookeeper-3.4.8 /usr/local/zookeeper
sudo chown -R tester /usr/local/zookeeper
# 下載並部署HBase
curl -v -j -k -L http://apache.stu.edu.tw/hbase/0.98.20/hbase-0.98.20-hadoop2-bin.tar.gz -o hbase-0.98.20-hadoop2-bin.tar.gz
tar -zxvf hbase-0.98.20-hadoop2-bin.tar.gz
sudo mv hbase-0.98.20-hadoop2 /usr/local/hbase
sudo chown -R tester /usr/local/hbase
# 下載並部署phoenix
curl -v -j -k -L  http://apache.stu.edu.tw/phoenix/phoenix-4.7.0-HBase-1.1/bin/phoenix-4.7.0-HBase-1.1-bin.tar.gz -o phoenix-4.7.0-HBase-1.1-bin.tar.gz
tar -zxvf phoenix-4.7.0-HBase-1.1-bin.tar.gz
## phoenix部署比較不同，後面再處理
# 下載並部署scala
curl -v -j -k -L http://downloads.lightbend.com/scala/2.10.6/scala-2.10.6.tgz -o scala-2.10.6.tgz
tar -zxvf scala-2.10.6.tgz
sudo mv scala-2.10.6 /usr/local/scala
sudo chown -R tester /usr/local/scala
# 下載並部署spark
curl -v -j -k -L   http://apache.stu.edu.tw/spark/spark-1.6.2/spark-1.6.2-bin-hadoop2.6.tgz -o spark-1.6.2-bin-hadoop2.6.tgz
tar -zxvf spark-1.6.2-bin-hadoop2.6.tgz
sudo mv spark-1.6.2-bin-hadoop2.6 /usr/local/spark
sudo chown -R tester /usr/local/spark
```
   
iii. 環境變數設置
    
``` bash
sudo tee -a /etc/bashrc << "EOF"
# JAVA
export JAVA_HOME=/usr/java/jdk1.8.0_101
# HADOOP
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"
export HADOOP_PREFIX=$HADOOP_HOME
export HADOOP_PID_DIR=$HADOOP_HOME/pids/
# ZOOKEEPER
export ZOOKEEPER_HOME=/usr/local/zookeeper
# HBASE
export HBASE_HOME=/usr/local/hbase
export HBASE_CLASSPATH=$HBASE_HOME/conf
export HBASE_PID_DIR=$HBASE_HOME/pids 
export HBASE_MANAGES_ZK=false
# PHOENIX
export PHOENIX_HOME=/usr/local/phoenix
export PHOENIX_CLASSPATH=$PHOENIX_HOME/lib
export PHOENIX_LIB_DIR=$PHOENIX_HOME/lib
# SCALA
export SCALA_HOME=/usr/local/scala
# SPARK
export SPARK_HOME=/usr/local/spark
# PATH
export PATH=$PATH:$JAVA_HOME:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$ZOOKEEPER_HOME/bin:$HBASE_HOME/bin:$SCALA_HOME/bin:$SPARK_HOME/bin:$PHOENIX_HOME/bin
EOF
source /etc/bashrc
```
  
iv. 配置Hadoop
  
a. core-site.xml
用`vi $HADOOP_CONF_DIR/core-site.xml`編輯，改成下面這樣：
  
``` xml
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://sparkServer0:9000</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/usr/local/hadoop/tmp</value>
  </property>
</configuration>
```

b. mapred-site.xml
先用`cp $HADOOP_CONF_DIR/mapred-site.xml.template $HADOOP_CONF_DIR/mapred-site.xml`，然後用`vi $HADOOP_CONF_DIR/mapred-site.xml`編輯，改成下面這樣：
    
``` xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

c. hdfs-site.xml
用`vi $HADOOP_CONF_DIR/hdfs-site.xml`編輯，改成下面這樣：
        
``` xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.permissions</name>
    <value>false</value>
  </property>
  <property>
    <name>dfs.name.dir</name>
    <value>file:///usr/local/hadoop/tmp/name</value>
  </property>
  <property>
    <name>dfs.data.dir</name>
    <value>file:///usr/local/hadoop/tmp/data</value>
  </property>
  <property>
    <name>dfs.namenode.checkpoint.dir</name>
    <value>file:///usr/local/hadoop/tmp/name/chkpt</value>
  </property>
</configuration>
```

建立node需要的資料夾：
        
``` bash
mkdir -p $HADOOP_HOME/tmp
mkdir -p $HADOOP_HOME/tmp/data
mkdir -p $HADOOP_HOME/tmp/name
```

d. yarn-site.xml
用`vi $HADOOP_CONF_DIR/yarn-site.xml`編輯，改成下面這樣：
    
``` xml
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>sparkServer0</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
</configuration>
```

e. 配置slaves
        
``` bash
# 傳入slaves的電腦名稱
tee $HADOOP_CONF_DIR/slaves << "EOF"
sparkServer1
sparkServer2
sparkServer3
EOF
```

ii. 配置Zookeeper
先用`cp $ZOOKEEPER_HOME/conf/zoo_sample.cfg $ZOOKEEPER_HOME/conf/zoo.cfg`，然後用`vi $ZOOKEEPER_HOME/conf/zoo.cfg`編輯，改成下面這樣：

``` bash
dataDir=/usr/local/zookeeper/data
server.1=sparkServer0:2888:3888
server.1=sparkServer1:2888:3888
server.1=sparkServer2:2888:3888
# 接著創立需要的資料夾，並新增檔案
mkdir $ZOOKEEPER_HOME/data
tee $ZOOKEEPER_HOME/data/myid << "EOF"
1
EOF
```

在sparkServer2跟sparkServer3分別設定為2跟3。

iii. 配置HBase
用`vi $HBASE_HOME/conf/hbase-site.xml`編輯，改成下面這樣：
      
``` xml
<configuration>
  <property>
    <name>hbase.master</name>
    <value>sparkServer0:60000</value>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://sparkServer0:9000/hbase</value>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>sparkServer0</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>file:///usr/local/zookeeper/data</value>
  </property>
</configuration>
```

接著，用`cp $HADOOP_CONF_DIR/slaves $HBASE_HOME/conf/regionservers`複製hadoop的slaves
  
iv. 配置phoenix
      
``` bash
# 縮短名稱
mv phoenix-4.7.0-HBase-1.1-bin phoenix-4.7.0
# 複製lib檔案到HBase/lib下
sudo cp phoenix/phoenix-4.7.0-HBase-1.1-server.jar $HBASE_HOME/lib
# 複製hbase設定到phoenix下
cp $HBASE_HOME/conf/hbase-site.xml $PHOENIX_HOME/bin
cp $HBASE_HOME/conf/hbase-env.sh $PHOENIX_HOME/bin
# 複製lib檔, bin
sudo mkdir /usr/local/phoenix
sudo chown -R tester $PHOENIX_HOME
mkdir $PHOENIX_HOME/lib
cp phoenix-4.7.0/phoenix-4.7.0-HBase-1.1-client-spark.jar $PHOENIX_HOME/lib/phoenix-4.7.0-HBase-1.1-client-spark.jar
cp phoenix-4.7.0/phoenix-4.7.0-HBase-1.1-client.jar $PHOENIX_HOME/lib/phoenix-4.7.0-HBase-1.1-client.jar
cp -R phoenix-4.7.0/bin $PHOENIX_HOME
cp -R phoenix-4.7.0/examples $PHOENIX_HOME
cp phoenix-4.7.0/LICENSE $PHOENIX_HOME/LICENSE
chmod +x $PHOENIX_HOME/bin/*.py
```

並且在用`vi $PHOENIX_HOME/bin/hbase-site.xml`加入下面的設定
      
``` xml
<property>
<name>fs.hdfs.impl</name>
<value>org.apache.hadoop.hdfs.DistributedFileSystem</value>
</property>
```
  
v. 配置scala and spark
      
``` bash
# 複製hadoop的slaves
cp $HADOOP_CONF_DIR/slaves $SPARK_HOME/conf/slaves

# 複製檔案
cp $SPARK_HOME/conf/spark-env.sh.template $SPARK_HOME/conf/spark-env.sh
# 傳入設定
sudo tee -a $SPARK_HOME/conf/spark-env.sh << "EOF"
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
SPARK_MASTER_IP=sparkServer0
SPARK_LOCAL_DIRS=/usr/local/spark
SPARK_DRIVER_MEMORY=2G
EOF
```
  
vi. slaves的部署

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
for hostname in `cat $HADOOP_CONF_DIR/slaves`; do
  ssh-copy-id -i ~/.ssh/id_rsa.pub $hostname
done
EOF
```

4. 啟動hadoop server / zookeeper server / hbase server / 

``` bash
# 執行hadoop的namenode format
hdfs namenode -format 
# 啟動hadoop server
start-dfs.sh & start-yarn.sh
# 啟動zookeeper server
zkServer.sh start
ssh tester@cassSpark2 "zkServer.sh start"
ssh tester@cassSpark3 "zkServer.sh start"
# 啟動hbase server
start-hbase.sh
```

5. 測試
i. Hadoop MapReduce例子 - pi estimation
  
``` bash
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.4.jar pi 10 1000
# output會像下面這樣
# Job Finished in 2.413 seconds
# Estimated value of Pi is 3.14800000000000000000
```

ii. zookeeper
再來是測試看看zookeeper是否有部署成功，先輸入`zkCli.sh -server cassSpark1:2181,cassSpark2:2181,cassSpark3:2181`可以登錄到zookeeper的server上，如果是正常運作會看到下面的訊息：

``` bash
[zk: cassSpark1:2181,cassSpark2:2181,cassSpark3:2181(CONNECTED) 0]
```

此時試著輸入看看`create /test01 abcd`，然後輸入`ls /`看看是否會出現`[test01, zookeeper]`

如果是，zookeeper就是設定成功，如果中間有出現任何錯誤，則否

最後用`delete /test01`做刪除即可，然後用`quit`離開。

iii. HBase
鍵入`hbase shell`會出現下面的訊息：
      
``` bash
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hbase/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 1.1.5, r239b80456118175b340b2e562a5568b5c744252e, Sun May  8 20:29:26 PDT 2016

hbase(main):001:0>
```

測試建表、塞資料、擷取資料跟刪除表(#後面是會出現的訊息)：

``` bash
# 建表
create 'testData','cf'  
# 0 row(s) in 1.3420 seconds
# 
# => Hbase::Table - testData

# 塞資料
put 'testData','row1','cf:a','value1'  
# 0 row(s) in 0.1170 seconds
put 'testData','row2','cf:b','value2'  
# 0 row(s) in 0.0130 seconds
put 'testData','row3','cf:c','value3'  
# 0 row(s) in 0.0240 seconds

# 列出所有表
list
# TABLE
# testData
# 1 row(s) in 0.0040 seconds
# 
# => ["testData"]

# 列出資料
scan 'testData'
# ROW                             COLUMN+CELL
#  row1                           column=cf:a, timestamp=1469372859574, value=value1
#  row2                           column=cf:b, timestamp=1469372859613, value=value2
#  row3                           column=cf:c, timestamp=1469372860195, value=value3
# 3 row(s) in 0.0450 seconds

# 擷取特定資料
get 'testData','row1'  
# COLUMN                          CELL
#  cf:a                           timestamp=1469372859574, value=value1
# 1 row(s) in 0.0200 seconds

# 刪除表
## 先無效表
disable 'testData'
## 再刪除表
drop 'testData'

# 列出所有表
list
# TABLE
# 0 row(s) in 0.0070 seconds
# 
# => []
```

最後可以用`exit`離開hbase shell。

iv. phoenix
  
``` bash
# 創表SQL
cat $PHOENIX_HOME/examples/STOCK_SYMBOL.sql
# -- creates stock table with single row
# CREATE TABLE IF NOT EXISTS STOCK_SYMBOL (SYMBOL VARCHAR NOT NULL PRIMARY KEY, COMPANY VARCHAR);
# UPSERT INTO STOCK_SYMBOL VALUES ('CRM','SalesForce.com');
# SELECT * FROM STOCK_SYMBOL;

# create table
psql.py sparkServer0:2181 $PHOENIX_HOME/examples/STOCK_SYMBOL.sql
# SLF4J: Class path contains multiple SLF4J bindings.
# SLF4J: Found binding in [jar:file:/usr/local/phoenix/lib/phoenix-4.7.0-HBase-1.1-client.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
# 16/07/24 23:25:53 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
# no rows upserted
# Time: 1.246 sec(s)
# 
# 1 row upserted
# Time: 0.102 sec(s)
# 
# SYMBOL                                   COMPANY
# ---------------------------------------- ----------------------------------------
# CRM                                      SalesForce.com
# Time: 0.028 sec(s)

# 要插入的資料
cat $PHOENIX_HOME/examples/STOCK_SYMBOL.csv
# AAPL,APPLE Inc.
# CRM,SALESFORCE
# GOOG,Google
# HOG,Harlet-Davidson Inc.
# HPQ,Hewlett Packard
# INTC,Intel
# MSFT,Microsoft
# WAG,Walgreens
# WMT,Walmart

# insert data
psql.py -t STOCK_SYMBOL sparkServer0:2181 $PHOENIX_HOME/examples/STOCK_SYMBOL.csv
# SLF4J: Class path contains multiple SLF4J bindings.
# SLF4J: Found binding in [jar:file:/usr/local/phoenix/lib/phoenix-4.7.0-HBase-1.1-client.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
# 16/07/24 23:32:14 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
# csv columns from database.
# CSV Upsert complete. 9 rows upserted
# Time: 0.074 sec(s)

# 查詢資料
sqlline.py sparkServer0:2181
# Setting property: [incremental, false]
# Setting property: [isolation, TRANSACTION_READ_COMMITTED]
# issuing: !connect jdbc:phoenix:sparkServer0:2181 none none org.apache.phoenix.jdbc.PhoenixDriver
# Connecting to jdbc:phoenix:sparkServer0:2181
# SLF4J: Class path contains multiple SLF4J bindings.
# SLF4J: Found binding in [jar:file:/usr/local/phoenix/lib/phoenix-4.7.0-HBase-1.1-client.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
# 16/07/24 23:34:17 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
# Connected to: Phoenix (version 4.7)
# Driver: PhoenixEmbeddedDriver (version 4.7)
# Autocommit status: true
# Transaction isolation: TRANSACTION_READ_COMMITTED
# Building list of tables and columns for tab-completion (set fastconnect to true to skip)...
# 85/85 (100%) Done
# Done
# sqlline version 1.1.8

### 計算COUNT
# 0: jdbc:phoenix:sparkServer0:2181> select count(*) from STOCK_SYMBOL;
## +-----------+
## | COUNT(1)  |
## +-----------+
## | 9         |
## +-----------+
## 1 row selected (0.046 seconds)

### 拉出表
# 0: jdbc:phoenix:sparkServer0:2181> select * from STOCK_SYMBOL;
## +---------+-----------------------+
## | SYMBOL  |        COMPANY        |
## +---------+-----------------------+
## | AAPL    | APPLE Inc.            |
## | CRM     | SALESFORCE            |
## | GOOG    | Google                |
## | HOG     | Harlet-Davidson Inc.  |
## | HPQ     | Hewlett Packard       |
## | INTC    | Intel                 |
## | MSFT    | Microsoft             |
## | WAG     | Walgreens             |
## | WMT     | Walmart               |
## +---------+-----------------------+
## 9 rows selected (0.03 seconds)
```

離開請按`CTRL+Z`或是用`!quit`，輸入`exit()`是沒用的。接下來，我們試試看在`hbase`裡面看不看的到我們剛剛插入的表，打`hbase shell`進入，然後開始測試：
  
``` bash
# scan看看就可以發現表已經存進來了
# hbase(main):001:0> scan 'STOCK_SYMBOL'
## ROW                             COLUMN+CELL
##  AAPL                           column=0:COMPANY, timestamp=1469374336318, value=APPLE Inc.
##  AAPL                           column=0:_0, timestamp=1469374336318, value=x
##  CRM                            column=0:COMPANY, timestamp=1469374336318, value=SALESFORCE
##  CRM                            column=0:_0, timestamp=1469374336318, value=x
##  GOOG                           column=0:COMPANY, timestamp=1469374336318, value=Google
##  GOOG                           column=0:_0, timestamp=1469374336318, value=x
##  HOG                            column=0:COMPANY, timestamp=1469374336318, value=Harlet-Davidson Inc.
##  HOG                            column=0:_0, timestamp=1469374336318, value=x
##  HPQ                            column=0:COMPANY, timestamp=1469374336318, value=Hewlett Packard
##  HPQ                            column=0:_0, timestamp=1469374336318, value=x
##  INTC                           column=0:COMPANY, timestamp=1469374336318, value=Intel
##  INTC                           column=0:_0, timestamp=1469374336318, value=x
##  MSFT                           column=0:COMPANY, timestamp=1469374336318, value=Microsoft
##  MSFT                           column=0:_0, timestamp=1469374336318, value=x
##  WAG                            column=0:COMPANY, timestamp=1469374336318, value=Walgreens
##  WAG                            column=0:_0, timestamp=1469374336318, value=x
##  WMT                            column=0:COMPANY, timestamp=1469374336318, value=Walmart
##  WMT                            column=0:_0, timestamp=1469374336318, value=x
## 9 row(s) in 0.1840 seconds
```
   
最後是進去`sqlline.py`去刪掉剛剛建立的表：
      
``` bash
sqlline.py sparkServer0:2181
# 0: jdbc:phoenix:sparkServer0:2181> DROP TABLE STOCK_SYMBOL;
## No rows affected (3.556 seconds)

# 列出現有的表
# 0: jdbc:phoenix:sparkServer0:2181> !tables 
## +------------+--------------+-------------+---------------+----------+------------+----------------------------+----------+
## | TABLE_CAT  | TABLE_SCHEM  | TABLE_NAME  |  TABLE_TYPE   | REMARKS  | TYPE_NAME  | SELF_REFERENCING_COL_NAME  | REF_GENE |
## +------------+--------------+-------------+---------------+----------+------------+----------------------------+----------+
## |            | SYSTEM       | CATALOG     | SYSTEM TABLE  |          |            |                            |          |
## |            | SYSTEM       | FUNCTION    | SYSTEM TABLE  |          |            |                            |          |
## |            | SYSTEM       | SEQUENCE    | SYSTEM TABLE  |          |            |                            |          |
## |            | SYSTEM       | STATS       | SYSTEM TABLE  |          |            |                            |          |
## +------------+--------------+-------------+---------------+----------+------------+----------------------------+----------+
```

v. spark 
利用spark提供的例子去測試看看 (記得要先開啟hadoop)
      
``` bash
spark-submit --class org.apache.spark.examples.SparkPi \
  --deploy-mode cluster  \
  --master yarn  \
  --num-executors 3 \
  --driver-memory 4g \
  --executor-memory 2g \
  --executor-cores 2 \
   /usr/local/spark/lib/spark-examples-1.6.2-hadoop2.6.0.jar
```

最後就可以看到任務成功，如圖所示：
![](/images/sparkSucceeded_1.PNG)

用上面的連結就可以看到任務的成功情況：
![](/images/sparkSucceeded_2.PNG)

5. Reference

    1. vm:
        1. https://dotblogs.com.tw/jhsiao/2013/09/26/120726
        1. https://read01.com/DQO2jg.html
        1. https://read01.com/KEQ6GP.html

    2. centos:
        1. https://www.howtoforge.com/creating_a_local_yum_repository_centos
        1.  http://www.serverlab.ca/tutorials/linux/network-services/creating-a-yum-repository-server-for-red-hat-and-centos/
        1. http://ask.xmodulo.com/change-network-interface-name-centos7.html
        1. http://yenpai.idis.com.tw/archives/240-%E6%95%99%E5%AD%B8-centos-6-3-%E5%AE%89%E8%A3%9D-2%E7%B6%B2%E8%B7%AF%E8%A8%AD%E5%AE%9A%E7%AF%87?doing_wp_cron=1468854397.3452270030975341796875

    3. hadoop, hbase, zookeeper:
        1. http://tsai-cookie.blogspot.tw/2015/09/hadoop-hbase-hive.html
        1. http://lyhpcha.pixnet.net/blog/post/60903916-hadoop%E3%80%81zookeeper%E3%80%81hbase%E5%AE%89%E8%A3%9D%E9%85%8D%E7%BD%AE%E8%AA%AA%E6%98%8E
        1. http://blog.csdn.net/smile0198/article/details/17660205

    4. phoenix:
        1. http://www.aboutyun.com/thread-12403-1-1.html
        1. http://www.zhangshuai.top/2015/09/01/phoenix%E5%AE%89%E8%A3%85%E4%B8%8E%E4%BD%BF%E7%94%A8%E6%96%87%E6%A1%A3/
        1. http://toutiao.com/i6222878197948613122/
        1. https://phoenix.apache.org/faq.html
        1. http://ju.outofmemory.cn/entry/237491

