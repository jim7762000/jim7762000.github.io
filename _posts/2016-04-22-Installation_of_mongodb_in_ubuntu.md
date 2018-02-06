---
layout: post
title: "Installation of mongodb in ubuntu"
---

mongodb is a noSQL database. I use it to construct the vd database.

``` bash
# install mongodb
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
echo "deb http://repo.mongodb.org/apt/ubuntu trusty/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
sudo apt-get update
sudo apt-get install -y mongodb-org
# test whether mongodb can be implemented
sudo service mongod start
# # to new file 'mongod.service' in /lib/systemd/system with content:
# [Unit]
# Description=High-performance, schema-free document-oriented database
# Documentation=man:mongod(1)
# After=network.target
#
# [Service]
# Type=forking
# User=mongodb
# Group=mongodb
# RuntimeDirectory=mongod
# PIDFile=/var/run/mongod/mongod.pid
# ExecStart=/usr/bin/mongod -f /etc/mongod.conf --pidfilepath /var/run/mongod/mongod.pid --fork
# TimeoutStopSec=5
# KillMode=mixed
#
# [Install]
# WantedBy=multi-user.target
# # Reference:
# http://askubuntu.com/questions/690993/mongodb-3-0-2-wont-start-after-upgrading-to-ubuntu-15-10

# port to mongodb
iptables -A INPUT -p tcp --dport 27017 -j ACCEPT

# check whether it success
cat /var/log/mongodb/mongod.log
```


use `subl /etc/mongod.conf` to edit the configuration of mongodb. The default file:

``` bash
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
#  engine:
#  mmapv1:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1


#processManagement:

#security:

#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options:

#auditLog:

#snmp:

```

After some edits:

``` bash
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
#  engine:
#  mmapv1:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1
## for remote connection, bind_ip need to be set 0.0.0.0.

#processManagement:

#security:
security:
   authorization: enable
#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options:

#auditLog:

#snmp:


```

For security, to add admin user and create users:

``` bash
use admin
# the user managing users
db.createUser(
  {
    user: "adminname",
    pwd: "password",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
# dbOwner
# read/write all databases user
db.createUser(
  {
    user: "adminname",
    pwd: "password",
    roles: [ { role: "dbOwner", db: "admin" } ]
  }
)
# read/write all databases user
db.createUser(
  {
    user: "adminname",
    pwd: "password",
    roles: [ { role: "readWriteAnyDatabase", db: "admin" } ]
  }
)
# general users
db.createUser(
    {
      user: "adminname",
      pwd: "password",
      roles: [
         { role: "read", db: "reporting" },
         { role: "read", db: "products" },
         { role: "read", db: "sales" },
         { role: "readWrite", db: "accounts" }
      ]
    }
)
```
