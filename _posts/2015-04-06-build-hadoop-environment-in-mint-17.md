---
layout: post
title: Build hadoop environment in mint 17
---

Hadoop is one of the most popular tool to deal with the big data. I construct the environment of Hadoop in mint 17. Mint 17 is based on the ubuntu 14.04. The following steps also works in ubuntu 14.04.

1. install java jdk 8

``` bash
sudo apt-get install python-software-properties
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
javac -version # check the version of java

# remove openjdk java
sudo apt-get purge openjdk-\*
# java version vs hadoop version
# refer: http://wiki.apache.org/hadoop/HadoopJavaVersions
```

2. install ssh

``` bash
sudo apt-get install ssh rsync openssh-server
ssh-keygen -t rsa -P "" # generate SSH key
# Enable SSH Key
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
# test whether it works, it should not need password if it works
ssh localhost
exit
```

3. download hadoop

``` bash
wget http://apache.stu.edu.tw/hadoop/common/stable2/hadoop-2.7.2.tar.gz
tar zxvf hadoop-2.7.2.tar.gz
sudo mv hadoop-2.7.2 /usr/local/hadoop
cd /usr/local
sudo chown -R celest hadoop
```

4. setting environment for java and hadoop

``` bash
sudo subl /etc/bash.bashrc
# add following 9 lines into file
# export JAVA_HOME=/usr/lib/jvm/java-8-oracle/
# export HADOOP_PID_DIR=/usr/local/hadoop/pids/
# export HADOOP_INSTALL=/usr/local/hadoop
# export PATH=$PATH:$HADOOP_INSTALL/bin
# export PATH=$PATH:$HADOOP_INSTALL/sbin
# export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
# export HADOOP_COMMON_HOME=$HADOOP_INSTALL
# export HADOOP_HDFS_HOME=$HADOOP_INSTALL
# export YARN_HOME=$HADOOP_INSTALL
source /etc/bash.bashrc
# in ubuntu, is >> etc/bash.bashrc
```

4-1. network environment (for multi-node hadoop)
If you use the VMware, you need to add another host-only network card. You can install hadoop successfully and clone it to be slaves. In following, `master` stands for the primary node and `slaveXX` for the other nodes.

a. setting the network

using `ifconfig` to check whether the network card is adding.

``` bash
sudo subl /etc/network/interfaces
```

the content of file looks like this:
(there must be eth1. Put a address for it on the machines master and slaves.)

``` bash
# The loopback network interface for master
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp

# Host-Only
auto eth1
  iface eth1:0 inet static
  address 192.168.29.130 # (192.168.29.131 for slave01)
  netmask 255.255.0.0
```

restart network by command `sudo /etc/init.d/networking restart`
and check the network again by `ifconfig`.

5. setup for hadoop

a. disabling IPv6 by editing `/etc/sysctl.conf`

``` bash
subl /etc/sysctl.conf
```

paste the following into `/etc/sysctl.conf`

``` bash
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

checking whether ipv6 is disable: `cat /proc/sys/net/ipv6/conf/all/disable_ipv6`. (retuen `1`)

b. editing following files:

create the folders for putting data.

``` bash
cd /usr/local/hadoop
sudo mkdir -p /usr/local/hadoop/tmp
sudo chown celest /usr/local/hadoop/tmp
subl /usr/local/hadoop/etc/hadoop/hadoop-env.sh
subl /usr/local/hadoop/etc/hadoop/core-site.xml
cp etc/hadoop/mapred-site.xml.template etc/hadoop/mapred-site.xml
subl /usr/local/hadoop/etc/hadoop/mapred-site.xml
subl /usr/local/hadoop/etc/hadoop/hdfs-site.xml
subl /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

i. /usr/local/hadoop/etc/hadoop/hadoop-env.sh

ii. /usr/local/hadoop/etc/hadoop/core-site.xml

iii. /usr/local/hadoop/etc/hadoop/mapred-site.xml

