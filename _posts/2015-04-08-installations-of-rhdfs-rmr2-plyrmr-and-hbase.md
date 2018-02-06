---
layout: post
title: Installations of rhdfs, rmr2, plyrmr and rhbase
---

Rsudio provides a series of packages for the connection between R and hadoop. rhdfs provides the manipulation of HDFS in hadoop in R. rmr2 and plyrmr let user do mapreduce job in R. rhbase allow user to access data in hbase.

Before using packages in R, we implement wordcount by using hadoop streaming. First, export the hadoop home in the terminal by command `export HADOOP_HOME=/usr/local/hadoop/`. New two R files named mapper.R and reducer.R, respectively.

``` R
#! /usr/bin/env Rscript
## mapper.R
trimWhiteSpace <- function(line) gsub("(^ +)|( +$)", "", line)
splitIntoWords <- function(line) unlist(strsplit(line, "[[:space:]]+"))
con <- file("stdin", open = "r")
while (length(line <- readLines(con, n = 1, warn = FALSE)) > 0) {
    line <- trimWhiteSpace(line)
    words <- splitIntoWords(line)
    ## **** can be done as cat(paste(words, "\t1\n", sep=""), sep="")
    for (w in words)
        cat(w, "\t1\n", sep="")
}
close(con)
```

``` R
#! /usr/bin/env Rscript
# reducer.R
trimWhiteSpace <- function(line) gsub("(^ +)|( +$)", "", line)
splitLine <- function(line) {
    val <- unlist(strsplit(line, "\t"))
    list(word = val[1], count = as.integer(val[2]))
}
env <- new.env(hash = TRUE)
con <- file("stdin", open = "r")
while (length(line <- readLines(con, n = 1, warn = FALSE)) > 0) {
    line <- trimWhiteSpace(line)
    split <- splitLine(line)
    word <- split$word
    count <- split$count
    if (exists(word, envir = env, inherits = FALSE)) {
        oldcount <- get(word, envir = env)
        assign(word, oldcount + count, envir = env)
    }
    else assign(word, count, envir = env)
}
close(con)
for (w in ls(env, all = TRUE))
    cat(w, "\t", get(w, envir = env), "\n", sep = "")
```

Using the example in previous article for hadoop and run hadoop streaming in the terminal:

``` bash
cd ~/Downloads && mkdir testData && cd testData
wget http://www.gutenberg.org/ebooks/5000.txt.utf-8
cd ..
hdfs dfs -copyFromLocal testData/ /user/celest/
hadoop jar /usr/local/hadoop/share/hadoop/tools/lib/hadoop-streaming-2.6.0.jar \
-files mapper.R,reducer.R,/opt/intel/composer_xe_2013_sp1.3.174/compiler/lib/intel64/libiomp5.so \
-mapper "mapper.R -m" -reducer "reducer.R -r" \
-input /user/celest/testData/* -output /user/celest/testData2-output
hdfs dfs -cat /user/celest/testData2-output/part-00000
```

We can obtain the same result for wordcount. Next, we are going to install the following packages:

``` bash
rhdfs
rmr
ravro
plyrmr
rhbase
```

Note that ravro is the data parser for avro in R.

Some dependencies need to be installed in system:

``` bash
sudo apt-get install libcurl4-openssl-dev git
R CMD javareconf # setting the environment for R
# for rhbase
sudo apt-get install libboost-dev libboost-test-dev libboost-program-options-dev libboost-system-dev libboost-filesystem-dev libevent-dev automake libtool flex bison pkg-config g++ libssl-dev
# optional package to install
sudo apt-get install php5-dev php5-cli phpunit libbit-vector-perl libclass-accessor-class-perl python-all python-all-dev python-all-dbg libglib2.0-dev ruby-full ruby-dev ruby-rspec rake
sudo gem install daemons gem_plugin mongrel
wget http://apache.stu.edu.tw//ant/binaries/apache-ant-1.9.4-bin.tar.gz
tar -zxvf apache-ant-1.9.4-bin.tar.gz
cd apache-ant-1.9.4-bin
sudo mv apache-ant-1.9.4-bin /usr/local/ant
cd /usr/local
sudo chown -R celest ant
# install mongrel may occur error, to see http://stackoverflow.com/questions/13851741/install-mongrel-in-ruby-1-9-3

# install thrift (for rhbase)
cd ~/Downloads
wget http://apache.stu.edu.tw/thrift/0.9.2/thrift-0.9.2.tar.gz
tar -zxvf thrift-0.9.2.tar.gz
cd thrift-0.9.2
./configure
make
sudo make install
sudo cp /usr/local/lib/libthrift-0.9.2.so /usr/lib/
sudo /sbin/ldconfig /usr/lib/libthrift-0.9.2.so
```

