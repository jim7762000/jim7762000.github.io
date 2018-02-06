---
layout: post
title: Installation of hbase and hive in mint 17
---

The second post for series of building hadoop environment in mint 17. HBase support the storage of big table in HDFS. Hive support hadoop to query data with sql commands.

Based on hadoop, we install the hbase.

1. Get hbase

``` bash
wget http://apache.stu.edu.tw/hbase/stable/hbase-1.0.0-bin.tar.gz
tar zxvf hbase-1.0.0-bin.tar.gz
sudo mv hbase-1.0.0 /usr/local/hbase
cd /usr/local/hbase
```

Put the following into the file by command: `subl conf/hbase-env.sh`.


``` bash
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export HBASE_HOME=/usr/local/hbase
export HADOOP_INSTALL=/usr/local/hadoop
export HBASE_CLASSPATH=/usr/local/hbase/conf
export HBASE_MANAGES_ZK=true
```

Set up the hbase by command: `subl conf/hbase-site.xml`

``` bash
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://localhost:54310/hbase</value>
    <!-- <value>hdfs://master:9000/hbase</value> for the multi-node case -->
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
    <value>localhost</value>
    <!-- <value>master</value> for the multi-node case -->
  </property>
  <property>
      <name>hbase.zookeeper.property.clientPort</name>
      <value>2181</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/usr/local/hadoop/tmp/data</value>
  </property>
</configuration>
```

Copy the setting of hdfs into hbase folder by command:
`cp /usr/local/hadoop/etc/hadoop/hdfs-site.xml /usr/local/hbase/conf`

2. start hbase

``` bash
sudo subl /etc/bash.bashrc
# add following 2 lines into file
# export HBASE_INSTALL=/usr/local/hbase
# export PATH=$PATH:$HBASE_INSTALL/bin
source /etc/bash.bashrc
# in ubuntu, is >> etc/bash.bashrc
```

``` bash
start-dfs.sh && start-yarn.sh # or start-all.sh
start-hbase.sh
```

3. test hbase
type `hbase shell` in the terminal, then it will show:

``` bash
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 1.0.0, r6c98bff7b719efdb16f71606f3b7d8229445eb81, Sat Feb 14 19:49:22 PST 2015

hbase(main):001:0>
```

type `list` and it return
``` bash
TABLE
0 row(s) in 4.4510 seconds
```

The other tests remain in the rhbase. Stop the server:

``` bash
stop-hbase.sh
stop-dfs.sh && stop-yarn.sh # or stop-all.sh
```

## hive installation

``` bash
wget http://apache.stu.edu.tw/hive/stable/apache-hive-1.1.0-bin.tar.gz
tar zxvf apache-hive-1.1.0-bin.tar.gz
sudo mv apache-hive-1.1.0-bin /usr/local/hive
sudo subl /etc/bash.bashrc
# add following 7 lines to file
# export HIVE_HOME="/usr/local/hive"
# export PATH=$PATH:$HIVE_HOME/bin
# export HADOOP_USER_CLASSPATH_FIRST=true
source /etc/bash.bashrc

# install mysql
sudo apt-get install mysql-server
# it need to set up a password to access mysql server
mysql -u root -p # enter the password you just set
# create user
mysql > create user 'celest'@'%' identified by 'celest';
mysql > grant all privileges on *.* to 'celest'@'%' with grant option;
mysql > flush privileges;
mysql > exit

cd /usr/local/hive
cp conf/hive-env.sh.template conf/hive-env.sh
subl conf/hive-env.sh
# add this line to file
# export HADOOP_HOME=/usr/local/hadoop
cp conf/hive-default.xml.template conf/hive-site.xml
subl hive-site.xml
# modify file like below
# <property>
#   <name>hive.metastore.warehouse.dir</name>
#   <value>/user/celest/warehouse</value>
#   <description>location of default database for the warehouse </ description>
# </property>
# <property>
#   <name>javax.jdo.option.ConnectionURL</name>
#   <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true</value>
# </property>
# <property>
#   <name>javax.jdo.option.ConnectionDriverName</name>
#   <value>com.mysql.jdbc.Driver</value>
# </property>
# <property>
#   <name>javax.jdo.option.ConnectionUserName</name>
#   <value>celest</value>
# </property>
# <property>
#   <name>javax.jdo.option.ConnectionPassword</name>
#   <value>celest</value>
# </property>
# <property>
#   <name>hive.querylog.location</name>
#   <value>/usr/local/hive/log</value>
#   <description>
#     Location of Hive run time structured log file
#   </description>
# </property>
# <property>
#   <name>hive.server2.logging.operation.log.location</name>
#   <value>/usr/local/hive/log/operation_logs</#alue>
#   <description>Top level directory where operation logs are stored if logging functionality is enabled</description>
# </property>
# <property>
#   <name>hive.exec.local.scratchdir</name>
#   <value>/usr/local/hive/tmpdir/tmp</value>
#   <description>Local scratch space for Hive jobs</description>
# </property>
# <property>
#   <name>hive.downloaded.resources.dir</name>
#   <value>/usr/local/hive/tmpdir/tmp_resources</value>
#   <description>Temporary local directory for added resources in the remote file system.</description>
# </property>
cd ~/Downloads
wget http://ftp.kaist.ac.kr/mysql/Downloads/Connector-J/mysql-connector-java-5.1.35.tar.gz
tar zxvf mysql-connector-java-5.1.35.tar.gz
cp -R mysql-connector-java-5.1.35//home/celest/Downloads/mysql-connector-java-5.1.35/mysql-connector-java-5.1.35-bin.jar /usr/local/hive/lib/

## test hive
mkdir /usr/local/hive/tmpdir
mkdir /usr/local/hive/tmpdir/logs
mkdir /usr/local/hive/tmpdir/operation_logs
mkdir /usr/local/hive/tmpdir/tmp
mkdir /usr/local/hive/tmpdir/tmp_resources
hdfs dfs -mkdir /user/celest/warehouse
hive
hive > show tables;
hive > exit;

mysql -u celest -p
mysql > show databases;
mysql > use hive
mysql > show tables;
mysql > exit
# There shows that the hive had connected to mysql server.
```