iv. /usr/local/hadoop/etc/hadoop/hdfs-site.xml

v. /usr/local/hadoop/etc/hadoop/slaves # only need for multi-node hadoop

a. hadoop-env.sh
replace the line `JAVA_HOME={JAVA_HOME}` with your java root, in my case, it is
`export JAVA_HOME=/usr/lib/jvm/java-8-oracle`.

b. core-site.xml
put the following content to the file.

``` xml
<configuration>
 <property>
  <name>hadoop.tmp.dir</name>
  <value>/usr/local/hadoop/tmp</value>
  <description>
    A base for other temporary directories.
  </description>
 </property>
 <property>
  <name>fs.default.name</name>
  <value>hdfs://localhost:9000</value>
  <!-- <value>hdfs://master:9000</value> for the multi-node case -->
  <description>
    The name of the default file system.  A URI whose
    scheme and authority determine the FileSystem implementation.  The
    uri's scheme determines the config property (fs.SCHEME.impl) naming
    the FileSystem implementation class.  The uri's authority is used to
    determine the host, port, etc. for a filesystem.
  </description>
 </property>
</configuration>
```

c. mapred-site.xml
There is no file `mapred-site.xml`, we get by copy the `mapred-site.xml.template`. This command would be helpful: `cp etc/hadoop/mapred-site.xml.template etc/hadoop/mapred-site.xml && subl etc/hadoop/mapred-site.xml`. We set the specification of job tracker like this:

``` xml
<configuration>
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
</configuration>
```

d. hdfs-site.xml

``` xml
<configuration>
<property>
  <name>dfs.replication</name>
  <value>3</value>
  <description>
    Default block replication. The actual number of replications can be specified when the file is created.   The default is used if replication is not specified in create time.
  </description>
</property>
<property>
  <name>dfs.datanode.data.dir</name>
  <value>/usr/local/hadoop/tmp/data</value>
</property>
<property>
  <name>dfs.namenode.name.dir</name>
  <value>/usr/local/hadoop/tmp/name</value>
</property>
</configuration>
```

e. yarn-site.xml

``` xml
<configuration>
<!-- <property>
     <name>yarn.resourcemanager.hostname</name>
     <value>master</value>
</property> -->
<!-- for multi-node -->
<property>
     <name>yarn.nodemanager.aux-services</name>
     <value>mapreduce_shuffle</value>
</property>
<property>
     <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
     <value>org.apache.hadoop.mapred.ShuffleHandler</value>
</property>
</configuration>
```

f. slaves (for multi-node.)

put names of your machines in the `hadoop/etc/hadoop/slaves`. Command: `subl etc/hadoop/slaves`. The file looks like:

``` bash
master
slave01
```

6. Starting hadoop

``` bash
# import environment variable
# ubuntu: source ~/.bashrc
source /etc/bash.bashrc
# format hadoop space
hdfs namenode -format
# start the hadoop
start-dfs.sh && start-yarn.sh
# or start-all.sh
```

The output looks like this: (standalone)

``` bash
localhost: starting namenode, logging to /usr/local/hadoop/logs/hadoop-master-namenode-master-virtual-machine.out
localhost: starting datanode, logging to /usr/local/hadoop/logs/hadoop-master-datanode-master-virtual-machine.out
Starting secondary namenodes [0.0.0.0]
0.0.0.0: starting secondarynamenode, logging to /usr/local/hadoop/logs/hadoop-master-secondarynamenode-master-virtual-machine.out
starting yarn daemons
starting resourcemanager, logging to /usr/local/hadoop/logs/yarn-master-resourcemanager-master-virtual-machine.out
localhost: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-master-nodemanager-master-virtual-machine.out
```

The output looks like this: (standalone)

