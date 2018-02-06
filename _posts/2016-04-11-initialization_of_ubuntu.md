---
layout: post
title: Initialization of ubuntu
---

``` bash
sudo add-apt-repository ppa:webupd8team/sublime-text-3
sudo apt-get update
sudo apt-get install sublime-text-installer
sudo add-apt-repository ppa:webupd8team/java && sudo apt-get update && sudo apt-get install oracle-java8-installer && sudo apt-get install oracle-java8-set-default
apt-cache search readline xorg-dev && sudo apt-get install libreadline6 libreadline6-dev xorg-dev tcl8.6-dev tk8.6-dev libtiff5 libtiff5-dev libjpeg-dev libpng12-dev libcairo2-dev libglu1-mesa-dev libgsl0-dev libicu-dev R-base R-base-dev libnlopt-dev libstdc++6 build-essential libcurl4-openssl-dev libxml2-dev aptitude r-base r-base-dev libnlopt-dev libstdc++6 build-essential libcurl4-openssl-dev libxml2-dev libssl-dev

# installation of rstudio-server
sudo apt-get install gdebi-core
wget https://download2.rstudio.org/rstudio-server-0.99.896-amd64.deb
sudo gdebi rstudio-server-0.99.896-amd64.deb
sudo cp /usr/lib/rstudio-server/extras/init.d/debian/rstudio-server /etc/init.d/
sudo apt-get install sysv-rc-conf
sudo sysv-rc-conf rstudio-server on
# check whether rstudio-server open when boot
sysv-rc-conf --list rstudio-server
# port to rstudio-server
iptables -A INPUT -p tcp --dport 8787 -j ACCEPT

# remove R
rm -r /usr/local/lib/R

# download the installation file
wget https://mran.revolutionanalytics.com/install/mro/3.2.4/MRO-3.2.4-Ubuntu-15.4.x86_64.deb
wget https://mran.revolutionanalytics.com/install/mro/3.2.4/RevoMath-3.2.4.tar.gz
# install MRO
sudo dpkg -i MRO-3.2.4-Ubuntu-15.4.x86_64.deb
# unpack MKL
tar -xzf RevoMath-3.2.4.tar.gz
cd RevoMath
# install MKL
sudo bash ./RevoMath.sh

sudo chown -R celest.celest /usr/lib64/MRO-3.2.4/R-3.2.4/lib/R
sudo chmod -R 775 /usr/lib64/MRO-3.2.4/R-3.2.4/lib/R

# install texlive
sudo apt-get install texinfo texlive texlive-binaries texlive-latex-base texlive-latex-extra texlive-fonts-extra

# for server
sudo apt-get install ssh rsync openssh-server
ssh-keygen -t rsa -P "" # generate SSH key
# Enable SSH Key
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# for VM, to use unity mode
sudo apt-get install gnome-shell
```
