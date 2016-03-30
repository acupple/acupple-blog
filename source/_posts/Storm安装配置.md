---
title: Storm安装配置
date: 2016-03-02 11:27:46
tags: 大数据
---
我们使用3台机器搭建Storm集群:
```bash
192.168.4.142   h1
192.168.4.143   h2
192.168.4.144   h3
```
首先要保证zookeeper集群正常运行，假设zk也同样部署在h1, h2, h3机器上，端口为为默认的2181。
然后，在h1节点上，执行如下命令安装：
```bash
$> cd /usr/local/
$> wget http://mirror.bit.edu.cn/apache/incubator/storm/apache-storm-0.9.2-incubating/apache-storm-0.9.2-incubating.tar.gz
$> tar xvzf apache-storm-0.9.2-incubating.tar.gz
$> ln -s /usr/local/apache-storm-0.9.2-incubating /usr/local/storm
$> chown -R storm:storm /usr/local/apache-storm-0.9.2-incubating /usr/local/storm
```
然后，修改配置文件conf/storm.yaml，内容如下所示：
```bash
 storm.zookeeper.servers:
     - "h1"
     - "h2"
     - "h3"
storm.zookeeper.port: 2181
nimbus.host: "h1"

supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703

storm.local.dir: "/tmp/storm"
```

将配置好的安装文件，分发到其他节点上：
```bash
$> scp -r /usr/local/apache-storm-0.9.2-incubating/ h2:/usr/local/
$> scp -r /usr/local/apache-storm-0.9.2-incubating/ h3:/usr/local/
```
最后，在h2、h3节点上配置，执行如下命令：
```bash
$> cd /usr/local/
$> ln -s /usr/local/apache-storm-0.9.2-incubating /usr/local/storm
$> chown -R storm:storm /usr/local/apache-storm-0.9.2-incubating /usr/local/storm
```

Storm集群的主节点为Nimbus，从节点为Supervisor，我们需要在h1上启动Nimbus服务，在从节点h2、h3上启动Supervisor服务：
```bash
$> nohup bin/storm nimbus > /var/storm/nimbus.log &
$> bin/storm supervisor > /var/storm/supervisor.log &
```

为了方便监控，可以启动Storm UI，可以从Web页面上监控Storm Topology的运行状态，例如在h2上启动:
```bash
$> bin/storm ui > /var/storm/ui.log &
```
这样可以通过访问http://h2:8080/ 来查看Topology的运行状况。