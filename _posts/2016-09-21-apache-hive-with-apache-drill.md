---
layout: post
title: "Apache Hive with Apache Drill"
---

前一篇的Apache Drill效能方面極佳

唯一可惜的點是不能直接存寫hive, hbase

但是如果只需要用到讀取資料

不做insert, update的話，Drill無疑是最佳的方案

這篇主要介紹怎麼建立Hive，Hive的建立相當麻煩

以前建立過一次，就不想在建立它，只是沒想到還是得用它

Hive有三種建立模式：

1. localhost derby
2. localhost mysql
3. remote mysql

以前是直接走localhost mysql，這次則採用remote mysql的方式建立

這樣的好處是，全部的cluster都共用一個metastore，不需要再各台都建立mysql

mysql也可以用其他資料庫代替，如Oracle, PostgreSQL以及MS SQL Server

不過在centos中最簡單取得的就是Mysql，而我這使用的是Oracle MySQL community server

請到下面網址去下載Red Hat Enterprise Linux 7 / Oracle Linux 7 (x86, 64-bit), RPM Bundle：

http://dev.mysql.com/downloads/mysql/

並在 http://dev.mysql.com/downloads/connector/j/ 下載mysql的jdbc連線用的jar檔


1. 安裝

``` bash
# 開始部署
tar xvf mysql-5.7.15-1.el7.x86_64.rpm-bundle.tar
sudo yum remove mariadb-libs
sudo yum install -y mysql-community-client-5.7.15-1.el7.x86_64.rpm mysql-community-common-5.7.15-1.el7.x86_64.rpm mysql-community-devel-5.7.15-1.el7.x86_64.rpm mysql-community-libs-5.7.15-1.el7.x86_64.rpm mysql-community-server-5.7.15-1.el7.x86_64.rpm mysql-community-test-5.7.15-1.el7.x86_64.rpm

# 配置mysql檔案
tee -a /etc/my.cnf << "EOF"
bind-address=192.168.0.121
skip_ssl
EOF

# 啟動mysql
sudo systemctl start mysqld

# 抓取密碼
sudo grep 'temporary password' /var/log/mysqld.log # find the password
# 登入mysql
mysql -u root -p
# 修改密碼跟創新的user for hive
# mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'Qscf123%^';
# mysql> create user 'hive'@'%' identified by 'Qscf123%^';
# mysql> create database hive DEFAULT CHARACTER SET utf8;
# mysql> grant all PRIVILEGES on *.* TO 'hive'@'%' IDENTIFIED BY 'Qscf123%^' WITH GRANT OPTION;
# 重啟mysql
sudo systemctl restart mysqld

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

# 配置hive
cp $HIVE_HOME/conf/hive-default.xml.template $HIVE_HOME/conf/hive-site.xml
vi $HIVE_HOME/conf/hive-site.xml
```

hive的配置項目：

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
    <value>/tmp/tester/hive_resources</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
```

配置完hive之後，就可以開始啟動了

``` bash
# 新增hive所需的目錄
sudo mkdir /tmp/tester
sudo mkdir /tmp/tester/hive_resource
sudo chown -R tester.tester /tmp/tester
# 初始化hive的schema
schematool -initSchema -dbType mysql
# 啟動hive
hive 
# 成功進去之後就可以用exit;離開了

# 配置hive的supervisord啟動方案
sudo vi /etc/supervisor/supervisord.conf
# environmet
# HIVE_HOME=/usr/local/bigdata/hive

sudo tee -a /etc/supervisor/supervisord.conf << "EOF"
[program:hive]
command=/bin/bash -c "$HIVE_HOME/bin/hive --service metastore"
stdout_logfile=/var/log/supervisor/hive-stdout.out
stderr_logfile=/var/log/supervisor/hive-stderr.out
autostart=true
startsecs=5
priority=90
EOF
# 重啟supervisord
sudo systemctl restart supervisor

