---
layout: post
title: "重返Hadoop Architecture (設定HA群集)"
---

用完Cassandra的primary key跟secondary index之後

覺得Cassandra適合表格有固定query pattern做使用才方便

primary key中的partition key在query時都要用到

clustering key要照順序使用，secondary index不能做複合查詢

更不用提ALLOW FILTERING的功能帶來的崩潰效能

我的用途真的很難用Cassandra去滿足，也不願意透過Spark SQL去使用

(畢竟Spark SQL要吃的資源也不少)

因此，我就survey了SQL on Hadoop的solutions...

而這一篇會先forcus在Hadoop的群集建立

SQL on Hadoop將會在下一篇介紹

這一篇會非常的長，因為Hadoop之前沒有完全的建好 (讓它能夠開機自動啟動，並且有HA的能力)

這一篇斷斷續續寫了三週，一直提不太起勁一口氣完成他

廢話不多說，下面是本文


配置架構：

1. Hadoop的HA只有兩台電腦、YARN也是同理，並且配置hdfs HA的電腦要啟動zkfc的服務
2. 每一台都要配置Journal node去讓資料同步、並且在Hadoop Master當機時能夠做自動切換
3. Zookeeper要設定成奇數台

重新佈署：

0. 部署之前的環境設定

ssh部分要實現全部都能互相連接，不能只有master能夠連到slaves

因此，這裡給一個簡單的script去做key的傳遞

``` bash
# 每一台都執行完下面兩個指令後
ssh-keygen -t rsa -P ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
# 在master上跑

tee ~/all_hosts << "EOF"
cassSpark1
cassSpark2
cassSpark3
EOF
scp ~/all_hosts tester@cassSpark2:~/
scp ~/all_hosts tester@cassSpark2:~/

# 然後在每一台都跑下面這個指令(要打很多次密碼)
for hostname in `cat all_hosts`; do
  ssh-copy-id -i ~/.ssh/id_rsa.pub $hostname
done
```

1. 下載檔案

``` bash
# 建立放置資料夾
sudo mkdir /usr/local/bigdata
sudo chown -R tester /usr/local/bigdata
# 下載並安裝java
curl -v -j -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.rpm -o jdk-8u101-linux-x64.rpm
sudo yum install -y jdk-8u101-linux-x64.rpm
# 下載並部署Hadoop
curl -v -j -k -L http://apache.stu.edu.tw/hadoop/common/hadoop-2.7.3/hadoop-2.7.3.tar.gz -o hadoop-2.7.3.tar.gz
tar -zxvf hadoop-2.7.3.tar.gz
mv hadoop-2.7.3 /usr/local/bigdata/hadoop
# 下載並部署zookeeper
curl -v -j -k -L http://apache.stu.edu.tw/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz -o zookeeper-3.4.8.tar.gz
tar -zxvf zookeeper-3.4.8.tar.gz
mv zookeeper-3.4.8 /usr/local/bigdata/zookeeper
# 下載並部署HBase
curl -v -j -k -L http://apache.stu.edu.tw/hbase/stable/hbase-1.2.2-bin.tar.gz -o hbase-1.2.2-bin.tar.gz
tar -zxvf hbase-1.2.2-bin.tar.gz
mv hbase-1.2.2 /usr/local/bigdata/hbase
# 下載並部署mesos
curl -v -j -k -L http://repos.mesosphere.com/el/7/x86_64/RPMS/mesos-1.0.0-2.0.89.centos701406.x86_64.rpm -o mesos-1.0.0-2.0.89.centos701406.x86_64.rpm
sudo yum install mesos-1.0.0-2.0.89.centos701406.x86_64.rpm
# 下載並部署scala
curl -v -j -k -L http://downloads.lightbend.com/scala/2.11.8/scala-2.11.8.tgz -o scala-2.11.8.tgz
tar -zxvf scala-2.11.8.tgz
mv scala-2.11.8 /usr/local/bigdata/scala
# 下載並部署spark
curl -v -j -k -L http://apache.stu.edu.tw/spark/spark-2.0.0/spark-2.0.0-bin-hadoop2.7.tgz -o spark-2.0.0-bin-hadoop2.7.tgz
tar -zxvf spark-2.0.0-bin-hadoop2.7.tgz
mv spark-2.0.0-bin-hadoop2.7 /usr/local/bigdata/spark
```

2. 環境變數設置：

