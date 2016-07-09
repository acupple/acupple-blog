---
title: ETCD实现leader选举
date: 2016-05-31 10:43:34
tags: Java
---
客户端的leader选举实现由多种选择，最简单的做法是使用zookeeper，例如[ZK Curator framework](http://curator.apache.org/curator-recipes/leader-election.html)
这里我只简单说明一下我在ETCD中实现leader election的做法

1. 每一个client需要启动的时候需要有一个唯一的ID
2. 需要一个目录来保存leader的信息： 例如： /root/path
3. client判断自己是否为leader的时候：
```bash
#获取etcd上的leader信息
curl http://127.0.0.1:2379/v2/keys/root/path
```
如果返回成功，判断leader是否是自己，例如判断ID是否和自己的ID相同。

如果返回失败，设置/root/path的值，并把TTL设置为60s（具体的根据自己的需要设定）
```bash
curl http://127.0.0.1:2379/v2/keys/root/path -XPUT -d value=$(ID) -d ttl=60
```
这样TTL时间一到，节点就会被删除，所有的client再公平竞争

[ETCD Java client](https://github.com/acupple/jetcd), 基于OkHttp实现，依赖少，避免http client/async httpclient的包版本冲突