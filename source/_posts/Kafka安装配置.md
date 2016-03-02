---
title: Kafka安装配置
date: 2016-03-02 11:04:50
tags: 大数据
---
我们使用3台机器搭建Kafka集群：
```java
192.168.4.142   h1
192.168.4.143   h2
192.168.4.144   h3

```
在安装Kafka集群之前，这里没有使用Kafka自带的Zookeeper，而是独立安装了一个Zookeeper集群，也是使用这3台机器，保证Zookeeper集群正常运行。
首先，在h1上准备Kafka安装文件，执行如下命令：
```java
cd /usr/local/
wget http://mirror.bit.edu.cn/apache/kafka/0.8.1.1/kafka_2.9.2-0.8.1.1.tgz
tar xvzf kafka_2.9.2-0.8.1.1.tgz
ln -s /usr/local/kafka_2.9.2-0.8.1.1 /usr/local/kafka
chown -R kafka:kafka /usr/local/kafka_2.9.2-0.8.1.1 /usr/local/kafka
```
修改配置文件/usr/local/kafka/config/server.properties，修改如下内容：
```java
broker.id=0
zookeeper.connect=h1:2181,h2:2181,h3:2181/kafka
```

这里需要说明的是，默认Kafka会使用ZooKeeper默认的/路径，这样有关Kafka的ZooKeeper配置就会散落在根路径下面，如果你有其他的应用也在使用ZooKeeper集群，查看ZooKeeper中数据可能会不直观，所以强烈建议指定一个chroot路径，直接在zookeeper.connect配置项中指定：
```java
zookeeper.connect=h1:2181,h2:2181,h3:2181/kafka
```

而且，需要手动在ZooKeeper中创建路径/kafka，使用如下命令连接到任意一台ZooKeeper服务器：
```java
cd /usr/local/zookeeper
bin/zkCli.sh
```

在ZooKeeper执行如下命令创建chroot路径：
```java
create /kafka ''
```
这样，每次连接Kafka集群的时候（使用--zookeeper选项），也必须使用带chroot路径的连接字符串，后面会看到。
然后，将配置好的安装文件同步到其他的h2、h3节点上：
```java
scp -r /usr/local/kafka_2.9.2-0.8.1.1/ h2:/usr/local/
scp -r /usr/local/kafka_2.9.2-0.8.1.1/ h3:/usr/local/
```

最后，在h2、h3节点上配置，执行如下命令：
```java
cd /usr/local/
ln -s /usr/local/kafka_2.9.2-0.8.1.1 /usr/local/kafka
chown -R kafka:kafka /usr/local/kafka_2.9.2-0.8.1.1 /usr/local/kafka
```

并修改配置文件/usr/local/kafka/config/server.properties内容如下所示：
```java
broker.id=1  # 在h1修改
broker.id=2  # 在h2修改
```

因为Kafka集群需要保证各个Broker的id在整个集群中必须唯一，需要调整这个配置项的值（如果在单机上，可以通过建立多个Broker进程来模拟分布式的Kafka集群，也需要Broker的id唯一，还需要修改一些配置目录的信息）。
在集群中的h1、h2、h3这三个节点上分别启动Kafka，分别执行如下命令：
```java
bin/kafka-server-start.sh /usr/local/kafka/config/server.properties &
```
可以通过查看日志，或者检查进程状态，保证Kafka集群启动成功。

我们创建一个名称为my-replicated-topic5的Topic，5个分区，并且复制因子为3，执行如下命令：
```java
bin/kafka-topics.sh --create --zookeeper h1:2181,h2:2181,h3:2181/kafka /
	--replication-factor 3 --partitions 5 --topic my-replicated-topic5
```

查看创建的Topic，执行如下命令：
```java
bin/kafka-topics.sh --describe --zookeeper h1:2181,h2:2181,h3:2181/kafka /
	--topic my-replicated-topic5
```

上面Leader、Replicas、Isr的含义如下：
```java
Partition： 分区
Leader   ： 负责读写指定分区的节点
Replicas ： 复制该分区log的节点列表
Isr      ： "in-sync" replicas，当前活跃的副本列表（是一个子集），并且可能成为Leader
```
我们可以通过Kafka自带的bin/kafka-console-producer.sh和bin/kafka-console-consumer.sh脚本，来验证演示如果发布消息、消费消息。
在一个终端，启动Producer，并向我们上面创建的名称为my-replicated-topic5的Topic中生产消息，执行如下脚本：
```java
bin/kafka-console-producer.sh --broker-list h1:9092,h2:9092,h3:9092 /
	--topic my-replicated-topic5
```

在另一个终端，启动Consumer，并订阅我们上面创建的名称为my-replicated-topic5的Topic中生产的消息，执行如下脚本：
```java
bin/kafka-console-consumer.sh --zookeeper h1:2181,h2:2181,h3:2181/kafka /
	--from-beginning --topic my-replicated-topic5
```
可以在Producer终端上输入字符串消息行，然后回车，就可以在Consumer终端上看到消费者消费的消息内容。也可以参考Kafka的Producer和Consumer的Java API，通过API编码的方式来实现消息生产和消费的处理逻辑。