``` bash
sudo tee -a /etc/bashrc << "EOF"
# JAVA
export JAVA_HOME=/usr/java/jdk1.8.0_101
# HADOOP
export HADOOP_HOME=/usr/local/bigdata/hadoop
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"
# ZOOKEEPER
export ZOOKEEPER_HOME=/usr/local/bigdata/zookeeper
# HBASE
export HBASE_HOME=/usr/local/bigdata/hbase
export HBASE_MANAGES_ZK=false
export HBASE_CLASSPATH=$HBASE_CLASSPATH:$HADOOP_CONF_DIR
export HBASE_CONF_DIR=$HBASE_HOME/conf
# SCALA
export SCALA_HOME=/usr/local/bigdata/scala
# SPARK
export SPARK_HOME=/usr/local/bigdata/spark
# PATH
export PATH=$PATH:$JAVA_HOME:$ZOOKEEPER_HOME/bin:$HADOOP_HOME/bin:$HBASE_HOME/bin:$SCALA_HOME/bin:$SPARK_HOME/bin
EOF
source /etc/bashrc
```

3. 開始設定

a. Zookeeper, Mesos設定

``` bash
## Zookeeper
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

# 佈置到其他台
scp -r /usr/local/bigdata/zookeeper tester@cassSpark2:/usr/local/bigdata
scp -r /usr/local/bigdata/zookeeper tester@cassSpark3:/usr/local/bigdata
ssh tester@cassSpark2 "sed -i -e 's/1/2/g' $ZOOKEEPER_HOME/data/myid"
ssh tester@cassSpark2 "sed -i -e 's/1/3/g' $ZOOKEEPER_HOME/data/myid"

## Mesos
# 修改zookeeper
sudo tee /etc/mesos/zk << "EOF"
zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos
EOF

# 配置quorum
sudo tee /etc/mesos-master/quorum << "EOF"
2
EOF
```

b. Hadoop設定

``` bash
# slaves
tee $HADOOP_CONF_DIR/slaves << "EOF"
cassSpark1
cassSpark2
cassSpark3
EOF

# core-site.xml
sed -i -e 's/<\/configuration>//g' $HADOOP_CONF_DIR/core-site.xml
tee -a $HADOOP_CONF_DIR/core-site.xml << "EOF"
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://hc1</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/usr/local/bigdata/hadoop/tmp</value>
  </property>
  <property>
    <name>ha.zookeeper.quorum</name>
    <value>192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181</value>
  </property>
</configuration>
EOF

mkdir -p $HADOOP_HOME/tmp

# hdfs-site.xml
sed -i -e 's/<\/configuration>//g' $HADOOP_CONF_DIR/hdfs-site.xml
tee -a $HADOOP_CONF_DIR/hdfs-site.xml << "EOF"
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.permissions</name>
    <value>false</value>
  </property>
  <property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.name.dir</name>
    <value>file:///usr/local/bigdata/hadoop/tmp/name</value>
  </property>
  <property>
    <name>dfs.data.dir</name>
    <value>file:///usr/local/bigdata/hadoop/tmp/data</value>
  </property>
  <property>
    <name>dfs.namenode.checkpoint.dir</name>
    <value>file:///usr/local/bigdata/hadoop/tmp/name/chkpt</value>
  </property>
  <property>
    <name>dfs.nameservices</name>
    <value>hc1</value>     
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>    
    <value>qjournal://192.168.0.121:8485;192.168.0.122:8485;192.168.0.123:8485/hc1</value>
  </property>
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/usr/local/bigdata/hadoop/tmp/journal</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.hc1</name>
    <value>nn1,nn2</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.hc1.nn1</name>
    <value>192.168.0.121:9000</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.hc1.nn2</name>
    <value>192.168.0.122:9000</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.hc1.nn1</name>
    <value>192.168.0.121:50070</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.hc1.nn2</name>
    <value>192.168.0.122:50070</value>
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>    
    <value>file:///usr/local/bigdata/hadoop/tmp/ha-name-dir-shared</value>
  </property>
  <property>
    <name>dfs.client.failover.proxy.provider.hc1</name> 
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence(tester)</value>
    <-- 如果不能用root權限登入ssh，記得加上能登入ssh的username -->
  </property>
  <property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/home/tester/.ssh/id_rsa</value>
  </property>
  <property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>
</configuration>
EOF

mkdir -p $HADOOP_HOME/tmp/data
mkdir -p $HADOOP_HOME/tmp/name
mkdir -p $HADOOP_HOME/tmp/journal
mkdir -p $HADOOP_HOME/tmp/name/chkpt
mkdir -p $HADOOP_HOME/tmp/ha-name-dir-shared

# mapred-site.xml
cp $HADOOP_CONF_DIR/mapred-site.xml.template $HADOOP_CONF_DIR/mapred-site.xml
sed -i -e 's/<\/configuration>//g' $HADOOP_CONF_DIR/mapred-site.xml
tee -a $HADOOP_CONF_DIR/mapred-site.xml << "EOF"
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
EOF

# yarn-site.xml
sed -i -e 's/<\/configuration>//g' $HADOOP_CONF_DIR/yarn-site.xml
tee -a $HADOOP_CONF_DIR/yarn-site.xml << "EOF"
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181</value>
  </property>
  <property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>yarn-ha</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>cassSpark1</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>cassSpark2</value>
  </property>
  <property>
    <name>yarn.resourcemanager.recovery.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.client.failover-proxy-provider</name>
    <value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value>
  </property>
  <property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
  </property>
</configuration>
EOF

# 複製到各台
scp -r /usr/local/bigdata/hadoop tester@cassSpark2:/usr/local/bigdata
scp -r /usr/local/bigdata/hadoop tester@cassSpark3:/usr/local/bigdata
```

