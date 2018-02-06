---
layout: post
title: "sparklyr extensions"
---

稍微介紹一下sparklyr的extension寫法

但是細節都還在研究，只是環境的配置跟使用官方的extension套件而已

環境只有一點要注意，sparklyr只會找下面三個地方有沒有`scala-2.10`或是`scala-2.11`

1. /opt/local/scala
1. /usr/local/scala
1. /opt/scala

所以根據我之前的配置就需要重新做調整

使用下面的指令做調整的動作即可

``` bash
sudo mkdir /usr/local/scala
sudo mv /usr/local/bigdata/scala /usr/local/scala/scala-2.11
```

接著run下面的script

``` R
library(sparklyr)

# 下載sparkhello套件
if (!dir.exists("sparkhello"))
  git2r::clone("https://github.com/jjallaire/sparkhello.git", "sparkhello")
# 移除prebuild的jar檔
file.remove("sparkhello/inst/java/sparkhello-1.6-2.10.jar")
file.remove("sparkhello/inst/java/sparkhello-2.0-2.11.jar")

# 工作目錄設定在套件root
setwd("sparkhello")
# 指定jar名稱
jar_name <- sparklyr:::infer_active_package_name() %>>% paste0("-2.0-2.11.jar")
# 編譯jar
compile_package_jars(spark_version = "2.0.0", spark_home = Sys.getenv("SPARK_HOME"),
                     scalac_path = find_scalac("2.11"), jar_name = jar_name)
# build套件
devtools::build()
# install套件
devtools::install()
# 工作目錄回到上一層
setwd("..")

# 切斷之前的連線，這點很重要，不然用之前的連線的話
# 不會把套件中的jar檔load進去
spark_disconnect_all()

# library套件
library(sparkhello)
# create spark connection
spark_master <- "mesos://zk://192.168.0.121:2181,192.168.0.122:2181,192.168.0.123:2181/mesos"
sc <- spark_connect(master = spark_master, config = spark_config("config.yml", FALSE))

# 執行
spark_hello(sc)
# [1] "Hello, world! - From Scala"
```

以上就是sparklyr的extension部分

很可惜的是這個範例沒有講到怎麼傳參數到scala裡面

也不知道能不能設定一些函數，output到R裡面使用

