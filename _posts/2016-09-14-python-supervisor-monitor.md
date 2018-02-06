---
layout: post
title: "使用python的套件supervisor監控程式 - 以Cassandra, Spark, Mesos為例"
---

Python的supervisor是一套簡單、輕量的監控系統服務之工具

透過簡單的安裝跟些許的設定即可以達到想要的效果

先安裝supervisor:

``` bash
sudo yum install python-setuptools
sudo easy_install pip
sudo pip install supervisor
# echo default config
sudo mkdir /etc/supervisor
sudo mkdir /var/log/supervisor
sudo bash -c 'echo_supervisord_conf > /etc/supervisor/supervisord.conf'
```

使用`sudo vi /etc/supervisor/supervisord.conf`編輯，更動下面的設定(記得針對每一台的IP要做更改)：

```
[inet_http_server]         ; inet (TCP) server disabled by default
port=192.168.0.121:10088   ; (ip_address:port specifier, *:port for all iface)
username=tester            ; (default is no username (open server))
password=qscf12356         ; (default is no password (open server))

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
serverurl=http://192.168.0.121:10088 ; use an http:// url to specify an inet socket
username=tester              ; should be same as http_username if set
password=qscf12356          ; should be same as http_password if set

[supervisord]
environment=
  JAVA_HOME=/usr/java/jdk1.8.0_101,
  SPARK_HOME=/usr/local/bigdata/spark,
  ZOOKEEPER_HOME=/usr/local/bigdata/zookeeper,
  CASSANDRA_HOME=/usr/local/bigdata/cassandra
```

如果之前有設定過自動啟動，先關掉：

``` bash
sudo systemctl stop mesos-master
sudo systemctl stop mesos-slave
sudo systemctl disable mesos-master
sudo systemctl disable mesos-slave

sudo service cassandra stop
sudo service zookeeper stop
sudo service spark-shuffle stop

sudo chkconfig --del cassandra
sudo chkconfig --del zookeeper
sudo chkconfig --del spark-shuffle 

sudo rm /etc/init.d/cassandra
sudo rm /etc/init.d/spark-shuffle   
sudo rm /etc/init.d/zookeeper
```

最後是使用下面的script去配置supervisord：

``` bash
sudo tee -a /etc/supervisor/supervisord.conf << "EOF"

[program:cassandra]
command=/bin/bash -c "$CASSANDRA_HOME/bin/cassandra -f"
stdout_logfile=/var/log/supervisor/cassandra-stdout.out
stderr_logfile=/var/log/supervisor/cassandra-stderr.out
autostart=true
startsecs=5
priority=50

[program:mesos-slave]
command=/bin/bash -c "/usr/bin/mesos-init-wrapper slave"
stdout_logfile=/var/log/supervisor/mesos-slave-stdout.out
stderr_logfile=/var/log/supervisor/mesos-slave-stderr.out
autostart=true
startsecs=5
priority=80

[program:spark-mesos-shuffle]
command=/bin/bash -c "$SPARK_HOME/bin/spark-submit --class org.apache.spark.deploy.mesos.MesosExternalShuffleService 1"
stdout_logfile=/var/log/supervisor/spark-mesos-shuffle-stdout.out
stderr_logfile=/var/log/supervisor/spark-mesos-shuffle-stderr.out
autostart=true
startsecs=5
priority=90

; 下面只有需要的node才要配置
[program:zookeeper]
command=/bin/bash -c "$ZOOKEEPER_HOME/bin/zkServer.sh start-foreground"
stdout_logfile=/var/log/supervisor/zookeeper-stdout.out
stderr_logfile=/var/log/supervisor/zookeeper-stderr.out
autostart=true
startsecs=10
priority=50

[program:mesos-master]
command=/bin/bash -c "/usr/bin/mesos-init-wrapper master"
stdout_logfile=/var/log/supervisor/mesos-master-stdout.out
stderr_logfile=/var/log/supervisor/mesos-master-stderr.out
autostart=true
startsecs=5
priority=60
EOF
```

再來是配置supervisor的自動啟動服務

``` bash
sudo tee -a /usr/lib/systemd/system/supervisor.service << "EOF"
[Unit]
Description=supervisor
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisor/supervisord.conf
ExecStop=/usr/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/bin/supervisorctl $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start supervisor.service
sudo systemctl enable supervisor.service
```

最後使用設定的IP跟PORT，連上該網址，像是我的設定是連`http://192.168.0.121:10088`

就可以看到可以手動操作的部分了