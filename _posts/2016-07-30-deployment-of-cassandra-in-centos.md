---
layout: post
title: "在centos下部署cassandra"
---

這篇是我在centos部署cassandra的紀錄

1. 準備工作

基本上同Hadoop那篇，這裡不贅述

2. 部署Cassandra

``` bash
curl -v -j -k -L http://apache.stu.edu.tw/cassandra/2.2.7/apache-cassandra-2.2.7-bin.tar.gz -o apache-cassandra-2.2.7-bin.tar.gz
tar -zxvf apache-cassandra-2.2.7-bin.tar.gz
sudo mv apache-cassandra-2.2.7 /usr/local/cassandra
sudo chown -R tester /usr/local/cassandra

sudo tee -a /etc/bashrc << "EOF"
export CASSANDRA_HOME=/usr/local/cassandra
export PATH=$PATH:$CASSANDRA_HOME/bin
EOF
```

3. 修改配置

使用`vi $CASSANDRA_HOME/conf/cassandra.yaml`去改設定檔，改的部分如下：

``` yaml
# first place:
cluster_name: 'sparkSever'

# second place:
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          - seeds: "192.168.0.161,192.168.0.162,192.168.0.163,192.168.0.164"
          
# third place:
listen_address: 192.168.0.161

# fourth place:
rpc_address: 192.168.0.161

# fifth place:
endpoint_snitch: GossipingPropertyFileSnitch

# sixth place:

# seventh place:

```

一台裝完之後，可以用下面指令做複製的動作，然後修改需要設定的地方(listen_address跟rpc_address)：

``` bash
scp -rp /usr/local/cassandra tester@sparkServer1:/usr/local
scp -rp /usr/local/cassandra tester@sparkServer2:/usr/local
scp -rp /usr/local/cassandra tester@sparkServer3:/usr/local

ssh tester@sparkServer1 "sed -i -e 's/: 192.168.0.161/: 192.168.0.162/g' /usr/local/cassandra/conf/cassandra.yaml"
ssh tester@sparkServer2 "sed -i -e 's/: 192.168.0.161/: 192.168.0.163/g' /usr/local/cassandra/conf/cassandra.yaml"
ssh tester@sparkServer3 "sed -i -e 's/: 192.168.0.161/: 192.168.0.164/g' /usr/local/cassandra/conf/cassandra.yaml"
```

4. 啟動Cassandra

在sparkServer0上輸入下面的指令，就可以成功開啟四台Cassandra的node：

``` bash
ssh tester@sparkServer1 "cassandra"
ssh tester@sparkServer2 "cassandra"
ssh tester@sparkServer3 "cassandra"
cassandra
```

用`nodetool status`可以確定一下是不是都有跑起來，顯示資訊如下：

``` bash
nodetool status
# Datacenter: dc1
# ===============
# Status=Up/Down
# |/ State=Normal/Leaving/Joining/Moving
# --  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
# UN  192.168.0.161  150.93 KB  256          45.5%             5fbf33ca-a88c-4ca4-86c3-6fee0679f2ac  rack1
# UN  192.168.0.162  98.96 KB   256          49.4%             752cd742-8169-415a-b6e5-b0d60fdf3fc0  rack1
# UN  192.168.0.163  68.49 KB   256          52.8%             b6b7f4b9-9843-430f-a2a4-7ad7a56e4361  rack1
# UN  192.168.0.164  90.3 KB    256          52.2%             2ee0ec24-b542-4bd6-86d3-fc284a12bf5e  rack1
```

5. 自動啟動Cassandra

開機自動啟動Cassandra的script(用`sudo vi /etc/init.d/cassandra`去create)：

``` bash 
#!/bin/bash
# chkconfig: 2345 99 01
# description: Cassandra

. /etc/rc.d/init.d/functions

CASSANDRA_HOME=/usr/local/cassandra
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

6. 測試

打開Terminal，輸入`cqlsh 192.168.0.161` (任意一台有cassandra在運行的電腦IP)就可以開始用Cassandra的cql了，簡單測試如下：

``` SQL
-- Create KEYSPACE
CREATE KEYSPACE mariadbtest2
    WITH replication = {'class': 'SimpleStrategy', 
        'replication_factor': '3'};

-- Use the KEYSPACE
USE mariadbtest2;

-- create table
CREATE TABLE t1 (rowid text, data1 text, data2 int, PRIMARY KEY (rowid));

-- insert data
INSERT INTO t1 (rowid, data1, data2) VALUES ('rowid001', 'g1', 123456);
INSERT INTO t1 (rowid, data1, data2) VALUES ('rowid002', 'g2', 34543);
INSERT INTO t1 (rowid, data1, data2) VALUES ('rowid003', 'g1', 97548);
INSERT INTO t1 (rowid, data1, data2) VALUES ('rowid004', 'g1', 62145);
INSERT INTO t1 (rowid, data1, data2) VALUES ('rowid005', 'g2', 140578);

-- query whole data
SELECT * FROM t1;
/*
 rowid    | data1 | data2
----------+-------+--------
 rowid003 |    g1 |  97548
 rowid002 |    g2 |  34543
 rowid001 |    g1 | 123456
 rowid005 |    g2 | 140578
 rowid004 |    g1 |  62145

(5 rows)
*/

-- simple calculation
SELECT sum(data2) FROM t1;
/*
 system.sum(data2)
-------------------
            458270

(1 rows)
*/
```

現行的Cassandra CQL還沒支援使用`GROUP BY`的功能

看了一下網路討論，主要是Cassandra最原本的開發目的是為了快速讀取、儲存的地方，而非計算之用

因此，CQL這一塊還在發展，我有看到issue已經要準備在3.X更新`GROUP BY`的部分了，敬請期待

7. Reference

1. http://blog.fens.me/nosql-r-cassandra/
1. https://twgame.wordpress.com/2015/02/16/real-machine-cassandra-cluster/
1. http://www.planetcassandra.org/blog/installing-the-cassandra-spark-oss-stack/
1. http://datastax.github.io/python-driver/getting_started.html
1. https://docs.datastax.com/en/developer/python-driver/1.0/python-driver/quick_start/qsSimpleClientAddSession_t.html
1. https://mariadb.com/kb/en/mariadb/cassandra-storage-engine-use-example/