``` bash
sudo subl /etc/bash.bashrc
# add following 7 lines into file
# # for rhbase
# export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig/
# # for rhdfs and rmr2
# export HADOOP_CMD=/usr/local/hadoop/bin/hadoop
# # for rmr2
# export HADOOP_STREAMING=/usr/local/hadoop/share/hadoop/tools/lib/hadoop-streaming-2.6.0.jar
subl /usr/local/hadoop/etc/hadoop/hadoop-env.sh
# # for compiled R by mkl
# source /opt/intel/composer_xe_2013_sp1.3.174/mkl/bin/mklvars.sh intel64
# source /opt/intel/composer_xe_2015.1.133/mkl/bin/mklvars.sh intel64
# # for compiled R by icc
# source /opt/intel/composer_xe_2013_sp1.3.174/bin/compilervars.sh intel64
# source /opt/intel/composer_xe_2015.1.133/bin/compilervars.sh intel64
# # for hive
# export R_HOME=/usr/lib/R

source /etc/bash.bashrc
# in ubuntu, is >> etc/bash.bashrc
start-dfs.sh
start-yarn.sh
hive --service hiveserver
```

Install the dependent R packages:

``` R
install.packages(c("rJava", "Rcpp", "rjson", "RJSONIO", "bit64", "reshape2", "data.table", "plyr", "dplyr", "digest", "functional", "stringr", "caTools", "lazyeval", "Hmisc", "testthat", "devtools", "iterators", "itertools", "pryr"))
library(devtools)
install_github("RevolutionAnalytics/quickcheck@3.2.0", subdir = "pkg")
install_github("RevolutionAnalytics/memoise")
install_github("RevolutionAnalytics/rhdfs", subdir = "pkg")
install_github("RevolutionAnalytics/ravro", subdir = "pkg/ravro")
install_github("RevolutionAnalytics/rmr2", subdir = "pkg")
install_github("RevolutionAnalytics/plyrmr", subdir = "pkg")
install_github("RevolutionAnalytics/rhbase", subdir = "pkg")
```

When I install rhbase, I encounter a problem. The command `pkg-config --cflags thrift` does not return `-I/usr/local/include` instead the correct path `-I/usr/local/include/thrift`. So I copy all files in `/usr/local/include/thrift` into `-I/usr/local/include` by `cp -R /usr/local/include/thrift/* /usr/local/include/`.

When I use the function `hdfs.init()` in package `rhdfs`, it come to a error massage:

``` R
sh: /usr/lib/hadoop/bin/: is a directory
Error in .jnew("org/apache/hadoop/conf/Configuration") :
   java.lang.ClassNotFoundException
In addition: Warning message:
running command '/usr/lib/hadoop/bin/ classpath' had status 126
```

The reason why cause this problem is wrong setting of HADOOP_CMD, fix it and get work.

install rhbase, I encounter a problem. The command `pkg-config --cflags thrift` does not return `-I/usr/local/include` instead the correct path `-I/usr/local/include/thrift`. So I copy all files in `/usr/local/include/thrift` into `-I/usr/local/include` by `cp -R /usr/local/include/thrift/* /usr/local/include/`.

``` bash
cd ~/Downloads
git clone https://github.com/RevolutionAnalytics/rmr2.git
mv rmr2/pkg pkg
rm -r rmr2
mv pkg rmr2
subl rmr2/R/streaming.R
R CMD INSTALL rmr2
```

The lines shown in following:

``` R
paste.options(
  files =
    paste(
      collapse = ",",
        c(image.files, map.file, reduce.file, combine.file)))
```

can be modified to following:

``` R
## for intel parallel studio 2013
paste.options(
  files =
    paste(
      collapse = ",",
        c(image.files, map.file, reduce.file, combine.file,
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libiomp5.so",
                "/usr/lib/jvm/java-8-oracle/jre/lib/amd64/server/libjvm.so"))),

## for intel parallel studio 2015
paste.options(
  files =
    paste(
      collapse = ",",
        c(image.files, map.file, reduce.file, combine.file,
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libiomp5.so",
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libifport.so.5",
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libifcoremt.so.5",
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libimf.so",
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libsvml.so",
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libirc.so",
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libirng.so",
                "/opt/intel/composer_xe_2015.1.133/compiler/lib/intel64/libintlc.so.5",
                "/usr/lib/jvm/java-8-oracle/jre/lib/amd64/server/libjvm.so"))),
```

Test rhadoop series packages:

