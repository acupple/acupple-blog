---
title: JVM性能调优工具
date: 2016-03-25 13:28:52
tags: Java
---
企业级Java应用中经常碰到的问题：
1. OutOfMemoryError，内存不足
2. 内存泄露
3. 线程死锁
4. 锁争用（Lock Contention）
5. Java进程消耗CPU过高

往往大多数人的处理办法只是重启服务，者调大内存，而不会深究问题根源。其实JVM自带了很多调优监控工具，例如jps、jstack、jmap、jhat、jstat、hprof等，下面我们对这些工具做一下详细的整理，以备不时之需。

## jps(Java Virtual Machine Process Status Tool)
从字面意思可以看出这是一个用于输出VM中运行的进程状态信息，语法格式如下：
```bash
jps [options] [hostid]
```
如果不指定hostid就默认为当前主机或服务器, 命令行参数选项说明如下：

> **-q** 不输出类名、Jar名和传入main方法的参数
> **-m** 输出传入main方法的参数
> **-l** 输出main类或Jar的全限名
> **-v** 输出传入JVM的参数

比如下面：
```bash
> jps -mlv
1203 org.apache.zookeeper.server.quorum.QuorumPeerMain 
   /usr/local/etc/zookeeper/zoo.cfg 
   -Dzookeeper.log.dir=. 
   -Dzookeeper.root.logger=INFO,CONSOLE 
   -Dcom.sun.management.jmxremote 
   -Dcom.sun.management.jmxremote.local.only=false
```
java main方法：org.apache.zookeeper.server.quorum.QuorumPeerMain
main方法的参数：/usr/local/etc/zookeeper/zoo.cfg，明显是个配置文件
jvm参数：-Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false

## jstack
jstack主要用来查看某个Java进程内的线程堆栈信息。语法格式如下：
```bash
jstack [option] pid
jstack [option] executable core
jstack [option] [server_id@]<remote server IP or hostname>
```
命令行参数选项说明如下：
> **-F**  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
> **-l** long listings，Prints additional information about locks
> **-m** mixed mode，to print both java and native frames
jstack可以定位到线程堆栈，根据堆栈信息我们可以定位到具体代码，所以它在JVM性能调优中使用得非常多。下面我们来一个实例找出某个Java进程中最耗费CPU的Java线程并定位堆栈信息，用到的命令有ps、top、printf、jstack、grep。

