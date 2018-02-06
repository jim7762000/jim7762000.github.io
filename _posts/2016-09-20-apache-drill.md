---
layout: post
title: "Apache Drill"
---

SQL on Hadoop不外乎Apache Drill, Hive, Hive on Tez, Phoenix, 

Cloudera Impala (正在孵化為Apache專案), Presto,

Pivotal HAWQ, IBM BigSQL, Apache Tajo, Apache Kylin等

在這麼多選擇中，我選擇用Drill，以下闡述我的原因

Drill是一套SQL on Hadoop的解決方案

一個Schema-free的data model，這意思代表說不在受限於data model限制而無法查詢

但是他不能用index，所以在有些查詢上會比有schema的data model來的慢

儘管如此，Drill在query的表現仍然相當優秀，勝過Hive with MapReduce, Hive on Tez,

Spark SQL, Presto，跟Impala互有上下 ([reference 1](http://allegro.tech/2015/06/fast-data-hackathon.html), [reference 2](https://www.mapr.com/blog/comparing-sql-functions-and-performance-apache-spark-and-apache-drill))


不過問題來了，Drill要base什麼去建立？ hdfs? hbase? hive? 還是其他nosql的架構？

這個就depends on每個人的需求了，因為我需要一個Loader去同步在Oracle的資料

而這loader最簡單的方式就是使用Spark SQL去做(因為Drill沒辦法直接讀寫hive, hbase等資料庫)

所以這裡我採用hive當做表格儲存的位置，能讓Spark SQL直接做table的存寫

而且Drill直接搜尋hive的速度相當快，不需要透過hive下的engine(mapredure, tez or spark)去處理


我原本傾向使用有schema的hbase跟hive，可以直接透過Spark SQL去存讀資料

但是又想到我需要一個Loader去同步在Oracle的資料，不過Drill無法提供直接使用DataFrame存寫的功能

做了一點功課之後，決定直接使用HDFS存JSON檔案

在Spark SQL可以透過JDBC接口直接使用Spark SQL去處理資料再存新的json回去


1. 下載檔案並部署

``` bash
curl -v -j -k -L http://apache.stu.edu.tw/drill/drill-1.8.0/apache-drill-1.8.0.tar.gz -o apache-drill-1.8.0.tar.gz
tar -xzvf apache-drill-1.8.0.tar.gz
mv apache-drill-1.8.0 /usr/local/bigdata/drill
```

2. 配置

使用`vi /usr/local/bigdata/drill/conf/drill-override.conf`去configure Drill

``` json
drill.exec: {
  cluster-id: "drillcluster",
  zk.connect: "192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181"
}
```

如果記憶體不多的話建議下面配置：

``` bash
tee -a /usr/local/bigdata/drill/conf/drill-env.sh << "EOF"
# 這一行一定要留著
DRILL_MAX_DIRECT_MEMORY="2G"
DRILL_MAX_HEAP="1G"

export DRILL_JAVA_OPTS="-Xms1G -Xmx$DRILL_MAX_HEAP -XX:MaxDirectMemorySize=$DRILL_MAX_DIRECT_MEMORY -XX:MaxPermSize=512M -XX:ReservedCodeCacheSize=1G -ea"
EOF
```

複製到各台node

```
scp -r /usr/local/bigdata/drill tester@cassSpark2:/usr/local/bigdata
scp -r /usr/local/bigdata/drill tester@cassSpark3:/usr/local/bigdata
```

使用`/usr/local/bigdata/drill/bin/drillbit.sh start`去啟動Drill

就可以用`http://192.168.0.121:8047/`連到Drill的web UI了

可以再Storage Page看到dfs的Option，按下Update即可更新(更新下面的值即可)：

``` json
"connection": "hdfs://hc1",
"workspaces": {
  "root": {
    "location": "/drill",
    "writable": true,
    "defaultInputFormat": null
  },
  "tmp": {
    "location": "/drill/tmp",
    "writable": true,
    "defaultInputFormat": null
  }
}
```

按下Update，然後Close


再來是上傳一些資料吧

``` bash
tee ~/test_df.json << "EOF"
[{"v1":"b","v2":"d","v3":0.01991423},{"v1":"b","v2":"e","v3":0.73282957},{"v1":"b","v2":"c","v3":0.00552144},
{"v1":"b","v2":"d","v3":0.83103357},{"v1":"a","v2":"d","v3":-0.79378789},{"v1":"a","v2":"e","v3":-0.36969293},
{"v1":"a","v2":"c","v3":-1.66246829},{"v1":"b","v2":"c","v3":-0.73893442},{"v1":"a","v2":"c","v3":-0.49832169},
{"v1":"b","v2":"e","v3":-0.83277815},{"v1":"a","v2":"e","v3":-0.10230033},{"v1":"a","v2":"c","v3":0.14617246},
{"v1":"a","v2":"d","v3":-0.50122879},{"v1":"a","v2":"d","v3":-1.01900482},{"v1":"b","v2":"e","v3":-0.08073591},
{"v1":"a","v2":"d","v3":-1.02780559},{"v1":"b","v2":"d","v3":-1.28585578},{"v1":"a","v2":"e","v3":2.19115715},
{"v1":"b","v2":"c","v3":0.22095974},{"v1":"a","v2":"d","v3":0.98055576},{"v1":"b","v2":"d","v3":0.92121262},
{"v1":"a","v2":"c","v3":0.43742128},{"v1":"a","v2":"d","v3":0.1683678},{"v1":"a","v2":"e","v3":-1.31987619},
{"v1":"b","v2":"c","v3":1.0675091},{"v1":"b","v2":"e","v3":0.49024668},{"v1":"a","v2":"e","v3":-1.65978632},
{"v1":"b","v2":"c","v3":-0.1294416},{"v1":"b","v2":"c","v3":1.00880932},{"v1":"a","v2":"d","v3":0.27295147},
{"v1":"b","v2":"d","v3":-0.1935518},{"v1":"a","v2":"d","v3":0.92765404},{"v1":"b","v2":"d","v3":-0.49652849},
{"v1":"b","v2":"c","v3":0.11603917},{"v1":"a","v2":"d","v3":1.00496088},{"v1":"a","v2":"e","v3":0.5742589},
{"v1":"a","v2":"c","v3":-0.07431593},{"v1":"a","v2":"e","v3":1.91539019},{"v1":"a","v2":"c","v3":0.07681478},
{"v1":"b","v2":"c","v3":0.69678599}]
EOF

tee ~/test_widecol.json << "EOF"
[{"v1":"b","v2":"d","v3":[0.01991423,0.83103357,-1.28585578,0.92121262,-0.1935518,-0.49652849]},
{"v1":"b","v2":"e","v3":[0.73282957,-0.83277815,-0.08073591,0.49024668]},
{"v1":"b","v2":"c","v3":[0.00552144,-0.73893442,0.22095974,1.0675091,-0.1294416,1.00880932,0.11603917,0.69678599]},
{"v1":"a","v2":"d","v3":[-0.79378789,-0.50122879,-1.01900482,-1.02780559,0.98055576,0.1683678,0.27295147,0.92765404,1.00496088]},
{"v1":"a","v2":"e","v3":[-0.36969293,-0.10230033,2.19115715,-1.31987619,-1.65978632,0.5742589,1.91539019]},
{"v1":"a","v2":"c","v3":[-1.66246829,-0.49832169,0.14617246,0.43742128,-0.07431593,0.07681478]}]
EOF
```

上傳到hdfs去：

``` bash
hdfs dfs -mkdir /drill
hdfs dfs -mkdir /drill/tmp
hdfs dfs -put ~/test_df.json /drill
hdfs dfs -put ~/test_widecol.json /drill
# 確定資料有上去
hdfs dfs -ls /drill
# Found 3 items
# -rw-r--r--   3 tester supergroup       1456 2016-09-19 23:49 /drill/test_df.json
# -rw-r--r--   3 tester supergroup        618 2016-09-19 23:49 /drill/test_widecol.json
# drwxr-xr-x   - tester supergroup          0 2016-09-19 23:46 /drill/tmp
```

執行下面的SQL可以得到下圖的結果

``` SQL
SELECT * FROM dfs.`/drill/test_df.json`;
```

![](/images/drill_test_df.png)

執行下面的SQL可以得到下圖的結果

``` SQL
SELECT * FROM dfs.`/drill/test_widecol.json`;
```

![](/images/drill_test_widecol.png)


使用[SQuirreL SQL Client](http://squirrel-sql.sourceforge.net/)透過JDBC去操作Drill

先點擊左邊的Driver按鈕，然後按下左上角藍色的+去新增Driver

其中Example URL填`jdbc:drill:zk=cassSpark1:2181`

Extra Class Path新增Drill binary tar裡面`jars/jdbc-driver/drill-jdbc-all-1.8.0.jar`

Driver就可以選`org.apache.drill.jdbc.Driver`了，如同下圖的設定

![](/images/drill_jdbc_1.png)


接著點擊左邊的`Alias`，一樣按下藍色的+去新增，其中URL的填法如下：

```
jdbc:drill:zk=<zookeeper_quorum>/<drill_directory_in_zookeeper>/<cluster_ID>;schema=<schema_to_use_as_default>
```

像是我的是`jdbc:drill:zk=cassSpark1:2181/drill/drillcluster;schema=dfs`

`cassSpark1:2181`是我的zookeeper地址，`drillcluster`是我在conf裡面設定的cluster name

使用的預設schema就dfs (請看Drill網頁UI裡面的Storage那頁)，設定會如下圖：

![](/images/drill_jdbc_2.png)

然後就可以連線了！如下圖：

![](/images/drill_jdbc_3.png)

![](/images/drill_jdbc_4.png)


再來是Drill的REST接口

這裡分別用linux的curl指令跟R httr套件的POST去做：

![](/images/drill_rest_1.png)

![](/images/drill_rest_2.png)

``` R
library(httr)

POST("http://192.168.0.121:8047/query.json", 
     body = list(queryType = "SQL", 
     query = "SELECT v1,v2,v3 FROM dfs.`/drill/test_df.json`"),
     encode = "json")
```


另外，也可以用jdbc去連Oracle，先從Oracle官方網站下載到ojdbc7.jar

將ojdbc7.jar放到`/usr/local/bigdata/drill/lib/3party`裡面，然後重開drill (記得是每一台都要放)

(如果有用我之前Spark在Oracle的配置，可以直接下`cp $SPARK_HOME/extraClass/ojdbc7.jar /usr/local/bigdata/drill/jars/3rdparty/`)

然後在web UI的Storage增加一個New Storage Plugin，叫做oracle：

``` json
{
  type: "jdbc",
  enabled: true,
  driver: "oracle.jdbc.OracleDriver",
  url:"jdbc:oracle:thin:system/qscf12356@192.168.0.120:1521/orcl"
}
```

就可以使用`select * from oracle.<user_name>.<table_name>`去做查詢了


最後附上supervisor的config

```
sudo tee -a /etc/supervisor/supervisord.conf << "EOF"
[program:drill]
command=/bin/bash -c "/usr/local/bigdata/drill/bin/drillbit.sh run"
stdout_logfile=/var/log/supervisor/drill-stdout.out
stderr_logfile=/var/log/supervisor/drill-stderr.out
autostart=true
startsecs=5
priority=95
EOF
sudo systemctl restart supervisor
```
