---
title: cloudera-scm-server dead but pid file exists
date: 2016-03-14 13:46:34
tags: 大数据
---

问题: 系统磁盘或者内存问题导致cloudera server挂掉，启动异常。进程已死，pid文件存在。
```bash
# service cloudera-scm-server start
$> Starting cloudera-scm-server:                              [  OK  ]
# service cloudera-scm-server status
$> cloudera-scm-server dead but pid file exists
```

解决办法:
```basj
$> service cloudera-scm-server stop
$> service cloudera-scm-server-db stop 
$> rm /var/run/cloudera-scm-server.pid
$> service cloudera-scm-server-db start
$> service cloudera-scm-server start
```