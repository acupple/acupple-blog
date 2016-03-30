---
title: JVM性能调优实战
date: 2016-03-25 13:40:13
tags: Java
---
使用jstack来分析死锁问题
```java
public class Application {
    public static void main(String[] args){
        System.out.println(" start the example ----- ");
        final Object obj_1 = new Object(), obj_2 = new Object();

        Thread t1 = new Thread("t1"){
            @Override
            public void run() {
                synchronized (obj_1) {
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {}

                    synchronized (obj_2) {
                        System.out.println("thread t1 done.");
                    }
                }
            }
        };

        Thread t2 = new Thread("t2"){
            @Override
            public void run() {
                synchronized (obj_2) {
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {}

                    synchronized (obj_1) {
                        System.out.println("thread t2 done.");
                    }
                }
            }
        };

        t1.start();
        t2.start();
    }
}
```
这明显会产生死锁，及t1和t2都在相互等待对方释放锁。假使在我们不知情的情况下,运行这样一个方法后,发现等了N久都没有在屏幕打印线程完成信息。这个时候我们就可以使用jps查看该程序的jpid值和使用jstack来生产堆栈结果问题。
```bash
> jps 
2110 Launcher
2109 AppMain ##这个就是我要查找的进程ID
> jstack -l 2109 > deadlock.jstack
```
结果文件deadlock.jstack内容如下：
```bash
2016-03-25 13:53:36
Full thread dump Java HotSpot(TM) 64-Bit Server VM (24.75-b04 mixed mode):
.....................

"t2" prio=5 tid=0x00007fd4ca046800 nid=0x4d03 waiting for monitor entry [0x0000000116d4e000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Application$2.run(Application.java:35)
	- waiting to lock <0x00000007d57843a8> (a java.lang.Object)
	- locked <0x00000007d57843b8> (a java.lang.Object)

   Locked ownable synchronizers:
	- None

"t1" prio=5 tid=0x00007fd4c98fe000 nid=0x4b03 waiting for monitor entry [0x0000000116c4b000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Application$1.run(Application.java:20)
	- waiting to lock <0x00000007d57843b8> (a java.lang.Object)
	- locked <0x00000007d57843a8> (a java.lang.Object)

   Locked ownable synchronizers:
	- None
......................

Found one Java-level deadlock:
=============================
"t2":
  waiting to lock monitor 0x00007fd4c9823f58 (object 0x00000007d57843a8, a java.lang.Object),
  which is held by "t1"
"t1":
  waiting to lock monitor 0x00007fd4c9822c18 (object 0x00000007d57843b8, a java.lang.Object),
  which is held by "t2"

Java stack information for the threads listed above:
===================================================
"t2":
	at Application$2.run(Application.java:35)
	- waiting to lock <0x00000007d57843a8> (a java.lang.Object)
	- locked <0x00000007d57843b8> (a java.lang.Object)
"t1":
	at Application$1.run(Application.java:20)
	- waiting to lock <0x00000007d57843b8> (a java.lang.Object)
	- locked <0x00000007d57843a8> (a java.lang.Object)

Found 1 deadlock.
```
部分信息已经省略了，从结果上很容易发现发现了一个死锁