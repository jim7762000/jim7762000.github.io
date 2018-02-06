---
layout: post
title: "test on apache sqoop"
---

前四篇分別裝了Hadoop, Oracle, ROracle跟Python的cx_Oracle套件

上兩篇分別利用了ROracle跟cx_Oracle塞了一些資料進去Oracle

接下來是安裝sqoop，試試看用sqoop從Oracle DB把資料撈進HBase

這篇僅是紀錄而已，並沒有成功撈進，

1. 準備工作

基本上同Hadoop那篇，這裡就不贅述

我這裡是直接裝在Hadoop的master (sparkServer0)上

2. 安裝sqoop

從官網上下載下來，然後解壓縮，並加入環境變數：

``` bash
# 下載安裝sqoop
curl -v -j -k -L http://apache.stu.edu.tw/sqoop/1.4.6/sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz -o sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
tar -zxvf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
sudo mv sqoop-1.4.6.bin__hadoop-2.0.4-alpha /usr/local/sqoop
sudo chown -R tester /usr/local/sqoop

# 加入環境變數
sudo tee -a /etc/bashrc << "EOF"
export SQOOP_HOME=/usr/local/sqoop
export PATH=$PATH:$SQOOP_HOME/bin
export ZOOCFGDIR=$ZOOKEEPER_HOME/conf
EOF
source /etc/bashrc
```

Note: HBase 0.98版本才能兼容sqoop 1.4.6，如果是1.0以上版本請重新安裝HBase

3. 加入oracle連線jar