# 複製到各台，其他台都連這台的mysql即可，複製過去配置PATH跟supervisord即可使用
scp -r $HIVE_HOME tester@cassSpark2:/usr/local/bigdata
scp -r $HIVE_HOME tester@cassSpark3:/usr/local/bigdata
```

2. 測試：

``` bash 
tee ~/test_df.csv << "EOF"
b,d,0.01991423
b,e,0.73282957
b,c,0.00552144
b,d,0.83103357
a,d,-0.79378789
a,e,-0.36969293
a,c,-1.66246829
b,c,-0.73893442
a,c,-0.49832169
b,e,-0.83277815
a,e,-0.10230033
a,c,0.14617246
a,d,-0.50122879
a,d,-1.01900482
b,e,-0.08073591
a,d,-1.02780559
b,d,-1.28585578
a,e,2.19115715
b,c,0.22095974
a,d,0.98055576
b,d,0.92121262
a,c,0.43742128
a,d,0.1683678
a,e,-1.31987619
b,c,1.0675091
b,e,0.49024668
a,e,-1.65978632
b,c,-0.1294416
b,c,1.00880932
a,d,0.27295147
b,d,-0.1935518
a,d,0.92765404
b,d,-0.49652849
b,c,0.11603917
a,d,1.00496088
a,e,0.5742589
a,c,-0.07431593
a,e,1.91539019
a,c,0.07681478
b,c,0.69678599
EOF
```

``` SQL
CREATE TABLE test_df (v1 STRING, v2 STRING, v3 DOUBLE)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

LOAD DATA LOCAL INPATH '/home/tester/test_df.csv'
OVERWRITE INTO TABLE test_df;

select count(*) from test_df; # 40
select v1,v2,sum(v3) from test_df group by v1,v2; 
# a       c       -1.57469739
# a       d       0.012662860000000276
# a       e       1.22915047
# b       c       2.24724874
# b       d       -0.2037756499999998
# b       e       0.30956219000000007
```

附上成功執行畫面：

![](/images/hive_mr.PNG)

可以看到它是透過hadoop的mapreduce去做的，執行時間是23.925秒

3. 與Apache Drill共舞

先修改Storage的hive，改成下方這樣：

``` json
{
  "type": "hive",
  "enabled": true,
  "configProps": {
    "hive.metastore.uris": "thrift://192.168.0.121:9083",
    "hive.metastore.sasl.enabled": "false",
    "fs.default.name": "hdfs://hc1/"
  }
}
```

到query去執行下方的查詢：

``` SQL
select v1,v2,avg(v3) from test_df group by v1,v2; 
```

可以在Profile那裏看到執行的細節：

![](/images/hive_drill.PNG)

全部執行時間只有0.530s，整整比hive的mapreduce快上45倍


最後，也測試看看R直接用Drill的REST方式去query到hive的結果

```
library(httr)
library(jsonlite)

post_data <- POST("http://192.168.0.121:8047/query.json", 
     body = list(queryType = "SQL", 
                 query = "SELECT v1,v2,avg(v3) v3_avg FROM hive_cassSpark1.test_df group by v1,v2"),
     encode = "json")

content(post_data, type = "text") %>>% fromJSON %>>% `[[`(2)
# No encoding supplied: defaulting to UTF-8.
#                  v3_avg v1 v2
# 1          -0.262449565  a  c
# 2 0.0014069844444444442  a  d
# 3   0.17559292428571424  a  e
# 4   0.28090609250000004  b  c
# 5  -0.03396260833333329  b  d
# 6          0.0773905475  b  e

# 這裡v3_avg會是字串，需要自己parse，但是如果很熟R的話，這一點應該不成問題
# 用下面指令即可
library(data.table)
content(post_data, type = "text") %>>% fromJSON %>>% `[[`(2) %>>% 
  data.table %>>% `[`( , lapply(.SD, type.convert, as.is = TRUE)) %>>% str
# Classes ‘data.table’ and 'data.frame':	6 obs. of  3 variables:
#  $ v3_avg: num  -0.26245 0.00141 0.17559 0.28091 -0.03396 ...
#  $ v1    : chr  "a" "a" "a" "b" ...
#  $ v2    : chr  "c" "d" "e" "c" ...
#  - attr(*, ".internal.selfref")=<externalptr> 
```

4. 將mysql納入supervisord

執行下面的指令即可

``` bash
sudo systemctl stop mysqld
sudo systemctl disable mysqld

sudo tee -a /etc/supervisor/supervisord.conf << "EOF"
[program:mysql]
command=/usr/bin/pidproxy /var/run/mysqld/mysqld.pid /usr/sbin/mysqld
stdout_logfile=/var/log/supervisor/mysql-stdout.out
stderr_logfile=/var/log/supervisor/mysql-stderr.out
autostart=true
startsecs=5
priority=80
user=mysql
EOF
sudo systemctl restart supervisor
```