---
title: Java内存管理机制笔记
date: 2016-03-19 21:28:55
tags: Java
---
## JVM内存区域
- 程序计数器： 线程私有，存放当前线程所执行的字节码的行号，通过改变这个计数器的值来选取下一条需要执行的字节码命令。线程挂起恢复也需要通过该计数器恢复执行位置。
- JVM栈: 线程私有，与线程的生命周期相同。每个方法执行的同时都会创建一个栈帧，存放局部变量，对象引用，方法出口等。
- Java堆: 所以线程共享，几乎所有的对象实例都存在这里(有些产量，例如String会放在方法区的常量池中)，GC垃圾回收的主要对象。
- 方法区: 存放编译器编译后的代码信息，常量，静态变量等。注意该区域和GC回收中的“永久代”是不完全等同的。只是Hotspot JVM使用“永久代”来实现方法区，但是“永久代”不代表就全部被方法区占用。
- 运行时常量池： 属于方法区，用于存方编译期生成的各种字面量和符号引用。
- 直接内存：NIO类中使用DirectByteBuffer来避免JVM内存和系统内存中来回拷贝数据造成的性能损耗。

## JVM能够创建线程的具体计算公式：
这个异常问题本质原因是我们创建了太多的线程，而能创建的线程数是有限制的，导致了异常的发生。能创建的线程数的具体计算公式如下：
(MaxProcessMemory - JVMMemory - ReservedOsMemory) / (ThreadStackSize) = Number of threads
MaxProcessMemory 指的是一个进程的最大内存
JVMMemory         JVM内存
ReservedOsMemory  保留的操作系统内存
ThreadStackSize      线程栈的大小

在java语言里， 当你创建一个线程的时候，虚拟机会在JVM内存创建一个Thread对象同时创建一个操作系统线程，而这个系统线程的内存用的不是JVMMemory，而是系统中剩下的内存(MaxProcessMemory - JVMMemory - ReservedOsMemory)

MaxProcessMemory 在32位的 windows下是 2G
JVMMemory   eclipse默认启动的程序内存是64M
ReservedOsMemory  一般是130M左右
ThreadStackSize 32位 JDK 1.6默认的stacksize 325K左右, 可以通过-Xss 来设置
公式如下：
(2*1024*1024-64*1024-130*1024)/325 = 5841

## GC垃圾回收
考虑内存回收，需要思考3件事情：
1. 哪些内存需要回收？
2. 什么时候回收？
3. 如何回收？

结合前面的JVM内存区域，不难搞清楚，GC回收的主要就是Java堆内存了，也是JVM种最大的一块儿内存，Hotspot JVM对方法区的内存也有做垃圾回收(已经卸载的class， 常量池等)。

而什么时候回收，即如何判断对象已经“死亡”(不会再被使用)，现在主流的JVM实现都采用可达性分析算法，通过分析对象与GC root对象的是否可达来标记一个对象是否可以被垃圾回收，那么哪些对象可以作为GC root呢？包括以下几种：
1. 虚拟机栈中引用的对象
2. 方法区中类静态属性引用的对象
3. 方法区中常量引用的对象

JVM的引用类型
- 强引用： Object object = new Object， 这种对象只要GC root引用链还存在，垃圾回收器是永远不会回收的
- 软引用： 当堆内存不够用的时候，内存发生溢出之前，会把这些对象列入回收范围进行二次回收，如果没有回收掉才会抛OutOfMemeryException， 一般可以用于Cache数据。
- 弱引用： 对象只能存活到下一次GC回收
- 虚引用： 不能通过虚引用来访问一个对象，该引用的唯一目的是在发生GC的时候得到一个通知。

流行的回收算法基本上都是围绕：标记-移动擦除来完成垃圾回收的。所谓的标记，即JVM每隔一段时间会去遍历GC root的对象链(有向图)，对不可达的对象标记为可以回收。移动擦除是指将标记为可以回收的对象集中移到一个地方（To Survivor），然后对(From Survivor)其进行统一回收。
由于2各Survior存在，每次只能使用一个来给对象分配内存，这样做会减少内存的使用率，但是大多数对象都是“朝生夕死”的，主流的JVM实现（Hotspot）将堆内存分为新生代，永久代，又将新生代分为: Eden和两个较小的Survivor，当回收时，将Eden和From survivor中还存活的对象移到另一个To survivor, 然后清理掉Eden和From Survivor, 两个Survivor交替使用。这样只会浪费很少的内存， 例如Hotspot JVM中Eden和Survivor的默认比例为：8：1：1， 因此只会浪费10%的内存。其次如果在To survivor不够咋办？一般通过老年代来进行补偿，当然尽量不要这么做，可以通过调整JVM参数来控制整个比例。

## 主要的GC算法实现
1. Serial: 串行，效率最高，但是会使用户线程出现短暂的停止
2. ParNew: 并行多线程，清理使用多线程实现。
3. Parallel Scavenge： 高吞吐量
4. CMS: 用户不会感觉到停顿，吞吐量不如Parallel Scavenge
5. Gl: 3和4的折中