``` bash
master: starting namenode, logging to /usr/local/hadoop/logs/hadoop-master-namenode-master.out
slave01: starting datanode, logging to /usr/local/hadoop/logs/hadoop-master-datanode-slave01.out
Starting secondary namenodes [0.0.0.0]
0.0.0.0: starting secondarynamenode, logging to /usr/local/hadoop/logs/hadoop-master-secondarynamenode-master.out
starting yarn daemons
starting resourcemanager, logging to /usr/local/hadoop/logs/yarn-master-resourcemanager-master.out
slave01: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-master-nodemanager-slave01.out
```

check whether the server starts by connecting the local server.

``` bash
# for standalone
firefox http:\\localhost:50070
firefox http:\\localhost:50090
# for multi-node hadoop
hdfs dfsadmin -report
```

Run a example in the folder

``` bash
hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar pi 10 100
```

The last two line will show the following informations:

``` bash
Job Finished in 167.153 seconds
Estimated value of Pi is 3.14800000000000000000
```

Run second example

8. run wordaccout

``` bash
cd ~/Downloads && mkdir testData && cd testData
# download data for test
wget http://www.gutenberg.org/ebooks/5000.txt.utf-8
cd ..
# upload to the hadoop server
hdfs dfs -copyFromLocal testData/ /user/celest/
# check that the file is in the hadoop server
hdfs dfs -ls /user/celest/testData/
# run wordcount on hadoop
hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar wordcount /user/celest/testData /user/celest/testData-output
# check the whether it successes
hdfs dfs -ls /user/celest/testData-output/
# view the result
hdfs dfs -cat /user/celest/testData-output/part-r-00000

# clean the test file (optional)
hdfs dfs -rm -r /user/celest/testData
hdfs dfs -rm -r /user/celest/testData-output
```

8. Stopping hadoop
`stop-all.sh` or `stop-dfs.sh && stop-yarn.sh`


9. compiling hadoop library by yourself (optional)

To avoid the warning `WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable`, you can build the 64 bit hadoop lib by yourself.

Here is the bash script to compile:

``` bash
# install the necessary packages
sudo apt-get -y install build-essential protobuf-compiler autoconf automake libtool cmake zlib1g-dev pkg-config libssl-dev git subversion
cd ~/Downloads
wget https://protobuf.googlecode.com/files/protobuf-2.5.0.tar.gz
tar zxvf protobuf-2.5.0.tar.gz
cd protobuf-2.5.0
./configure
make
sudo make install
# get ant
cd ..
wget http://apache.stu.edu.tw/ant/binaries/apache-ant-1.9.4-bin.tar.gz
tar zxvf apache-ant-1.9.4-bin.tar.gz
sudo mv apache-ant-1.9.4 /usr/local/ant
sudo chown -R celest /usr/local/ant
# get maven
cd ..
wget http://apache.stu.edu.tw/maven/maven-3/3.3.1/binaries/apache-maven-3.3.1-bin.tar.gz
tar zxvf apache-maven-3.3.1-bin.tar.gz
sudo mv apache-maven-3.3.1 /usr/local/maven
sudo chown -R celest /usr/local/maven
sudo subl /etc/bash.bashrc
# add following 5 lines into file
# export MAVEN_HOME=/usr/local/maven
# export PATH=$PATH:$MAVEN_HOME/bin
# export ANT_HOME=/usr/local/ant
# export PATH=$PATH:$ANT_HOME/bin
# export MAVEN_OPTS="-Xms256m -Xmx512m"
source /etc/bash.bashrc

# get the source code of hadoop
cd ..
wget http://apache.stu.edu.tw/hadoop/common/stable2/hadoop-2.7.2-src.tar.gz
tar zxvf hadoop-2.7.2-src.tar.gz
cd hadoop-2.7.2-src
mvn clean package -Pdist -Dtar -Dmaven.javadoc.skip=true -DskipTests -fail-at-end -Pnative
sudo cp -r hadoop-2.7.2-src/hadoop-dist/target/hadoop-2.7.2 /usr/local/hadoop
sudo mv /usr/local/hadoop-2.7.2 /usr/local/hadoop
sudo chown -R celest hadoop
```

