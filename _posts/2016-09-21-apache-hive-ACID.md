---
layout: post
title: "Apache Hive and ACID"
---

Hive支援ACID，可以讓資料庫做transactions

其具備以下四種性質

1. 原子性(atomicity)：做就要做完，不做就全部都不做，不會做一半
2. 一致性(consistency)：資料庫的資訊是完整的
3. 隔離性(isolation)：可以同時讀寫多個transactions，讓讀寫不會互相影響
4. 持久性(durability)：資料的修改是永久的，不會丟失

那麼Hive支援這個有什麼好處？讓Hive能夠如同RMDB去做資料的UPDATE, DELETE

重新下載一個新的Hive做部署，MySQL那裏則跳過

``` bash
# 下載hive並部署
curl -v -j -k -L http://apache.stu.edu.tw/hive/stable/apache-hive-1.2.1-bin.tar.gz -o apache-hive-1.2.1-bin.tar.gz
tar zxvf apache-hive-1.2.1-bin.tar.gz
mv apache-hive-1.2.1-bin /usr/local/bigdata/hive
# 增加path
sudo tee -a /etc/bashrc << "EOF"
export HIVE_HOME=/usr/local/bigdata/hive
export PATH=$PATH:$HIVE_HOME/bin
EOF
source /etc/bashrc
# 解壓縮mysql jdbc connector
tar zxvf mysql-connector-java-5.1.39.tar.gz
cp mysql-connector-java-5.1.39/mysql-connector-java-5.1.39-bin.jar $HIVE_HOME/lib
# 複製設定
cp $HIVE_HOME/conf/hive-default.xml.template $HIVE_HOME/conf/hive-site.xml
# 新增hive所需的目錄
sudo mkdir /tmp/tester
sudo mkdir /tmp/tester/hive_resource
sudo chown -R tester.tester /tmp/tester
# 初始化hive的schema
schematool -initSchema -dbType mysql
```

用`vi $HIVE_HOME/conf/hive-site.xml`去配置設定

``` XML
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://192.168.0.121:3306/hive?autoReconnect=true&amp;useSSL=true&amp;createDatabaseIfNotExist=true&amp;characterEncoding=utf8</value>
    <description>JDBC connection string used by Hive Metastore</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>JDBC Driver class</description>
  </property>
  <property>
      <name>javax.jdo.option.ConnectionUserName</name>
      <value>hive</value>
      <description>Metastore database user name</description>
  </property>  
  <property>
      <name>javax.jdo.option.ConnectionPassword</name>
      <value>Qscf123%^</value>
      <description>Metastore database password</description>
  </property>
  <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/tmp/tester</value>
    <description>Local scratch space for Hive jobs</description>
  </property>
  <property>
    <name>hive.downloaded.resources.dir</name>
    <value>/tmp/tester/hive_resource</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
  <property>
    <name>hive.compactor.initiator.on</name>
    <value>true</value>
    <description>
      Whether to run the initiator and cleaner threads on this metastore instance or not.
      Set this to true on one instance of the Thrift metastore service as part of turning
      on Hive transactions. For a complete list of parameters required for turning on
      transactions, see hive.txn.manager.
    </description>
  </property>
  <property>
    <name>hive.compactor.worker.threads</name>
    <value>2</value>
    <description>
      How many compactor worker threads to run on this metastore instance. Set this to a
      positive number on one or more instances of the Thrift metastore service as part of
      turning on Hive transactions. For a complete list of parameters required for turning
      on transactions, see hive.txn.manager.
      Worker threads spawn MapReduce jobs to do compactions. They do not do the compactions
      themselves. Increasing the number of worker threads will decrease the time it takes
      tables or partitions to be compacted once they are determined to need compaction.
      It will also increase the background load on the Hadoop cluster as more MapReduce jobs
      will be running in the background.
    </description>
  </property>
  <property>
    <name>hive.support.concurrency</name>
    <value>true</value>
    <description>
      Whether Hive supports concurrency control or not.
      A ZooKeeper instance must be up and running when using zookeeper Hive lock manager
    </description>
  </property>
  <property>
    <name>hive.enforce.bucketing</name>
    <value>true</value>
    <description>Whether bucketing is enforced. If true, while inserting into the table, bucketing is enforced.</description>
  </property>
 <property>
    <name>hive.exec.dynamic.partition</name>
    <value>true</value>
    <description>Whether or not to allow dynamic partitions in DML/DDL.</description>
  </property>
  <property>
    <name>hive.exec.dynamic.partition.mode</name>
    <value>nonstrict</value>
    <description>
      In strict mode, the user must specify at least one static partition
      in case the user accidentally overwrites all partitions.
      In nonstrict mode all partitions are allowed to be dynamic.
    </description>
  </property>
  <property>
    <name>hive.txn.manager</name>
    <value>org.apache.hadoop.hive.ql.lockmgr.DummyTxnManager</value>
    <description>
      Set to org.apache.hadoop.hive.ql.lockmgr.DbTxnManager as part of turning on Hive
      transactions, which also requires appropriate settings for hive.compactor.initiator.on,
      hive.compactor.worker.threads, hive.support.concurrency (true), hive.enforce.bucketing
      (true), and hive.exec.dynamic.partition.mode (nonstrict).
      The default DummyTxnManager replicates pre-Hive-0.13 behavior and provides
      no transactions.
    </description>
  </property>
```

