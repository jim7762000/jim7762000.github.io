---
layout: post
title: "Some installations of centos"
---

``` bash
# update system
sudo yml update

# installation of sublime text
cd ~/Downloads
wget https://download.sublimetext.com/sublime_text_3_build_3103_x64.tar.bz2
sudo tar -vxjf sublime_text_3_build_3103_x64.tar.bz2 -C /opt
## make a symbolic link to the installed Sublime3
sudo ln -s /opt/sublime_text_3/sublime_text /usr/bin/subl
## create Gnome desktop launcher
sudo subl /usr/share/applications/subl.desktop
## add following lines into file
# [Desktop Entry]
# Name=Subl
# Exec=subl
# Terminal=false
# Icon=/opt/sublime_text_3/Icon/48x48/sublime-text.png
# Type=Application
# Categories=TextEditor;IDE;Development
# X-Ayatana-Desktop-Shortcuts=NewWindow
#
# [NewWindow Shortcut Group]
# Name=New Window
# Exec=subl -n
# TargetEnvironment=Unity

# installation of java 8
sudo wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u66-b17/jdk-8u66-linux-x64.rpm"
sudo rpm -ivh jdk-8u66-linux-x64.rpm

## Setup JAVA Environment Variables
sudo nano ~/.bashrc
## add following lines into file
# JAVA_HOME="/usr/java/jdk1.8.0_66/bin/java"
# JRE_HOME="/usr/java/jdk1.8.0_66/jre/bin/java"
# PATH=$PATH:$HOME/bin:JAVA_HOME:JRE_HOME
source ~/.bashrc

# installation of required packages for building R
sudo yum install libxml2-devel libxml2-static tcl tcl-devel tk tk-devel libtiff-static libtiff-devel
libjpeg-turbo-devel libpng12-devel cairo-tools libicu-devel openssl-devel libcurl-devel freeglut
readline-static readline-devel cyrus-sasl-devel texlive texlive-xetex

# install R
su -c 'rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm'
sudo yum update
sudo yum install R R-devel R-java

## remove R
sudo rm -rf /usr/lib64/R

# install MRO
## Make sure the system repositories are up-to-date prior to installing Microsoft R Open.
sudo yum clean all
## get the installers
wget https://mran.microsoft.com/install/mro/3.2.4/MRO-3.2.4.el7.x86_64.rpm
wget https://mran.microsoft.com/install/mro/3.2.4/RevoMath-3.2.4.tar.gz
## install MRO
sudo yum install MRO-3.2.4.el7.x86_64.rpm
## install MKL
tar -xzf RevoMath-3.2.4.tar.gz
cd RevoMath
sudo bash ./RevoMath.sh
### Choose option 1 to install MKL and follow the onscreen prompts.

## change the right of folders makes owner can install packages in library
sudo chown -R celest.celest /usr/lib64/MRO-3.2.4/R-3.2.4/lib64/R
sudo chmod -R 775 /usr/lib64/MRO-3.2.4/R-3.2.4/lib64/R

# for ssh connection
sudo apt-get install rsync openssh-server-sysvinit
ssh-keygen -t rsa -P "" # generate SSH key
# Enable SSH Key
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

The installation of rstudio server is recorded:

``` bash
wget https://download2.rstudio.org/rstudio-server-rhel-0.99.896-x86_64.rpm
sudo yum install --nogpgcheck rstudio-server-rhel-0.99.896-x86_64.rpm
## start the rstudio-server
sudo rstudio-server start
## start rstudio-server on boot
sudo cp /usr/lib/rstudio-server/extras/init.d/redhat/rstudio-server /etc/init.d/
sudo chmod 755 /etc/init.d/rstudio-server
sudo chkconfig --add rstudio-server
## open the firewall for rstudio-server
sudo firewall-cmd --zone=public --add-port=8787/tcp --permanent
sudo firewall-cmd --reload

## To browse localhost:8787 for using the rstudio-server
```
The installation of shiny server is recorded:

``` bash
## install shiny-server
wget https://download3.rstudio.org/centos5.9/x86_64/shiny-server-1.4.2.786-rh5-x86_64.rpm
sudo yum install --nogpgcheck shiny-server-1.4.2.786-rh5-x86_64.rpm
## start the shiny-server
sudo systemctl start shiny-server
## start shiny-server on boot
sudo cp /opt/shiny-server/config/init.d/redhat/shiny-server /etc/init.d/
sudo chmod 755 /etc/init.d/shiny-server
sudo chkconfig --add shiny-server

## open the firewall for shiny-server
sudo firewall-cmd --zone=public --add-port=3838/tcp --permanent
sudo firewall-cmd --reload

