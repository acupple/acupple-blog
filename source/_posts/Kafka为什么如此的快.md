---
title: Kafka为什么如此的快
date: 2016-04-20 10:01:45
tags: 大数据，分布式
---
Kafka是分布式的消息系统，需要处理海量的消息，Kafka的设计初衷是把所有消息都写入速度
低容量大的硬盘，以此来换取更强的存储能力，但是实际上，使用硬盘并没有带来
过性能的损失，这究竟为何？

Kafka主要使用以下几种方式实现了超高吞吐率的

## 顺序读写
Kafka的消息是不断追加到文件中的，这个特性使它可以充分利用磁盘的顺序读写能力。
顺序读写降低了硬盘磁头的寻道时间，只需要很少的扇区旋转时间，所以速度远快于
随机读写。Kafka官方给出的测试数据(Raid-5, 7200rpm)
顺序I/O: 600MB/s
随机I/O: 100KB/s
## 零拷贝
先简单的了解下文件系统的操作流程，例如一个程序要把文件内容发送到网络。
这个过程发生在用户空间，文件和网络socket属于硬件资源，两者之间有一个
内核空间，在操作系统内部，整个过程为:
![image](/../../gallery/zero-copy.jpg)
在Linux Kernal 2.2之后出现了一种叫做“零拷贝(zero-copy)”系统调用机制，
就是跳过“用户缓冲区”的拷贝，建立一个磁盘空间和内存空间的直接映射，数据不再复制到“用户态缓冲区”
系统上下文切换减少2次，可以提升一倍性能
![image](/../../gallery/user-space.jpg)
## 文件分段
Kafka的队列topic被分为了多个区partition, 每个partition又分为了多个segment，
所以一个队列中的消息实际上是保存在N多个片段文件中
![image](/../../gallery/file-seperate.jpg)
通过分段的方式，每次文件操作都是对一个小文件的操作，非常轻便，同时也增加了并行处理能力
## 批量发送
Kafka允许进行批量消息发送，先将消息缓存在内存中，然后一次请求批量发送出去，
比如可以指定缓存的消息达到某个量的时候发送，或者缓存了固定的时间后发送
这种策略大大减少了服务器端的I/O次数
## 数据压缩
Kafka还支持消息压缩，Producer可以通过GZIP或者Snappy格式对消息集合进行压缩，
从而减少网络传输的压力


其实这些优化都是比较常用的手段，在自己的分布式应用中可以借鉴12