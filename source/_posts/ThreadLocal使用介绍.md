---
title: ThreadLocal使用介绍
date: 2016-03-22 18:10:30
tags: Java
---
## 如何使用ThreadLocal
在系统中任意一个适合的位置定义个ThreadLocal变量，可以定义为public static类型（直接new出来一个ThreadLocal对象），要向里面放入数据就使用set(Object)，要获取数据就用get()操作，删除元素就用remove()，其余的方法是非public的方法，不推荐使用。

下面是一个简单例子（代码片段1）：
```java
public class ThreadLocalTest2 {
 
    public final static ThreadLocal <String>TEST_THREAD_NAME_LOCAL = new ThreadLocal<String>();
 
    public final static ThreadLocal <String>TEST_THREAD_VALUE_LOCAL = new ThreadLocal<String>();
 
    public static void main(String[]args) {
        for(int i = 0 ; i < 100 ; i++) {
            final String name = "线程-【" + i + "】";
            final String value =  String.valueOf(i);
            new Thread() {
                public void run() {
                    try {
                        TEST_THREAD_NAME_LOCAL.set(name);
                        TEST_THREAD_VALUE_LOCAL.set(value);
                        callA();
                    }finally {
                        TEST_THREAD_NAME_LOCAL.remove();
                        TEST_THREAD_VALUE_LOCAL.remove();
                    }
                }
            }.start();
        }
    }
 
    public static void callA() {
        callB();
    }
 
    public static void callB() {
        new ThreadLocalTest2().callC();
    }
 
    public void callC() {
        callD();
    }
 
    public void callD() {
        System.out.println(TEST_THREAD_NAME_LOCAL.get() + "\t=\t" + TEST_THREAD_VALUE_LOCAL.get());
    }
}
```
这里模拟了100个线程去访问分别设置name和value，中间故意将name和value的值设置成一样，看是否会存在并发的问题，通过输出可以看出，线程输出并不是按照顺序输出，说明是并行执行的，而线程name和value是可以对应起来的，中间通过多个方法的调用，以模实际的调用中参数不传递，如何获取到对应的变量的过程，不过实际的系统中往往会跨类，这里仅仅在一个类中模拟，其实跨类也是一样的结果，大家可以自己去模拟就可以。

相信看到这里，很多程序员都对ThreadLocal的原理深有兴趣，看看它是如何做到的，尽然参数不传递，又可以像局部变量一样使用它，的确是蛮神奇的，其实看看就知道是一种设置方式，看到名称应该是是和Thread相关，那么废话少说，来看看它的源码吧，既然我们用得最多的是set、get和remove，那么就从set下手(代码片段2）：
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
首先获取了当前的线程，和猜测一样，然后有个getMap方法，传入了当前线程，我们先可以理解这个map是和线程相关的map，接下来如果   不为空，就做set操作，你跟踪进去会发现，这个和HashMap的put操作类似，也就是向map中写入了一条数据，如果为空，则调用createMap方法，进去后，看看（代码片段3）:
```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
返现创建了一个ThreadLocalMap，并且将传入的参数和当前ThreadLocal作为K-V结构写入进去（代码片段4）：
```java
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```
这里就不说明ThreadLocalMap的结构细节，只需要知道它的实现和HashMap类似，只是很多方法没有，也没有implements Map，因为它并不想让你通过某些方式（例如反射）获取到一个Map对他进一步操作，它是一个ThreadLocal里面的一个static内部类，default类型，仅仅在java.lang下面的类可以引用到它，所以你可以想到Thread可以引用到它。

我们再回过头来看看getMap方法，因为上面我仅仅知道获取的Map是和线程相关的，而通过代码片段3，有一个t.threadLocalMap = new ThreadLocalMap(this, firstValue)的时候，相信你应该大概有点明白，这个变量应该来自Thread里面，我们根据getMap方法进去看看：
```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
是的，是来自于Thread，而这个Thread正好又是当前线程，那么进去看看定义就是：
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```
这个属性就是在Thread类中，也就是每个Thread默认都有一个ThreadLocalMap，用于存放线程级别的局部变量，通常你无法为他赋值，因为这样的赋值通常是不安全的。

好像是不是有点乱，不着急，我们回头先摸索下思路：
1. Thread里面有个属性是一个类似于HashMap一样的东西，只是它的名字叫ThreadLocalMap，这个属性是default类型的，因此同一个package下面所有的类都可以引用到，因为是Thread的局部变量，所以每个线程都有一个自己单独的Map，相互之间是不冲突的，所以即使将ThreadLocal定义为static线程之间也不会冲突。

2. ThreadLocal和Thread是在同一个package下面，可以引用到这个类，可以对他做操作，此时ThreadLocal每定义一个，用this作为Key，你传入的值作为value，而this就是你定义的ThreadLocal，所以不同的ThreadLocal变量，都使用set，相互之间的数据不会冲突，因为他们的Key是不同的，当然同一个ThreadLocal做两次set操作后，会以最后一次为准。

3. 综上所述，在线程之间并行，ThreadLocal可以像局部变量一样使用，且线程安全，且不同的ThreadLocal变量之间的数据毫无冲突。
我们继续看看get方法和remove方法，其实就简单了：
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}
```
通过根据当前线程调用getMap方法，也就是调用了t.threadLocalMap，然后在map中查找，注意Map中找到的是Entry，也就是K-V基本结构，因为你set写入的仅仅有值，所以，它会设置一个e.value来返回你写入的值，因为Key就是ThreadLocal本身。你可以看到map.getEntry也是通过this来获取的。

同样remove方法为：
```java
public void remove() {
     ThreadLocalMap m = getMap(Thread.currentThread());
     if (m != null)
         m.remove(this);
}
```
同样根据当前线程获取map，如果不为空，则remove，通过this来remove。

大家从前面应该可以看出来，这个ThreadLocal相关的对象是被绑定到一个Map中的，而这个Map是Thread线程的中的一个属性，那么就有一个问题是，如果你不自己remove的话或者说如果你自己的程序中不知道什么时候去remove的话，那么线程不注销，这些被set进去的数据也不会被注销。容易造成OOM的问题。