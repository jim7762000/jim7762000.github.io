---
layout: post
title: "使用Ambari server與HDP repo快速部署Hadoop Cluster"
---

本篇會從頭開始介紹怎麼使用Ambari server跟HDP repository來快速部署Hadoop Cluster

0. 安裝CentOS以及基本部署

我使用VMware建立三台CentOS 7的電腦

三台的hostname分別為`ambaritest01`, `ambaritest02` and `ambaritest03`

此處建議都用`root`帳號安裝，不然ambari有一個地方要額外設定

a. 網路

ambaritest01的網路設定(/etc/sysconfig/ifcfg-XXXXXX, XXXXXX是網路卡的名稱)：

``` 
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=faa2688c-6d77-4e38-8c71-d65c67823dd5
DEVICE=ens33
ONBOOT=yes
DNS1=192.168.0.1
DNS2=8.8.8.8
IPADDR=192.168.0.121
PREFIX=24
GATEWAY=192.168.0.1
```

`ambaritest02`跟`ambaritest03`的網路設定只需要改IPADDR即可

b. 對時

下面直接使用網路對時，如果是local LAN請在一台建立可以對時的server (建立方法請google)

然後修改`/etc/ntp.conf`讓全部機器都去自動與那台對時

``` bash
yum install -y ntp ntpdate ntp-doc
ntpdate pool.ntp.org
systemctl start ntpd
systemctl enable ntpd
```

對時很重要，有時候Hadoop的問題就來自時間的不一致

c. 關閉SELinux

``` bash
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/sysconfig/selinux
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
setenforce 0
```

d. 關閉防火牆

``` bash
systemctl stop firewalld
systemctl disable firewalld
```

e. enable SSH連線

每一台都要先跑下面命令：

``` bash
ssh-keygen -t rsa -P ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 700 ~/
chmod 700 ~/.ssh
chmod 644 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
systemctl restart sshd 

tee -a /etc/hosts << "EOF"
192.168.0.121 ambaritest01
192.168.0.122 ambaritest02
192.168.0.123 ambaritest03
EOF
```

在你要安裝ambari server那台執行，假設安裝在ambaritest01：

``` bash
ssh-copy-id -i ~/.ssh/id_rsa.pub ambaritest01
ssh-copy-id -i ~/.ssh/id_rsa.pub ambaritest02
ssh-copy-id -i ~/.ssh/id_rsa.pub ambaritest03
```

然後測試一下，確定都可以用`ssh`直接相連

f. 安裝Oracle Java

個人不愛用openjdk，會遇到一些奇怪的bug，所以我都安裝Oracle JAVA

這裡`JAVA_HOME`很重要，後面會用到

```
curl -v -j -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u112-b15/jdk-8u112-linux-x64.rpm -o jdk-8u112-linux-x64.rpm
yum install -y jdk-8u112-linux-x64.rpm

scp jdk-8u112-linux-x64.rpm ambaritest02:~/
ssh ambaritest02 yum install -y jdk-8u112-linux-x64.rpm
scp jdk-8u112-linux-x64.rpm ambaritest03:~/
ssh ambaritest03 yum install -y jdk-8u112-linux-x64.rpm

tee -a /etc/bashrc << "EOF"
export JAVA_HOME=/usr/java/jdk1.8.0_112
export PATH=$PATH:$JAVA_HOME/bin
EOF
scp /etc/bashrc ambaritest02:/etc
scp /etc/bashrc ambaritest03:/etc
source /etc/bashrc
```

h. 安裝gcc-5.3, R(非必要)

因為我自己會用到R，之後也想要在Apache Zeppline上使用，所以順便一些紀錄