c. HBase設定

``` bash 
sed -i -e 's/<\/configuration>//g' $HBASE_HOME/conf/hbase-site.xml
tee -a $HBASE_HOME/conf/hbase-site.xml << "EOF"
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://hc1/hbase</value>
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
    <value>192.168.0.121,192.168.0.122,192.168.0.123</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>file:///usr/local/bigdata/zookeeper/data</value>
  </property>
  <property>
    <name>hbase.master.port</name>
    <value>60000</value>
  </property>
</configuration>
EOF

# 複製slaves
cp $HADOOP_CONF_DIR/slaves $HBASE_HOME/conf/regionservers

# 複製到各台
scp -r /usr/local/bigdata/hbase tester@cassSpark2:/usr/local/bigdata
scp -r /usr/local/bigdata/hbase tester@cassSpark3:/usr/local/bigdata
```

4. 啟動

``` bash
# 啟動zookeeper server (設定自動啟動可以跳過)
zkServer.sh start
ssh tester@cassSpark2 "zkServer.sh start"
ssh tester@cassSpark3 "zkServer.sh start"
# 格式化zkfc
hdfs zkfc -formatZK
ssh tester@cassSpark2 "hdfs zkfc -formatZK"
# 開啟journalnode
$HADOOP_HOME/sbin/hadoop-daemon.sh start journalnode
ssh tester@cassSpark2 "$HADOOP_HOME/sbin/hadoop-daemon.sh start journalnode"
ssh tester@cassSpark3 "$HADOOP_HOME/sbin/hadoop-daemon.sh start journalnode"
# 格式化namenode，並啟動namenode (nn1)
hadoop namenode -format hc1
$HADOOP_HOME/sbin/hadoop-daemon.sh start namenode
# 備用namenode啟動 (nn2)
ssh tester@cassSpark2 "hadoop namenode -format hc1"
ssh tester@cassSpark2 "hdfs namenode -bootstrapStandby"
ssh tester@cassSpark2 "$HADOOP_HOME/sbin/hadoop-daemon.sh start namenode"
# 啟動zkfc
$HADOOP_HOME/sbin/hadoop-daemon.sh start zkfc
ssh tester@cassSpark2 "$HADOOP_HOME/sbin/hadoop-daemon.sh start zkfc"
# 啟動datanode
$HADOOP_HOME/sbin/hadoop-daemon.sh start datanode
ssh tester@cassSpark2 "$HADOOP_HOME/sbin/hadoop-daemon.sh start datanode"
ssh tester@cassSpark3 "$HADOOP_HOME/sbin/hadoop-daemon.sh start datanode"
# 啟動yarn
$HADOOP_HOME/sbin/yarn-daemon.sh start resourcemanager
$HADOOP_HOME/sbin/yarn-daemon.sh start nodemanager
ssh tester@cassSpark2 "$HADOOP_HOME/sbin/yarn-daemon.sh start resourcemanager"
ssh tester@cassSpark2 "$HADOOP_HOME/sbin/yarn-daemon.sh start nodemanager"
ssh tester@cassSpark3 "$HADOOP_HOME/sbin/yarn-daemon.sh start nodemanager"
# 啟動HBase
hbase-daemon.sh start master
hbase-daemon.sh start regionserver
ssh tester@cassSpark2 "hbase-daemon.sh start master"
ssh tester@cassSpark2 "hbase-daemon.sh start regionserver"
ssh tester@cassSpark3 "hbase-daemon.sh start master"
ssh tester@cassSpark3 "hbase-daemon.sh start regionserver"
```

