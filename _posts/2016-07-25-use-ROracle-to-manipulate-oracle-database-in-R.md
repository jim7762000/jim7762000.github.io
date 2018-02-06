---
layout: post
title: 在R用ROracle去操作Oracle資料庫
---

前兩篇裝了Hadoop跟Oracle

為了接下來可以測試sqoop，使用ROracle去塞一下資料表進去

在windows下，安裝ROracle，也測試看看在centos下安裝看看

(ubuntu, mint部分前面有文章介紹怎麼裝R，就不在贅述，至於裝ROracle就跟centos大同小異了)

1. 準備工作

基本上同Hadoop那篇，這裡就不贅述

我這裡是直接裝在Hadoop的master (sparkServer0)上

centos部分： (windows請往下轉)

2. 安裝Microsoft R Open

主要參考我自己前幾篇文章 - [
Some installations of centos](http://chingchuan-chen.github.io/linux/2016/05/11/initialization_of_centos.html)

簡單敘述一下(不同的地方是centos最小安裝沒有wget)：

``` bash
## 更新repo，並安裝R
su -c 'rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm'
sudo yum update
sudo yum install gcc-c++ R R-devel R-java

## remove R
sudo rm -rf /usr/lib64/R

# install MRO
## Make sure the system repositories are up-to-date prior to installing Microsoft R Open.
sudo yum clean all
## get the installers
curl -v -j -k -L https://mran.microsoft.com/install/mro/3.3.0/MRO-3.3.0.el7.x86_64.rpm -o MRO-3.3.0.el7.x86_64.rpm 
curl -v -j -k -L https://mran.microsoft.com/install/mro/3.3.0/RevoMath-3.3.0.tar.gz -o RevoMath-3.3.0.tar.gz
## 安裝MRO
sudo yum install MRO-3.3.0.el7.x86_64.rpm
## 安裝MKL
tar -xzf RevoMath-3.3.0.tar.gz
## 備註一點，這裡如果直接用 sudo bash ./RevoMath/RevoMath.sh會失敗
## 失敗之後要先 sudo yum remove MRO重裝一次
cd RevoMath
sudo bash ./RevoMath.sh

## 開啟Library讀寫權限
sudo chmod -R 777 /usr/lib64/MRO-3.3.0/R-3.3.0/lib64/R

## 安裝之後打開R
R
## 會出現下方訊息
# R version 3.3.0 (2016-05-03) -- "Supposedly Educational"
# Copyright (C) 2016 The R Foundation for Statistical Computing
# Platform: x86_64-pc-linux-gnu (64-bit)
# 
# R is free software and comes with ABSOLUTELY NO WARRANTY.
# You are welcome to redistribute it under certain conditions.
# Type 'license()' or 'licence()' for distribution details.
# 
#   Natural language support but running in an English locale
# 
# R is a collaborative project with many contributors.
# Type 'contributors()' for more information and
# 'citation()' on how to cite R or R packages in publications.
# 
# Type 'demo()' for some demos, 'help()' for on-line help, or
# 'help.start()' for an HTML browser interface to help.
# Type 'q()' to quit R.
# 
# Microsoft R Open 3.3.0
# Default CRAN mirror snapshot taken on 2016-06-01
# The enhanced R distribution from Microsoft
# Visit https://mran.microsoft.com/ for information
# about additional features.
# 
# Multithreaded BLAS/LAPACK libraries detected. Using 2 cores for math algorithms.
## 有上面這行代表MKL才有裝成功

## 安裝一些ROracle的必要套件
R -e 'install.packages("DBI")'
## 也可以打開R輸入指令

## 安裝一些等等資料用到的套件
R -e 'install.packages(c("data.table", "pipeR", "dplyr", "GGally", "ggplot2", "nycflights13"))'
## 也可以打開R輸入指令
```

3. 安裝ROracle

先從[官方網站](http://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html)下載Oracle instant client，下載basck.rpm跟devel.rpm(詳細檔名看下方)，然後上傳到VM

接著下載ROracle原始包，利用`R CMD INSTALL`並同時configure做安裝的動作，指令如下：

``` bash
# 安裝Oracle instant client
sudo yum install oracle-instantclient12.1-basic-12.1.0.2.0-1.x86_64.rpm
sudo yum install oracle-instantclient12.1-devel-12.1.0.2.0-1.x86_64.rpm

# 設定環境變數
sudo tee -a /etc/bashrc << "EOF"
export ORACLE_HOME=/usr/lib/oracle/12.1/client64
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib  
export PATH=$PATH:$ORACLE_HOME/bin
EOF

# 下載ROracle套件
curl -v -j -k -L https://cran.r-project.org/src/contrib/ROracle_1.2-2.tar.gz -o ROracle_1.2-2.tar.gz

# 安裝ROracle
R CMD INSTALL --configure-args='--with-oci-lib=/usr/lib/oracle/12.1/client64/lib --with-oci-inc=/usr/include/oracle/12.1/client64' ROracle_1.2-2.tar.gz
```

windows部分：

2. 安裝MRO跟MKL

到[官方網站](https://mran.revolutionanalytics.com/download/)下載安裝包

然後安裝MRO跟MKL，安裝之後，利用installr安裝Rtools

打開R輸入

``` R
install.packages("installr")
installr::install.Rtools()
```

接著安裝一些需要的套件：

``` R
## 安裝一些ROracle的必要套件
install.packages("DBI")

## 安裝一些等等資料用到的套件
install.packages(c("data.table", "pipeR", "dplyr", "GGally", "ggplot2", "nycflights13"))
```

然後一樣到Oracle的官方網站下載instant client，要下載basic跟sdk

解壓縮放到`C:\instantclient_12_1`

接著下載R的套件包：https://cran.r-project.org/src/contrib/ROracle_1.2-2.tar.gz

在安裝的目錄下新增一個`install_ROracle.bat`的檔案

按右鍵編輯貼上下方內容後執行就可以成功安裝ROracle了

``` batch
SET OCI_LIB64=C:\instantclient_12_1
SET OCI_INC=C:\instantclient_12_1\sdk\include
SET PATH=%PATH%;C:\instantclient_12_1
R CMD INSTALL --build ROracle_1.2-2.tar.gz
```

PS: 這裡我還沒有測試過，因為避免牽扯到windwos的環境變數就省掉

如果有人測試過不行，我在提供增加的環境變數
    
我是直接安裝Rtools手動設定環境變數... 比較不適合一般初學者

4. 創造適當的使用者

我想要創造三個使用者，等等利用ROracle上傳到這三個使用者下的schema，SQL內容如下：

``` sql
CREATE TABLESPACE nycflights13
  DATAFILE 'nycflights13.dat'
  SIZE 40M REUSE AUTOEXTEND ON;

CREATE TEMPORARY TABLESPACE nycflights13_tmp
  TEMPFILE 'nycflights13_tmp.dbf'
  SIZE 10M REUSE AUTOEXTEND ON;
  
CREATE USER C##nycflights13
IDENTIFIED BY nycflights13
DEFAULT TABLESPACE nycflights13
TEMPORARY TABLESPACE nycflights13_tmp
quota unlimited on nycflights13;

GRANT CREATE SESSION,
      CREATE TABLE,
      CREATE VIEW,
      CREATE PROCEDURE
TO C##nycflights13; 

CREATE TABLESPACE datasets
  DATAFILE 'datasets.dat'
  SIZE 40M AUTOEXTEND ON;

CREATE TEMPORARY TABLESPACE datasets_tmp
  TEMPFILE 'datasets_tmp.dbf'
  SIZE 10M AUTOEXTEND ON;

CREATE USER C##datasets
IDENTIFIED BY datasets
DEFAULT TABLESPACE datasets
TEMPORARY TABLESPACE datasets_tmp
quota unlimited on datasets;

GRANT CREATE SESSION,
      CREATE TABLE,
      CREATE VIEW,
      CREATE PROCEDURE
TO C##datasets; 

CREATE TABLESPACE hadley
  DATAFILE 'hadley.dat'
  SIZE 40M AUTOEXTEND ON;

CREATE TEMPORARY TABLESPACE hadley_tmp
  TEMPFILE 'hadley_tmp.dbf'
  SIZE 10M AUTOEXTEND ON;

CREATE USER C##hadley
IDENTIFIED BY hadley
DEFAULT TABLESPACE hadley
TEMPORARY TABLESPACE hadley_tmp
quota unlimited on hadley;

GRANT CREATE SESSION,
      CREATE TABLE,
      CREATE VIEW,
      CREATE PROCEDURE
TO C##hadley; 

/* DELETE USERS AND TABLESPACES
DROP USER C##hadley;
DROP USER C##datasets;
DROP USER C##nycflights13;
DROP TABLESPACE hadley;
DROP TABLESPACE datasets;
DROP TABLESPACE nycflights13;
DROP TABLESPACE hadley_tmp;
DROP TABLESPACE datasets_tmp;
DROP TABLESPACE nycflights13_tmp;
*/
```

PS: C##是Oracle要求的，請搜尋common users vs local users oracle就知道差異，還有命名要求

把上面的SQL下載下來命名成`createUser.sql`並上傳到Oracle的server

使用`$ORACLE_HOME/bin/sqlplus system/password@oracleServer:1521/orcl @createUser.sql`來執行創造使用者的SQL

5. 使用ROracle去上傳資料

程式如下：

``` R
library(ROracle)
library(data.table)
library(pipeR)
Sys.setenv(TZ = "Asia/Taipei", ORA_SDTZ = "Asia/Taipei")

# 創造資料列表
# 全部資料大小: 60 Mb
uploadDataList <- '
TableName,pkgName,TableSize,savingSchema,savingTblName
airlines,nycflights13,"2.9 Kb",nycflights13,airlines
airports,nycflights13,"228.9 Kb",nycflights13,airports
flights,nycflights13,"38.7 Mb",nycflights13,flights
planes,nycflights13,"347.9 Kb",nycflights13,planes
weather,nycflights13,"2.8 Mb",nycflights13,weather
iris,datasets,"6.9 Kb",datasets,iris
cars,datasets,"1.5 Kb",datasets,cars
USCancerRates,latticeExtra,"504.8 Kb",datasets,uscancerrates
gvhd10,latticeExtra,"6.5 Mb",datasets,gvhd10
nasa,GGally,"4.6 Mb",hadley,nasa
happy,GGally,"2.7 Mb",hadley,happy
diamonds,ggplot2,"3.3 Mb",hadley,diamonds
txhousing,ggplot2,"541.8 Kb",hadley,txhousing
'

# 讀取列表
tblListDT <- fread(uploadDataList)
# 讀取資料
tblListDT %>>% apply(1, function(v){
  data(v[1], package = v[2])
}) %>>% invisible 

#連線資訊
host <- "192.168.0.120"
port <- 1521
sid <- "orcl"
connectString <- paste(
  "(DESCRIPTION=",
  "(ADDRESS=(PROTOCOL=TCP)(HOST=", host, ")(PORT=", port, "))",
  "(CONNECT_DATA=(SID=", sid, ")))", sep = "")

# 先用system權限登入查看user
con <- dbConnect(dbDriver("Oracle"), username = "system", 
                 password = "qscf12356", dbname = connectString)
# query all_users
userList <- dbSendQuery(con, "select * from ALL_USERS") %>>% 
  fetch(n = -1) %>>% data.table
# 印出表格，user按照創造時間排列
# 可以看到已經創造了C##NYCFLIGHTS13, C##DATASETS跟C##HADLEY三個user
print(userList[order(CREATED)])
# system帳號斷線
dbDisconnect(con)

st <- proc.time()
uniqueUserNames <- unique(tblListDT$savingSchema)
sapply(uniqueUserNames, function(userName){
  # filter要上傳的表
  uploadDataDT <- tblListDT[savingSchema == userName]
  mapply(function(tblName, pkgName) data(list = tblName, package = pkgName),
         uploadDataDT$TableName, uploadDataDT$pkgName) %>>% invisible
  # 用user去上傳，這樣才會傳到以該user命名的schema下
  con <- dbConnect(dbDriver("Oracle"), username = paste0("C##", userName), 
                   password = userName, dbname = connectString)
  # 把每一張要上傳的表 上傳上去
  mapply(function(tblName, tblNameUpload){
    if (!dbExistsTable(con, tblNameUpload))
      dbWriteTable(con, tblNameUpload, as.data.frame(get(tblName)), row.names = FALSE)
    }, uploadDataDT$TableName, uploadDataDT$savingTblName) %>>% invisible
  dbDisconnect(con)
}) %>>% invisible
proc.time() - st
# user  system elapsed 
# 0.95    0.16   11.56 

# 移除所有表格
rm(list = tblListDT$TableName)

con <- dbConnect(dbDriver("Oracle"), username = "system", 
                 password = "qscf12356", dbname = connectString)
# list schema
dbListTables(con) 
# find the table in current schema (parameter, schema = NULL)
dbExistsTable(con, "airlines")
# find the table in the schema
dbExistsTable(con, "airlines", schema = 'C##NYCFLIGHTS13')
# remove table
dbRemoveTable(con, "airlines")
# query data, fetch all data and convert to data.table (這是查詢全部的tablespaces)
dbSendQuery(con, "select * from C##NYCFLIGHTS13.\"airlines\"") %>>% 
  fetch(n = -1) %>>% data.table
# query rowid and name from airlines data, 
# fetch all data and convert to data.table (這是查詢全部的tablespaces)
dbSendQuery(con, "select rowid,t.\"name\" from C##NYCFLIGHTS13.\"airlines\" t") %>>% 
  fetch(n = -1) %>>% data.table
# query data, fetch all data and convert to data.table (這是查詢全部的tablespaces)
dbSendQuery(con, "select * from dba_tablespaces") %>>% 
  fetch(n = -1) %>>% data.table
# query data, fetch all data and convert to data.table (這是查詢全部的tables)
dbSendQuery(con, "select * from all_tables") %>>% 
  fetch(n = -1) %>>% data.table
# query data, fetch all data and convert to data.table (這是查詢全部的users)
dbSendQuery(con, "select * from all_users") %>>% 
  fetch(n = -1) %>>% data.table
# 從DB斷線
dbDisconnect(con)

# remove all tables (用最高權限刪除)
st <- proc.time()
con <- dbConnect(dbDriver("Oracle"), username = "system", 
                 password = "qscf12356", dbname = connectString)
mapply(function(schemaName, tblNameUpload){
  sn <- paste0("C##", schemaName) %>>% toupper
  if (dbExistsTable(con, tblNameUpload, schema = sn))
    dbRemoveTable(con, tblNameUpload, schema = sn)
}, tblListDT$savingSchema, tblListDT$savingTblName) %>>% invisible
dbDisconnect(con)
proc.time() - st
# user  system elapsed 
# 0.02    0.00    0.35 
```

6. 小抱怨

ROracle寫入表格的時候，會自動加上double quote框住表格

因為這一個自動功能，讓我debug，de了一天晚上

最後是去下載Oracle PL/SQL Developer用介面的自動提示才發現這件事情...

然後才回去看dbWriteTable下面的解釋

> Table, schema, and column names are case sensitive, e.g., table names ABC and abc are not the same. All database schema object names should not include double quotes as they are enclosed in double quotes when the corresponding SQL statement is generated.

只是最神奇的事情是

``` R
dbWriteTable(con, "airlines", as.data.frame(airlines), row.names = FALSE)
```

寫入表格之後，query的表格名字要加quote，像這樣：

``` R
dbSendQuery(con, "select * from \"airlines\"") %>>% fetch(n = -1) %>>% data.table
```
我查表是否存在不用quote

```
dbExistsTable(con, "airlines")
```

害我一直想說為啥我找不到我的表

最後只能去安裝Oracle SQL Developer查看真正的表格名稱

7/31補充：

更扯的事情是column name都有包quote，select時候，都要多打""去框住column name

8/12更新：

全部使用大寫字元，就不會產生上面的情況