測試：

``` bash
# 新增資料
tee ~/test_df2.csv << "EOF"
id1,a,0.01991423
id2,b,0.73282957
id3,c,0.00552144
id4,d,0.83103357
id5,a,-0.79378789
id6,b,-0.36969293
id7,c,-1.66246829
id8,d,-0.73893442
EOF
```

``` SQL
-- 創一張表，先放資料
CREATE TABLE test_df2 (v1 STRING, v2 STRING, v3 DOUBLE)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
-- 讀入資料
LOAD DATA LOCAL INPATH '/home/tester/test_df2.csv'
OVERWRITE INTO TABLE test_df2;
-- 創一張transaction表
create table test_df2_transaction
(v1 STRING, v2 STRING, v3 DOUBLE) 
clustered by (v1) into 3 buckets 
stored as orc TBLPROPERTIES ('transactional'='true');
-- 把資料加入transaction表
insert into test_df2_transaction select * from test_df2;
-- 看資料
select * from test_df2_transaction;
/*
id8     d       -0.73893442
id5     a       -0.79378789
id2     b       0.73282957
id6     b       -0.36969293
id3     c       0.00552144
id7     c       -1.66246829
id4     d       0.83103357
id1     a       0.01991423
*/
-- 試試看update
update test_df2_transaction set v2='c' where v1='id1';
select * from test_df2_transaction;
/*
id8     d       -0.73893442
id5     a       -0.79378789
id2     b       0.73282957
id11    a       2.001
id6     b       -0.36969293
id3     c       0.00552144
id9     b       2.001
id7     c       -1.66246829
id4     d       0.83103357
id1     c       2.001
*/
-- 試試看delete
delete from test_df2_transaction where v1='id1';
select * from test_df2_transaction;
/*
id8     d       -0.73893442
id5     a       -0.79378789
id2     b       0.73282957
id6     b       -0.36969293
id3     c       0.00552144
id7     c       -1.66246829
id4     d       0.83103357
*/
-- 試試看insert
insert into table test_df2_transaction values ('id9', 'b', 2.001),('id1', 'b', 2.001),('id11', 'a', 2.001);
select * from test_df2_transaction;
/*
id8     d       -0.73893442
id5     a       -0.79378789
id2     b       0.73282957
id11    a       2.001
id6     b       -0.36969293
id3     c       0.00552144
id9     b       2.001
id7     c       -1.66246829
id4     d       0.83103357
id1     b       2.001
*/
```

接下來用spark去測試

用`spark-shell mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos`開啟

``` scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder().appName("spark on hive").
  config("spark.sql.warehouse.dir", "hdfs://hc1/spark").
  enableHiveSupport().getOrCreate()

spark.sql("update test_df2_transaction set v2='c' where v1='id1'").show()
# org.apache.spark.sql.catalyst.parser.ParseException
spark.sql("delete from test_df2_transaction where v1='id1'").show()
# org.apache.spark.sql.catalyst.parser.ParseException  
```

Spark SQL無法使用update或是delete的動作