首先找出Java进程ID，服务器上的Java应用名称为tomcat_9000：
```bash
> ps -ef|grep tomcat_9000 ##也可以使用上面介绍的jps命令来查看
> root  21120   1  0 Mar24 ?    00:01:33 /usr/bin/java...........
```
得到进程ID为1203，第二步找出该进程内最耗费CPU的线程，可以使用
1）ps -Lfp pid
2）ps -mp pid -o THREAD, tid, time
3）top -Hp pid
用第三个，输出如下：
```bash
top - 11:44:22 up 280 days,  6:51,  1 user,  load average: 0.05, 0.09, 0.08
Tasks:  40 total,   0 running,  40 sleeping,   0 stopped,   0 zombie
Cpu(s):  4.6%us,  3.3%sy,  0.0%ni, 91.9%id,  0.0%wa,  0.0%hi,  0.1%si,  0.0%st
Mem:  16332304k total, 16087428k used,   244876k free,   161628k buffers
Swap:  2097144k total,    67620k used,  2029524k free,  4264820k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
21166 root      20   0 6381m 529m  10m S  0.3  3.3   0:03.64 java  ##这个时间最长
21120 root      20   0 6381m 529m  10m S  0.0  3.3   0:00.00 java
21123 root      20   0 6381m 529m  10m S  0.0  3.3   0:00.85 java
.........
```
TIME列就是各个Java线程耗费的CPU时间，CPU时间最长的是线程ID为21742的线程，用
```bash
> printf "%x\n" 21166
> 52ae ##得到21166的十六进制值为52ae
```
下一步终于轮到jstack上场了，它用来输出进程21120的堆栈信息，然后根据线程ID的十六进制值grep，如下：
```bash
> jstack 21120 | grep 52ae
> "pool-1-thread-2" prio=10 tid=0x00007f5520a8e000 nid=0x52ae 
          waiting on condition [0x00007f55678db000]
```
可以看到CPU消耗在pool-1-thread-2这个类的waiting一个资源，然后再根据这个信息找相关的代码进行定位
**我们实际开发当中最好是为自己用到的线程都起一个名字，这样方便问题定位。**
## jmap（Memory Map）和jhat（Java Heap Analysis Tool）
jmap用来查看堆内存使用状况，一般结合jhat使用。jmap语法格式如下：
```bash
jmap [option] pid
jmap [option] executable core
jmap [option] [server-id@]remote-hostname-or-ip
```
如果运行在64位JVM上，可能需要指定-J-d64命令选项参数。
```bash
jmap -permstat pid
```
打印进程的类加载器和类加载器加载的持久代对象信息，输出：类加载器名称、对象是否存活（不可靠）、对象地址、父类加载器、已加载的类大小等信息
```bash
Attaching to process ID 21120, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.65-b04
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness.liveness analysis may be inaccurate ...
class_loader    classes bytes   parent_loader   alive?  type

<bootstrap>     2709    15916216          null          live    <internal>
0x00000007cf4c71b8      1       1888    0x00000007094d4dd0      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
0x00000007cf4c6e78      1       3048    0x00000007094d4dd0      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
0x0000000709746150      1       3032    0x00000007094d4e90      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
0x00000007cf4c63b8      1       3160    0x00000007094d4dd0      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
0x00000007cf4c7c78      1       1888    0x00000007094d4dd0      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
0x00000007cebf33f8      843     5023416 0x00000007094829a0      dead    com/github/ompc/greys/agent/AgentLauncher$1@0x0000000703f6a428
0x00000007cf4c67f8      1       1888    0x00000007094d4dd0      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
0x00000007cf4c7838      1       3056    0x00000007094d4dd0      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
0x00000007cf4c75f8      1       1888    0x00000007094d4dd0      dead    sun/reflect/DelegatingClassLoader@0x0000000701a4fc00
............
```
使用jmap -heap pid查看进程堆内存使用情况，包括使用的GC算法、堆配置参数和各代中堆内存使用情况
```bash
> jmap -heap 21120
Attaching to process ID 21120, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.65-b04

using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio = 0
   MaxHeapFreeRatio = 100
   MaxHeapSize      = 4181721088 (3988.0MB)
   NewSize          = 1310720 (1.25MB)
   MaxNewSize       = 17592186044415 MB
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2
   SurvivorRatio    = 8
   PermSize         = 21757952 (20.75MB)
   MaxPermSize      = 85983232 (82.0MB)
   G1HeapRegionSize = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 529006592 (504.5MB)
   used     = 144885120 (138.1732177734375MB)
   free     = 384121472 (366.3267822265625MB)
   27.388150202861745% used
From Space:
   capacity = 36700160 (35.0MB)
   used     = 29430008 (28.06664276123047MB)
   free     = 7270152 (6.933357238769531MB)
   80.19040788922992% used
To Space:
   capacity = 38273024 (36.5MB)
   used     = 0 (0.0MB)
   free     = 38273024 (36.5MB)
   0.0% used
PS Old Generation
   capacity = 174063616 (166.0MB)
   used     = 52265328 (49.84410095214844MB)
   free     = 121798288 (116.15589904785156MB)
   30.026566838643635% used
PS Perm Generation
   capacity = 48758784 (46.5MB)
   used     = 48530080 (46.281890869140625MB)
   free     = 228704 (0.218109130859375MB)
   99.53094810567876% used

28979 interned Strings occupying 3495552 bytes.
```
使用jmap -histo[:live] pid查看堆内存中的对象数目、大小统计直方图，如果带上live则只统计活对象
```bash
>jmap -histo:live 21120 | more
[root@hadoop-slave2 ~]# jmap -histo:live 21120 | more

 num     #instances         #bytes  class name
----------------------------------------------
   1:         85719       12566608  <constMethodKlass>
   2:         85719       10984000  <methodKlass>
   3:         81753       10101688  [C
   4:         24330        9440816  [B
   5:          7688        9084536  <constantPoolKlass>
   6:          7688        5545568  <instanceKlassKlass>
   7:          6456        5112896  <constantPoolCacheKlass>
   8:         80168        1924032  java.lang.String
   9:          2826        1487288  <methodDataKlass>
  10:          8338        1005248  java.lang.Class
  11:         24326         778432  java.util.HashMap$Entry
  12:          9065         725200  java.lang.reflect.Method
  13:         11042         663144  [S
  14:         12478         649856  [[I
  15:         17517         560544  java.util.concurrent.ConcurrentHashMap$HashEntry
  16:          3317         486688  [Ljava.util.HashMap$Entry;
  17:         11360         454400  java.util.LinkedHashMap$Entry
  18:           633         344352  <objArrayKlassKlass>
  19:          6276         313664  [Ljava.lang.Object;
  20:          5230         251040  org.apache.catalina.loader.ResourceEntry
  21:          3684         235776  java.net.URL
  22:          4166         233296  org.apache.naming.resources.CacheEntry
  23:          1367         171576  [Ljava.util.concurrent.ConcurrentHashMap$HashEntry;
  24:          2992         167552  java.util.LinkedHashMap
  25:          4159         166360  java.lang.ref.SoftReference
  26:          1262         161712  [I
  27:          4761         152352  java.lang.ref.WeakReference
  ...................
```
class name是对象类型，说明如下:
> **B**  byte
> **C**  char
> **D**  double
> **F**  float
> **I**  int
> **J**  long
> **Z**  boolean
> **[**  数组，如[I表示int[]
> **[L+类名** 其他对象

还有一个很常用的情况是：用jmap把进程内存使用情况dump到文件中，再用jhat分析查看。jmap进行dump命令格式如下：
```bash
jmap -dump:format=b,file=dumpFileName pid
```
我一样地对上面进程ID为21120进行Dump：
```bash
> jmap -dump:format=b,file=/tmp/dump.dat 21120     
Dumping heap to /tmp/dump.dat ...
Heap dump file created
```
dump出来的文件可以用MAT、VisualVM等工具查看，这里用jhat查看：
```bash
> jhat -port 9998 /tmp/dump.dat
Dumping heap to /tmp/dump.dat ...
Heap dump file created
[root@hadoop-slave2 ~]# jhat -port 9998 /tmp/dump.dat
Reading from /tmp/dump.dat...
Dump file created Fri Mar 25 12:55:56 CST 2016
Snapshot read, resolving...
Resolving 425879 objects...
Chasing references, expect 85 dots.....................................................................................
Eliminating duplicate references.....................................................................................
Snapshot resolved.
Started HTTP server on port 9998
Server is ready.
```
注意如果Dump文件太大，可能需要加上-J-Xmx512m这种参数指定最大堆内存，即jhat -J-Xmx512m -port 9998 /tmp/dump.dat。然后就可以在浏览器中输入主机地址:9998查看了
![image](/../../gallery/jvm_1.png)

## jstat（JVM统计监测工具）
语法格式如下：
```bash
jstat [ generalOption | outputOptions vmid [interval[s|ms] [count]] ]
```
vmid是Java虚拟机ID，在Linux/Unix系统上一般就是进程ID。interval是采样时间间隔。count是采样数目。比如下面输出的是GC信息，采样时间间隔为250ms，采样数为4
```bash
> jstat -gc 21120 250 4
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
36864.0 37376.0  0.0    0.0   516608.0  7578.5   110592.0   28695.4   83968.0 47088.6      9    0.585   1      0.173    0.758
36864.0 37376.0  0.0    0.0   516608.0  7578.5   110592.0   28695.4   83968.0 47088.6      9    0.585   1      0.173    0.758
36864.0 37376.0  0.0    0.0   516608.0  7578.5   110592.0   28695.4   83968.0 47088.6      9    0.585   1      0.173    0.758
36864.0 37376.0  0.0    0.0   516608.0  7578.5   110592.0   28695.4   83968.0 47088.6      9    0.585   1      0.173    0.758
```
要明白上面各列的意义，先看JVM堆内存布局：
![image](/../../gallery/jvm_2.jpg)
可以看出：
>堆内存 = 年轻代 + 年老代 + 永久代
>年轻代 = Eden区 + 两个Survivor区（From和To）

现在来解释各列含义：
>S0C、S1C、S0U、S1U：Survivor 0/1区容量（Capacity）和使用量（Used）
>EC、EU：Eden区容量和使用量
>OC、OU：年老代容量和使用量
>PC、PU：永久代容量和使用量
>YGC、YGT：年轻代GC次数和GC耗时
>FGC、FGCT：Full GC次数和Full GC耗时
>GCT：GC总耗时

## hprof（Heap/CPU Profiling Tool）
hprof能够展现CPU使用率，统计堆内存使用情况。语法格式如下：
```bash
java -agentlib:hprof[=options] ToBeProfiledClass
java -Xrunprof[:options] ToBeProfiledClass
javac -J-agentlib:hprof[=options] ToBeProfiledClass
```
来几个官方指南上的实例。
```bash
CPU Usage Sampling Profiling(cpu=samples)的例子：
java -agentlib:hprof=cpu=samples,interval=20,depth=3 Hello
  上面每隔20毫秒采样CPU消耗信息，堆栈深度为3，生成的profile文件名称是java.hprof.txt，在当前目录。 
  CPU Usage Times Profiling(cpu=times)的例子，它相对于CPU Usage Sampling Profile能够获得更加细粒度的CPU消耗信息，能够细到每个方法调用的开始和结束，它的实现使用了字节码注入技术（BCI）：
javac -J-agentlib:hprof=cpu=times Hello.java
  Heap Allocation Profiling(heap=sites)的例子：
javac -J-agentlib:hprof=heap=sites Hello.java
  Heap Dump(heap=dump)的例子，它比上面的Heap Allocation Profiling能生成更详细的Heap Dump信息：
javac -J-agentlib:hprof=heap=dump Hello.java
  虽然在JVM启动参数中加入-Xrunprof:heap=sites参数可以生成CPU/Heap Profile文件，但对JVM性能影响非常大，不建议在线上服务器环境使用。
```