開啟之後就可以用jps去看各台開啟狀況，如果確定都沒問題之後

接下來就可以往下去設定自動啟動的部分了，也可以先往下跳去測試部分


這裡採用python的supervisord去協助監控service的進程，並做自動啟動的動作

先安裝supervisor:

``` bash
sudo yum install python-setuptools
sudo easy_install pip
sudo pip install supervisor
# echo default config
sudo mkdir /etc/supervisor
sudo bash -c 'echo_supervisord_conf > /etc/supervisor/supervisord.conf'
```

使用`sudo vi /etc/supervisor/supervisord.conf`編輯，更動下面的設定：

```
[inet_http_server]         ; inet (TCP) server disabled by default
port=192.168.0.121:10088   ; (ip_address:port specifier, *:port for all iface)
username=tester            ; (default is no username (open server))
password=qscf12356         ; (default is no password (open server))

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
serverurl=http://192.168.0.121:10088 ; use an http:// url to specify an inet socket
username=tester              ; should be same as http_username if set
password=qscf12356          ; should be same as http_password if set

[supervisord]
environment=
  JAVA_HOME=/usr/java/jdk1.8.0_101,
  SCALA_HOME=/usr/local/scala/scala-2.11,
  SPARK_HOME=/usr/local/bigdata/spark,
  CASSANDRA_HOME=/usr/local/bigdata/cassandra,
  ZOOKEEPER_HOME=/usr/local/bigdata/zookeeper,
  HADOOP_HOME=/usr/local/bigdata/hadoop,
  HADOOP_COMMON_HOME=/usr/local/bigdata/hadoop,
  HADOOP_CONF_DIR=/usr/local/bigdata/hadoop/etc/hadoop,
  HADOOP_COMMON_LIB_NATIVE_DIR=/usr/local/bigdata/hadoop/lib/native,
  HADOOP_OPTS="-Djava.library.path=/usr/local/bigdata/hadoop/lib/native",
  HBASE_HOME=/usr/local/bigdata/hbase,
  HBASE_MANAGES_ZK="false",
  HBASE_CLASSPATH=/usr/local/bigdata/hadoop/etc/hadoop,
  HBASE_CONF_DIR=/usr/local/bigdata/hbase/conf
```

然後用下面的指令在設定後面追加下面的設定

