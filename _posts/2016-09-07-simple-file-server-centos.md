---
layout: post
title: "simple file server in centos"
---

這篇主要是講在centos做一個簡單的file server

``` bash
# 安裝httpd
sudo yum install httpd

# create sharing folder
sudo mkdir /var/www/html/share
sudo chown tester:tester /var/www/html/share

# link to ~/share
ln -s /var/www/html/share /home/tester/share

# 啟動
sudo service httpd start  
```

接下來只要把檔案放進`/home/tester/html/share`

然後連到那台電腦的IP下面的share就可以看到file server了

像我電腦就是連到`192.168.0.121:80/share`