到[oracle官網](http://www.oracle.com/technetwork/apps-tech/jdbc-112010-090769.html)去下載oracle的ojdbc6.jar上傳到tester的home目錄中

執行`mv ~/ojdbc6.jar $SQOOP_HOME/lib`搬去sqoop下的lib

4. 開始執行

```
## 在sparkServer0開啟hadoop
start-dfs.sh & start-yarn.sh & zkServer.sh start & start-hbase.sh
## 把oracleServer開啟，他會自動開啟oracle database1

## 先列出所有的database看看
sqoop list-databases --connect jdbc:oracle:thin:@192.168.0.120:1521:orcl --username system --P
# Warning: /usr/local/sqoop/../hcatalog does not exist! HCatalog jobs will fail.
# Please set $HCAT_HOME to the root of your HCatalog installation.
# Warning: /usr/local/sqoop/../accumulo does not exist! Accumulo imports will fail.
# Please set $ACCUMULO_HOME to the root of your Accumulo installation.
# 16/07/31 19:14:03 INFO sqoop.Sqoop: Running Sqoop version: 1.4.6
# Enter password:
# 16/07/31 19:14:06 INFO oracle.OraOopManagerFactory: Data Connector for Oracle and Hadoop is disabled.
# 16/07/31 19:14:06 INFO manager.SqlManager: Using default fetchSize of 1000
# SLF4J: Class path contains multiple SLF4J bindings.
# SLF4J: Found binding in [jar:file:/usr/local/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: Found binding in [jar:file:/usr/local/hbase/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
# SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
# SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
# 16/07/31 19:14:37 INFO manager.OracleManager: Time zone has been set to GMT
# ORACLE_OCM
# OJVMSYS
# SYSKM
# XS$NULL
# GSMCATUSER
# MDDATA
# SYSBACKUP
# DIP
# SYSDG
# APEX_PUBLIC_USER
# SPATIAL_CSW_ADMIN_USR
# SPATIAL_WFS_ADMIN_USR
# GSMUSER
# AUDSYS
# FLOWS_FILES
# DVF
# MDSYS
# ORDSYS
# DBSNMP
# WMSYS
# APEX_040200
# APPQOSSYS
# GSMADMIN_INTERNAL
# ORDDATA
# CTXSYS
# ANONYMOUS
# XDB
# ORDPLUGINS
# DVSYS
# SI_INFORMTN_SCHEMA
# OLAPSYS
# LBACSYS
# OUTLN
# SYSTEM
# SYS
# C##DATASETS_1
# C##DATASETS_2
# C##DATASETS_3

## 再來測試看看拉oracle中的表格C##DATASETS_1.IRIS到HDFS上去
sqoop import \
--connect jdbc:oracle:thin:@192.168.0.120:1521:orcl \
--username system --P \
--query "SELECT * FROM C##DATASETS_1.IRIS" \
--split-by SPECIES -m 5 --target-dir /user/tester/iris

## 出現錯誤：
# 16/07/31 19:57:28 ERROR tool.ImportTool: Encountered IOException running import job: 
# java.io.IOException: Query [SELECT * FROM C##DATASETS_1.IRIS] must contain 
# '$CONDITIONS' in WHERE clause.

## 加入CONDITIONS再一次
export CONDITIONS=1=1
sqoop import \
--connect jdbc:oracle:thin:@192.168.0.120:1521:orcl \
--username system --P \
--query "SELECT * FROM C##DATASETS_1.IRIS WHERE 1=1 AND \$CONDITIONS" \
--split-by SPECIES -m 5 --target-dir /user/tester/iris
## 出現錯誤：
# 16/07/31 20:00:25 ERROR manager.SqlManager: Error executing statement: 
# java.sql.SQLRecoverableException: IO Error: Connection reset
# java.sql.SQLRecoverableException: IO Error: Connection reset


## 換成匯到HBase試試看
export CONDITIONS=1=1
sqoop import \
--connect jdbc:oracle:thin:@192.168.0.120:1521:orcl \
--username system --P \
--query "SELECT SEPALLENGTH,SEPALWIDTH,PETALLENGTH,PETALWIDTH,SPECIES FROM C##DATASETS_1.IRIS WHERE 1=1 AND \$CONDITIONS" \
--hbase-table C##DATASETS_1.IRIS --hbase-create-table \
--hbase-row-key id --split-by SPECIES -m 10 \
--column-family SEPALLENGTH,SEPALWIDTH,PETALLENGTH,PETALWIDTH,SPECIES
## 出現錯誤：
# 16/07/31 20:01:36 ERROR manager.SqlManager: Error executing statement: 
# java.sql.SQLRecoverableException: IO Error: Connection reset
# java.sql.SQLRecoverableException: IO Error: Connection reset

## 試試看網路解法，出現不同錯誤
sqoop import -D mapred.child.java.opts="\-Djava.security.egd=file:/dev/../dev/urandom" \
--connect jdbc:oracle:thin:@192.168.0.120:1521:orcl \
--username system --P \
--query "SELECT SEPALLENGTH,SEPALWIDTH,PETALLENGTH,PETALWIDTH,SPECIES FROM C##DATASETS_1.IRIS WHERE 1=1 AND \$CONDITIONS" \
--hbase-table C##DATASETS_1.IRIS --hbase-create-table \
--hbase-row-key id --split-by SPECIES -m 10 \
--column-family SEPALLENGTH,SEPALWIDTH,PETALLENGTH,PETALWIDTH,SPECIES
## 出現錯誤
# ERROR tool.ImportTool: Imported Failed: Illegal character code:35, <#> at 1. 
# User-space table qualifiers can only contain 'alphanumeric characters': 
# i.e. [a-zA-Z_0-9-.]: C##DATASETS_1.IRIS

## 換使用者在一次，還是出現一樣的錯誤
sqoop import -D mapred.child.java.opts="\-Djava.security.egd=file:/dev/../dev/urandom" \
--connect jdbc:oracle:thin:@192.168.0.120:1521:orcl \
--username C##DATASETS_1 --P \
--query "SELECT SEPALLENGTH,SEPALWIDTH,PETALLENGTH,PETALWIDTH,SPECIES FROM C##DATASETS_1.IRIS WHERE 1=1 AND \$CONDITIONS" \
--hbase-table IRIS --hbase-create-table \
--hbase-row-key id --split-by SPECIES -m 10 \
--column-family SEPALLENGTH,SEPALWIDTH,PETALLENGTH,PETALWIDTH,SPECIES
## 出現錯誤
# 16/08/02 20:12:17 ERROR manager.SqlManager: Error executing statement: 
# java.sql.SQLRecoverableException: IO Error: Connection reset
# java.sql.SQLRecoverableException: IO Error: Connection reset
```

然後我就放棄再嘗試了，我中間還有把HBase從1.1.0降板到0.98，但是還是沒用，特此紀錄。

5. Reference

1. http://www.cnblogs.com/byrhuangqiang/p/3922594.html
1. https://www.zybuluo.com/aitanjupt/note/209968
1. http://blog.csdn.net/u010330043/article/details/51441135
1. http://www.cnblogs.com/smartloli/p/4202710.html