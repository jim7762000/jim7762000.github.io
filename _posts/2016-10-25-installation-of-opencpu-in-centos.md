---
layout: post
title: "Installation of OpenCPU in centos"
---

上次只是用R的`opencpu`套件小試一下

這次就直接在server上建立opencpu server

讓local的R可以去call server服務

``` bash
# install Microsoft R Open
su -c 'rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm'
sudo yum update
sudo yum install gcc-c++ R R-java libapreq2-devel R-devel libcurl-devel protobuf-devel openssl-devel libxml2-devel libicu-devel libssh2-devel

## remove R
sudo rm -rf /usr/lib64/R

# install Microsoft R Open
curl -v -j -k -L https://mran.microsoft.com/install/mro/3.3.1/microsoft-r-open-3.3.1.tar.gz -o microsoft-r-open-3.3.1.tar.gz
tar zxvf microsoft-r-open-3.3.1.tar.gz
sudo yum install -y microsoft-r-open/rpm/*
sudo chmod -R 777 /usr/lib64/microsoft-r/3.3/lib64/R 

# to use Microsoft R Open with opencpu
sudo cp -r /usr/lib64/microsoft-r/3.3/lib64/R /usr/lib64

# install git
sudo yum install git

# install opencpu
git clone https://github.com/jeroenooms/opencpu-server.git

chmod +rx ~/
chmod +x opencpu-server/rpm/*.sh
opencpu-server/rpm/buildscript.sh

# 最後會在rpmbuild/RPMS/x86_64/看到編譯好的rpm
```

最後使用下面指令開port就可以順利在網頁上登入`http://<your_ip_address>/ocpu/test`

``` bash
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo service iptables save
```