```
sudo tee -a /etc/supervisor/supervisord.conf << "EOF"

; 全部都要配置的服務
[program:hadoop-hdfs-journalnode]
command=/bin/bash -c "$HADOOP_HOME/bin/hdfs journalnode"
stdout_logfile=/var/log/supervisor/hadoop-hdfs-journalnode-stdout.out
stderr_logfile=/var/log/supervisor/hadoop-hdfs-journalnode-stderr.out
autostart=true
startsecs=5
priority=60

[program:mesos-slave]
command=/bin/bash -c "/usr/bin/mesos-init-wrapper slave"
stdout_logfile=/var/log/supervisor/mesos-slave-stdout.out
stderr_logfile=/var/log/supervisor/mesos-slave-stderr.out
autostart=true
startsecs=5
priority=80

[program:hadoop-hdfs-datanode]
command=/bin/bash -c "$HADOOP_HOME/bin/hdfs datanode"
stdout_logfile=/var/log/supervisor/hadoop-hdfs-datanode-stdout.out
stderr_logfile=/var/log/supervisor/hadoop-hdfs-datanode-stderr.out
autostart=true
startsecs=5
priority=80

[program:hadoop-yarn-nodemanager]
command=/bin/bash -c "$HADOOP_HOME/bin/yarn nodemanager"
stdout_logfile=/var/log/supervisor/hadoop-yarn-nodemanager-stdout.out
stderr_logfile=/var/log/supervisor/hadoop-yarn-nodemanager-stderr.out
autostart=true
startsecs=5
priority=85

[program:spark-mesos-shuffle]
command=/bin/bash -c "$SPARK_HOME/bin/spark-submit --class org.apache.spark.deploy.mesos.MesosExternalShuffleService 1"
stdout_logfile=/var/log/supervisor/spark-mesos-shuffle-stdout.out
stderr_logfile=/var/log/supervisor/spark-mesos-shuffle-stderr.out
autostart=true
startsecs=5
priority=90

[program:hbase-regionserver]
command=/bin/bash -c "$HBASE_HOME/bin/hbase regionserver start"
stdout_logfile=/var/log/supervisor/hbase-regionserver-stdout.out
stderr_logfile=/var/log/supervisor/hbase-regionserver-stderr.out
autostart=true
startsecs=5
priority=95


; 下面只有需要的node才要配置
; zookeeper只需要配置在三台有zookeeper的電腦上
; namenode, zkfc跟hadoop-yarn-resourcemanager只需要配置在要hdfs, yarn做HA的那兩台上面
; mesos-master跟hbase-master只需要配置在有zookeeper的電腦上
[program:zookeeper]
command=/bin/bash -c "$ZOOKEEPER_HOME/bin/zkServer.sh start-foreground"
stdout_logfile=/var/log/supervisor/zookeeper-stdout.out
stderr_logfile=/var/log/supervisor/zookeeper-stderr.out
autostart=true
startsecs=10
priority=50

[program:mesos-master]
command=/bin/bash -c "/usr/bin/mesos-init-wrapper master"
stdout_logfile=/var/log/supervisor/mesos-master-stdout.out
stderr_logfile=/var/log/supervisor/mesos-master-stderr.out
autostart=true
startsecs=5
priority=60

[program:hadoop-hdfs-namenode]
command=/bin/bash -c "$HADOOP_HOME/bin/hdfs namenode"
stdout_logfile=/var/log/supervisor/hadoop-hdfs-namenode-stdout.out
stderr_logfile=/var/log/supervisor/hadoop-hdfs-namenode-stderr.out
autostart=true
startsecs=5
priority=70

[program:hadoop-hdfs-zkfc]
command=/bin/bash -c "$HADOOP_HOME/bin/hdfs zkfc"
stdout_logfile=/var/log/supervisor/hadoop-hdfs-zkfc-stdout.out
stderr_logfile=/var/log/supervisor/hadoop-hdfs-zkfc-stderr.out
autostart=true
startsecs=5
priority=75

[program:hadoop-yarn-resourcemanager]
command=/bin/bash -c "$HADOOP_HOME/bin/yarn resourcemanager"
stdout_logfile=/var/log/supervisor/hadoop-yarn-resourcemanager-stdout.out
stderr_logfile=/var/log/supervisor/hadoop-yarn-resourcemanager-stderr.out
autostart=true
startsecs=5
priority=80

[program:hbase-master]
command=/bin/bash -c "$HBASE_HOME/bin/hbase master start"
stdout_logfile=/var/log/supervisor/hbase-master-stdout.out
stderr_logfile=/var/log/supervisor/hbase-master-stderr.out
autostart=true
startsecs=5
priority=90
EOF

sudo systemctl restart supervisor.service
```

5. 測試

用網頁連到cassSpark1:50070跟cassSpark2:50070

cassSpark1:50070應該會顯示是active node (或是用 `hdfs haadmin -getServiceState nn1`)

cassSpark2:50070則會顯示是standby node (或是用 `hdfs haadmin -getServiceState nn2`)

(根據啟動順序不同，active的node不一定就是cassSpark1)

在active node上，輸入`sudo service hadoop-hdfs-namenode stop`停掉Hadoop namenode試試看

(如果安裝了自動啟動，就直接在supervisor的web UI直接操作即可)

等一下下，就可以看到cassSpark2:50070會變成active node，這樣hadoop的HA就完成了

至於YARN的HA就連到8081去看就好(或是用 `yarn rmadmin -getServiceState rm1`, `yarn rmadmin -getServiceState rm2`)

HBase則是16010，用一樣方式都可以測試到HA是否有成功

至於zookeeper, hadoop其他測試就看我之前發的那篇文章即可[點這](http://chingchuan-chen.github.io/hadoop/2016/07/23/deployment-spark-phoenix-hbase-yarn-zookeeper-hadoop.html)就好

備註：版本記得改成這裡的2.7.3即可，一定要測試，不然後面出問題很難抓

而且記得要重開，看看是否全部服務都如同預期一樣啟動了

Reference:

1. http://debugo.com/yarn-rm-ha/
2. http://www.cnblogs.com/junrong624/p/3580477.html
3. http://www.cnblogs.com/captainlucky/p/4710642.html
4. https://phoenix.apache.org/server.html