``` R
# test rhdfs
library(rhdfs)
library(data.table)
library(dplyr)
library(magrittr)
N = 300
mydata = replicate(3, rnorm(N)) %>% tbl_dt() %>%
  setnames(paste0("x", 1:3)) %>% mutate(y = x1+2*x2+3*x3+rnorm(N,0,5))
model = lm(y ~., mydata)
hdfs.init()
modelfilename = "mymodel"
modelfile = hdfs.file(modelfilename, "w")
hdfs.write(model, modelfile)
hdfs.close(modelfile)
model
# Call:
# lm(formula = y ~ ., data = mydata)
#
# Coefficients:
# (Intercept)           x1           x2           x3
#      0.1105       1.9879       2.6298       3.0040

modelfile = hdfs.file(modelfilename, "r")
m = hdfs.read(modelfile)
model2 = unserialize(m)
hdfs.close(modelfile)
model2
# Call:
# lm(formula = y ~ ., data = mydata)
#
# Coefficients:
# (Intercept)           x1           x2           x3
#      0.1105       1.9879       2.6298       3.0040


N = 3000
mydata = replicate(3, rnorm(N)) %>% tbl_dt() %>%
  setnames(paste0("x", 1:3)) %>% mutate(y = x1+2*x2+3*x3+rnorm(N,0,5))
model = lm(y ~., mydata)
modelfilename = "my_smart_unique_name"
modelfile = hdfs.file(modelfilename, "w")
hdfs.write(model, modelfile)
hdfs.close(modelfile)
model

# Call:
# lm(formula = y ~ ., data = mydata)
#
# Coefficients:
# (Intercept)           x1           x2           x3
#      0.2506       1.2038       1.9056       3.0477

modelfile = hdfs.file(modelfilename, "r")
m = hdfs.read(modelfile)itertools
model2 = unserialize(m)
# error in reading data
hdfs.close(modelfile)

length(serialize(model,NULL))
# [1] 131535
length(m)
# [1] 65536
# it return the wrong size!

modelfile = hdfs.file(modelfilename, "r")
fileSize = hdfs.file.info(paste0("/user/celest/", modelfilename))$size
m = NULL
i = 1
repeat
{
  break_out = 65536*i > fileSize
  size = ifelse(break_out, 65536, fileSize - 65536*(i-1))
  tmp = hdfs.read(modelfile, size, 65536*(i-1))
  m = c(m, tmp)
  i = i + 1
  if (break_out)
    break
}
model2 = unserialize(m)
hdfs.close(modelfile)
length(m)
# [1] 131535
model2
# to delete this data by using following command
# hdfs.rm("/user/celest/my_smart_unique_name")

# Call:
# lm(formula = y ~ ., data = mydata)
#
# Coefficients:
# (Intercept)           x1           x2           x3
#      0.2506       1.2038       1.9056       3.0477
```

Back to terminal, you can use hdfs to find what R write. (There is some functions in R doing the same way.)

``` bash
hdfs dfs -ls # hdfs.ls("/user/celest") in R
## Found 1 items
## -rw-r--r--   3 celest supergroup     131535 2015-04-06 13:50 my_smart_unique_name
# remove the file
hdfs dfs -rm -r my_smart_unique_name # hdfs.rm("/user/celest/my_smart_unique_name") in R
```

``` R
# Sys.setenv("HADOOP_CMD"="/usr/local/hadoop/bin/hadoop")
# test the rmr2 plyrmr
library(data.table)
library(dplyr)
library(magrittr)
library(rmr2)
library(plyrmr)
library(rhdfs)
# a simple case
ints = to.dfs(1:100)
calc = mapreduce(input = ints, map = function(k, v) cbind(v, v=2*v))
output = from.dfs(calc)
output
ints2 = to.dfs(matrix(rnorm(25), 5))
calc2 = mapreduce(input = ints2, map = function(k, v) v %*% v)
output2 = from.dfs(calc2)
output2
# $key
# NULL
#
# $val
#          v
#   [1,]   1   2
#   [2,]   2   4
#   [3,]   3   6
#   [4,]   4   8
#   [5,]   5  10
#   [6,]   6  12
#   [7,]   7  14
#   [8,]   8  16
#   ............

N = 15
dat = replicate(3, rnorm(N)) %>% tbl_dt() %>%
  setnames(paste0("x", 1:3)) %>% mutate(y = x1+2*x2+3*x3+rnorm(N,0,5)) %>%
  as.data.frame()
hdfs.init()
mydata = to.dfs(dat)
as.data.frame(input(mydata), x1_x2 = x1*x2)
bind.cols(input(mydata), x1_x2 = x1*x2)
output(bind.cols(input(mydata),x1_x2 = x1*x2), "/tmp/mydata2")

# test the rhbase
hbase thrift start
R
R > library(rhbase)
R > hb.list.tables() # list()
R > q("no")
```