## the server file is in /srv/shiny-server
## there are some examples, you can browser localhost:3838,
## localhost:3838/sample-apps/hello and localhost:3838/sample-apps/rmd
```

Also, the installation of mongoDB is recorded:

``` bash
## Configure the package management system
sudo subl /etc/yum.repos.d/mongodb-org-3.2.repo
## add following lines into file
# [mongodb-org-3.2]
# name=MongoDB Repository
# baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.2/x86_64/
# gpgcheck=1
# enabled=1
# gpgkey=https://www.mongodb.org/static/pgp/server-3.2.asc
## install mongoDB
sudo yum install -y mongodb-org

## prevent updating version of mongodb
sudo subl /etc/yum.conf
## add following line into file
# exclude=mongodb-org,mongodb-org-server,mongodb-org-shell,mongodb-org-mongos,mongodb-org-tools

## start mongod on boot
sudo chkconfig mongod on

## increase the soft limit of number of process for mongod
sudo subl /etc/security/limits.conf
## add following line into file (third line must be added, other lines are optional.)
# mongod soft nofile 64000
# mongod hard nofile 64000
# mongod soft nproc 32000
# mongod hard nproc 32000

## to disable transparent huge pages
# https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/
sudo subl /etc/init.d/disable-transparent-hugepages
## add following line into file
# #!/bin/sh
# ### BEGIN INIT INFO
# # Provides:          disable-transparent-hugepages
# # Required-Start:    $local_fs
# # Required-Stop:
# # X-Start-Before:    mongod mongodb-mms-automation-agent
# # Default-Start:     2 3 4 5
# # Default-Stop:      0 1 6
# # Short-Description: Disable Linux transparent huge pages
# # Description:       Disable Linux transparent huge pages, to improve
# #                    database performance.
# ### END INIT INFO
#
# case $1 in
#   start)
#     if [ -d /sys/kernel/mm/transparent_hugepage ]; then
#       thp_path=/sys/kernel/mm/transparent_hugepage
#     elif [ -d /sys/kernel/mm/redhat_transparent_hugepage ]; then
#       thp_path=/sys/kernel/mm/redhat_transparent_hugepage
#     else
#       return 0
#     fi
#
#     echo 'never' > ${thp_path}/enabled
#     echo 'never' > ${thp_path}/defrag
#
#     unset thp_path
#     ;;
# esac
## make it executable
sudo chmod 755 /etc/init.d/disable-transparent-hugepages
## run it on boot
sudo chkconfig --add disable-transparent-hugepages

## Give a permissive policy on your /mongo/ directory that permits access to daemons (like the mongod service.).
# http://stackoverflow.com/questions/30182016/unable-to-start-mongodb-3-0-2-service-on-centos-7
# https://www.centos.org/docs/5/html/5.1/Deployment_Guide/sec-sel-enable-disable.html
sudo subl /etc/sysconfig/selinux
# set SELINUX=disabled
```

Automatically enabling your network connection at startup on CentOS 7:

``` bash
# In my case, it is ifcfg-eno1. It may be different for other machine.
sudo sed -i -e 's@^ONBOOT=no@ONBOOT=yes@' /etc/sysconfig/network-scripts/ifcfg-eno1
```

Installation of ftp server on CentOS 7:

``` bash
sudo yum install -y vsftpd

# edit the environment of vsftpd
sudo subl /etc/vsftpd/vsftpd.conf
# stop the anonymous logging
sudo sed -i -e 's@^anonymous_enable=YES@anonymous_enable=NO@' /etc/vsftpd/vsftpd.conf

# start the service and start on boot
sudo service vsftpd start
chkconfig vsftpd on

## open the firewall for xrdp
sudo firewall-cmd --zone=public --add-port=21/tcp --permanent
sudo firewall-cmd --reload
```

Installation of xrdp on CentOS 7 / RHEL 7 (remote desktop from windows):

``` bash
sudo subl /etc/yum.repos.d/xrdp.repo
## add following lines into file
# [xrdp]
# name=xrdp
# baseurl=http://li.nux.ro/download/nux/dextop/el7/x86_64/
# enabled=1
# gpgcheck=0
sudo yum install -y xrdp tigervnc-server

## open the firewall for xrdp
sudo firewall-cmd --zone=public --add-port=3389/tcp --permanent
sudo firewall-cmd --reload
# start service
sudo systemctl start xrdp.service
# enable service on boot
sudo systemctl enable xrdp.service

## Attention: the bpp color must be set to be 24bit. (in windows.)
```

Mount another network disk:

``` bash
sudo mount -t cifs -o username="xxxxxxx",password="yyyyyyy" //ip-address/folder /mnt/folder

# to mount on boot
subl /etc/fstab
## add following lines into file
# //ip-address/folder /mnt/folder username=xxxxxxx,password=yyyyyyy,uid=1000,gid=1000,sec=ntlm,iocharset=utf8 0 0
sudo mount -a
```