```
# install R from EPEL (default BLAS is openblas)
yum install wget
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install -y epel-release-latest-7.noarch.rpm
yum install -y R R-devel R-java libxml2-devel libxml2-static tcl tcl-devel tk tk-devel libtiff-static libtiff-devel libjpeg-turbo-devel libpng12-devel cairo-tools libicu-devel openssl-devel libcurl-devel freeglut readline-static readline-devel cyrus-sasl-devel

# install microsoft R open (default BLAS is MKL)
wget https://mran.microsoft.com/install/mro/3.3.2/microsoft-r-open-3.3.2.tar.gz
tar zxvf microsoft-r-open-3.3.2.tar.gz
yum install -y microsoft-r-open/rpm/microsoft-r-open-*

# remove R from EPEL, and use microsoft R open
rm -rf /usr/lib64/R
cp -r /usr/lib64/microsoft-r/3.3/lib64/R /usr/lib64
# let user access library dir (not to use personal library)
chmod -R 777 /usr/lib64/R/library

# change default repos
tee -a /usr/lib64/R/etc/Rprofile.site << EOF
options(repos = "https://cloud.r-project.org/")
EOF

# remove openjdk
yum remove -y openjdk-*

# config R-Java
R CMD javareconf
Rscript -e "install.packages('rJava')"
# enable C++11 for microsoft R open
sudo sed -i -e 's/CXX1X =/CXX1X = g++/g' /usr/lib64/R/etc/Makeconf
sudo sed -i -e 's/CXX1XFLAGS =/CXX1XFLAGS = -DU_STATIC_IMPLEMENTATIN -O2 -g/g' /usr/lib64/R/etc/Makeconf
sudo sed -i -e 's/CXX1XPICFLAGS =/CXX1XPICFLAGS = -fpic/g' /usr/lib64/R/etc/Makeconf
sudo sed -i -e 's/CXX1XSTD =/CXX1XSTD = -std=c++11/g' /usr/lib64/R/etc/Makeconf

# install rstudio server
wget https://download2.rstudio.org/rstudio-server-rhel-1.0.136-x86_64.rpm
sudo yum install -y --nogpgcheck rstudio-server-rhel-1.0.136-x86_64.rpm

# let user not to use personal library
sudo tee -a /etc/rstudio/rsession.conf << EOF
r-libs-user=/usr/lib64/R/library
EOF

# start rstudio-server
sudo systemctl enable rstudio-server
sudo systemctl start rstudio-server

# user for rstudio server
useradd rstudio
passwd rstudio
```

1. install ambari-server

每台都要新增repo資料:

``` bash
wget http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.4.2.0/ambari.repo -O /etc/yum.repos.d/ambari.repo
wget http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.5.3.0/hdp.repo -O /etc/yum.repos.d/hdp.repo
```

在ambaritest01安裝ambari-server:

``` bash
yum install ambari-server
```

2. start ambari-server

先跑`ambari-server setup`，基本上選不用Customize

然後選Custom JDK，JAVA_HOME用上面設定的`/usr/java/jdk1.8.0_112`

再使用`ambari-server start`啟動即可

也要記得使用`systemctl enable ambari-server`讓電腦自動開啟ambari-server

3. Install, configure and deploy an HDP cluster

在我電腦用瀏覽器登入`http://192.168.0.121:8080`就可以看到登入畫面，預設帳密為admin/admin

登入之後就可以看到下面的畫面：

![](/images/ambari-setup.PNG)

接下來就是按下`Launch Install Wizard`開始安裝就好

步驟基本上照著[官網手冊](https://docs.hortonworks.com/HDPDocuments/Ambari-2.4.1.0/bk_ambari-installation/content/log_in_to_apache_ambari.html)手就好

我只說明第五步，`Target Hosts`寫`ambaritest[01-03]`，`Host Registration Information`部分則用`cat ~/.ssh/ id_rsa`印出的資訊

接著第六步就會開始安裝ambari-agent，然後安裝完還會跑一個check，安裝以及check成功的畫面如下：

![](/images/ambari-install.PNG)

再下一頁就是選擇服務安裝了，其中Log Search跟SmartSense是Hortonworks的軟體，是要授權碼的，其他都是apache license

![](/images/ambari-choose-service.PNG)

license相關資訊請查詢[這裡](http://hortonworks.com/licenses/)

成功之後就可以看到cluster畫面： (我有些service沒裝成功，可能還要看一下原因)

![](/images/ambari-success.PNG)
