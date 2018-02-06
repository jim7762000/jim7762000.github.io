---
layout: post
title: "建置Mongodb Sharding Server"
---

業務擴展到找尋適合的NoSQL server了

這篇來試Mongodb sharding server

基本上都是參考下面幾篇去建置的：

1. [Install MongoDB Community Edition on CentOS Linux](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/)

1. [MongoDB Replica Set in CentOS 6.x](http://cyrilwang.blogspot.tw/2012/06/mongodb-replica-set-in-centos-6x_05.html)

1. [MongoDB Replica Set 高可用性架構搭建](https://blog.toright.com/posts/4508/mongodb-replica-set-%E9%AB%98%E5%8F%AF%E7%94%A8%E6%80%A7%E6%9E%B6%E6%A7%8B%E6%90%AD%E5%BB%BA.html)

1. [MongoDB Sharding 分散式儲存架構建置 (概念篇)](https://blog.toright.com/posts/4552/mongodb-sharding-%E5%88%86%E6%95%A3%E5%BC%8F%E5%84%B2%E5%AD%98%E6%9E%B6%E6%A7%8B%E5%BB%BA%E7%BD%AE-%E6%A6%82%E5%BF%B5%E7%AF%87.html)

1. [MongoDB Sharding 分散式儲存架構建置 (實作篇)](https://blog.toright.com/posts/4574/mongodb-sharding-%E5%88%86%E6%95%A3%E5%BC%8F%E5%84%B2%E5%AD%98%E6%9E%B6%E6%A7%8B%E5%BB%BA%E7%BD%AE-%E5%AF%A6%E4%BD%9C%E7%AF%87.html)

1. [github scripts to create mongodb sharding](https://github.com/Azure/azure-quickstart-templates/blob/master/mongodb-sharding-centos)

我根據我的需求把第6點的scripts改成我要，可以在[我的github](https://github.com/ChingChuan-Chen/create_mongodb_sharding_server)找到

我的伺服器分布規劃：

```
replica set 1: 192.168.0.121 (primary), 192.168.0.122, 192.168.0.123
replica set 2: 192.168.0.124 (primary), 192.168.0.125, 192.168.0.126
config server: 192.168.0.127
router: 192.168.0.128
```

但是為了fit我的分布，我做了一些修改

1. 我刪掉`config_primary.sh`第82行，81行改成`config={_id: "crepset", configsvr: true, members: [{_id: 0, host: "192.168.0.127:27019"}]}`

1. `router1.sh`中的`crepset/192.168.0.127:27019,192.168.0.128:27019,192.168.0.129:2701`都取代成`crepset/192.168.0.127:27019`

1. `config_secondary.sh`跟`router2.sh`就沒跑了

接下來就直接在各台先取得root權限(用`su`)，然後跑`./install_mongodb.sh 該台IP`安裝mongodb

PS: rpms可以在https://repo.mongodb.org/yum/redhat/7Server/mongodb-org/3.2/x86_64/RPMS/找到

接著在`192.168.0.121`跟`192.168.0.124`跑`./replica_primary.sh set名稱 該台IP 使用者名稱`

並同時在replica set secondary的電腦上跑``./replica_secondary.sh`

都完成之後，使用`config_primary.sh`部署config server

最後再用`router1.sh`部署router server

到此，mongodb sharding server就部署完畢了
