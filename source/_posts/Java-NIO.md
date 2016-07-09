---
title: Java NIO
date: 2016-04-01 10:55:23
tags: Java
---
Java NIO是是jdk1.4里提供的新API, 为所有的原始类型提供缓存支持。几个主要概念：

1. Channels and Buffers: 和标准的基于字节流或者字符流的标准IO API相比，NIO利用Channel和Buffer，数据直接从Channel读去到内存Buffer，或者从Buffer写入Channel， 减少了数据的拷贝。
Channel维持了客户端和服务器的链接资源。也可以理解为客户端和服务端的一个“管道”
2. NIO支持非阻塞IO(Non-blocking IO)：通过注册事件来触发对应的handler来执行
3. Selector: 用来监控多个Channel上的事件，一个线程可以监控多个Channel的数据读和些。