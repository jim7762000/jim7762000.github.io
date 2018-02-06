---
layout: post
title: "Spark on Mesos: dynamic resource allocation"
---

上篇部署了Spark on Mesos的環境

而這篇主要是想要使用Spark on Mesos的dynamic resource allocation跟external shuffle service

Dynamic resource allocation是為了能夠讓Spark能夠更有效的使用系統資源的系統

能夠動態的去增加worker以利application的運行，並能realease不在使用中的executor

而這個功能原本只有在Spark on Yarn的配置上才有，2.0.0的Spark也在Mesos上實現支援了

在external shuffle Service啟動下，它會把executors寫入的shuffle檔案安全的移除

這個服務的啟動是為了去讓Spark能夠使用dynamic resource allocation

同時，他也會協助Executor去分配shuffle的資料，以增加shuffle的效率，減少executor的loading


1. 準備工作

基本上就是有一組Spark On Mesos的cluster，並有Cassandra

2. 開始配置

這裡只要重新配置Spark即可

``` bash
# 移除舊的 (有的話)
rm -r $SPARK_HOME

# 下載新的binary包
curl -v -j -k -L http://d3kbcqa49mib13.cloudfront.net/spark-2.0.0-bin-hadoop2.6.tgz -o spark-2.0.0-bin-hadoop2.6.tgz
tar -zxvf spark-2.0.0-bin-hadoop2.6.tgz
mv spark-2.0.0-bin-hadoop2.6 /usr/local/bigdata/spark

# 複製檔案
cp $SPARK_HOME/conf/spark-env.sh.template $SPARK_HOME/conf/spark-env.sh
cp $SPARK_HOME/conf/spark-defaults.conf.template $SPARK_HOME/conf/spark-defaults.conf
cp $SPARK_HOME/conf/log4j.properties.template $SPARK_HOME/conf/log4j.properties

# additional jars (這兩個的取得在前幾篇都有說，有用到再放)
mkdir $SPARK_HOME/extraClass
cp spark-cassandra-connector-assembly-2.0.0-M1.jar $SPARK_HOME/extraClass/
cp ojdbc7.jar $SPARK_HOME/extraClass/

# 傳入設定
tee -a $SPARK_HOME/conf/spark-env.sh << "EOF"
SPARK_LOCAL_DIRS=/usr/local/bigdata/spark
SPARK_SCALA_VERSION=2.11
MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so
EOF

tee -a $SPARK_HOME/conf/spark-defaults.conf << "EOF"
spark.driver.extraClassPath /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar:/usr/local/bigdata/spark/extraClass/ojdbc7.jar
spark.driver.extraLibraryPath /usr/local/bigdata/spark/extraClass
spark.executor.extraClassPath /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar:/usr/local/bigdata/spark/extraClass/ojdbc7.jar
spark.executor.extraLibraryPath /usr/local/bigdata/spark/extraClass
spark.jars /usr/local/bigdata/spark/extraClass/spark-cassandra-connector-assembly-2.0.0-M1.jar,/usr/local/bigdata/spark/extraClass/ojdbc7.jar
spark.scheduler.mode fair
spark.driver.cores 1
spark.driver.memory 2g
spark.cores.max 6
spark.executor.cores 1
spark.executor.memory 2g
spark.dynamicAllocation.enabled true
spark.dynamicAllocation.initialExecutors 6
spark.executor.instances 6
spark.dynamicAllocation.executorIdleTimeout 15
spark.shuffle.service.enabled true
EOF

# 複製到其他台
ssh tester@cassSpar2 "rm -r $SPARK_HOME"
ssh tester@cassSpar3 "rm -r $SPARK_HOME"
scp -r /usr/local/bigdata/spark tester@cassSpark2:/usr/local/bigdata
scp -r /usr/local/bigdata/spark tester@cassSpark3:/usr/local/bigdata
```

設定自動啟動shuffle service

``` bash
sudo tee /etc/init.d/spark-shuffle << "EOF"
#!/bin/bash
#
# spark-shuffle
# 
# chkconfig: 2345 89 9 
# description: spark-shuffle

source /etc/rc.d/init.d/functions

# Spark install path (where you extracted the tarball)
SPARK_HOME=/usr/local/bigdata/spark

RETVAL=0
PIDFILE=/var/run/spark-shuffle.pid
desc="spark-shuffle daemon"

start() {
  echo -n $"Starting $desc (spark-shuffle): "
  daemon $SPARK_HOME/sbin/start-mesos-shuffle-service.sh
  RETVAL=$?
  echo
  [ $RETVAL -eq 0 ] && touch /var/lock/subsys/spark-shuffle
  return $RETVAL
}

stop() {
  echo -n $"Stopping $desc (spark-shuffle): "
  daemon $SPARK_HOME/sbin/stop-mesos-shuffle-service.sh
  RETVAL=$?
  sleep 5
  echo
  [ $RETVAL -eq 0 ] && rm -f /var/lock/subsys/spark-shuffle $PIDFILE
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
  [ -e /var/lock/subsys/spark-shuffle ] && restart || :
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
sudo chmod +x /etc/init.d/spark-shuffle
sudo chkconfig --add spark-shuffle
sudo service spark-shuffle start
```

試試看用下面指令submit任務

``` bash
spark-submit --master mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos --class cassSpark test_cassspark_2.11-1.0.jar
```

下方的指令可以用來找出現在的Mesos master

``` bash
mesos-resolve `cat /etc/mesos/zk`
```

執行完後會出現你配置zookeeper的其中一個IP with port 5050

例如我的Mesos master ip是192.168.0.121:5050，那麼我連上這個網址就會看到現在任務狀況

用spark-submit後，在Frameworks那個分頁會看到一個framework被啟動

接著spark會開始動態的規畫所需的executor進行MapReduce的job

運行的時候會在Mesos的主頁看到Task被起起來做運行的動作


最後是關於刪掉framework的方式，可以在unix系統上使用下方指令去刪除framework：

``` bash
# framework id可以在Framework那頁找到，按copy id就可以複製下來，然後貼到命令裡面來發出刪除的動作
curl -XPOST http://192.168.0.121:5050/master/teardown -d 'frameworkId=2444f6a3-1bfb-47d6-8b11-ab9c8f56e3c9-0000'
```
  
我這裡還是覺得為什麼mesos的webUI會沒有提供直接kill的命令感到疑惑
