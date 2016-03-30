---
title: LMAX Distruptor
date: 2016-02-25 15:24:06
tags: Java
---

LAMAX Distruptor是一个高性能，低延迟的producer-consumer框架。到底有多吊呢，可以参考Github上给出的性能测试结果：
https://github.com/LMAX-Exchange/disruptor/wiki/Performance-Results

其核心就是RingBuffer这个东东, 感兴趣的同学可以参考下面的链接了解更多:
http://mechanitis.blogspot.jp/2011/06/dissecting-disruptor-whats-so-special.html

可以实现多种组合模式，我们这里列举几个比较常用的，在开始之前我们先定义需要在生产者和消费者之间传递的对象，以及产生实例的工厂方法。

{% codeblock lang:java%}
public class LongEvent {
    private long value;
    public void setValue(long value)
    {
        this.value = value;
    }
    @Override
    public String toString(){
        return this.value + "";
    }
    public static EventFactory FACTORY = new EventFactory(){
        @Override
        public LongEvent newInstance() {
            return new LongEvent();
        }
    };
}
{% endcodeblock %}

因为所有的Event对象都是托管给Distruptor由EventFactory创建的，且需要重复利用，所以定义Event的时候必须要提供setter方法。
其次我们需要定义一个处理Event的handler:

{% codeblock lang:java%}
public class LongEventHandler implements EventHandler {
    private final String name;
    public LongEventHandler(String name){
        this.name = name;
    }
    public void onEvent(LongEvent event,
                  long sequence, boolean endOfBatch)
    {
        System.out.println(this.name + ": " + event);
    }
}
{% endcodeblock %}

然后我们定义一个产生Event的Producer:

{% codeblock lang:java %}
public class LongEventProducer {
    private final RingBuffer ringBuffer;
    public LongEventProducer(RingBuffer ringBuffer)
    {
        this.ringBuffer = ringBuffer;
    }

    public void onData(long val)
    {
        // Grab the next sequence
        long sequence = ringBuffer.next();
        try
        {
            // Get the entry in the Disruptor for the sequence
            LongEvent event = ringBuffer.get(sequence);
            // Fill with data
            event.setValue(val);
        }
        finally
        {
            //发布Event
            ringBuffer.publish(sequence);
        }
    }
}
{% endcodeblock %}
Single producer- single consumer:
{% codeblock lang:java %}
public static void main(String[] args){
    int bufferSize = 1024;
    Executor executor = Executors.newCachedThreadPool();
    Disruptor disruptor = new Disruptor(LongEvent.FACTORY,
            bufferSize,
            executor, ProducerType.MULTI,
            new SleepingWaitStrategy());
    LongEventHandler handler = new new LongEventHandler("p1-s1");
    disruptor.handleEventsWith(handler);

    LongEventProducer producer = new
            LongEventProducer(disruptor.getRingBuffer());
    disruptor.start();
    for (int i = 0; i &lt; 1000; i++)
    {
        producer.onData(i);
    }
}
{% endcodeblock %}
Single producer-pipeline consumers:
{% codeblock lang:java %}
public static void main(String[] args){
    int bufferSize = 1024;
    Executor executor = Executors.newCachedThreadPool();
    Disruptor disruptor = new Disruptor(LongEvent.FACTORY,
            bufferSize,
            executor, ProducerType.MULTI,
            new SleepingWaitStrategy());
    LongEventHandler handler = new LongEventHandler("p1-s1");
    LongEventHandler handler1 = new LongEventHandler("p2-s2");
    disruptor.handleEventsWith(handler)
                .handleEventsWith(handler1);

    LongEventProducer producer = new
            LongEventProducer(disruptor.getRingBuffer());
    disruptor.start();
    for (int i = 0; i &lt; 1000; i++)
    {
        producer.onData(i);
    }
}
{% endcodeblock %}

可以发现p2-s2的处理都在p1-s2之后完成
Single producer-work pool(多个consumer竞争消费),这种模式是多个handler并发的竞争同一个Event消息，需要实现WorkHandler接口
{% codeblock lang:java %}
public class LongEventHandler2 implements WorkHandler {
    private final String name;
    public LongEventHandler2(String name){
        this.name = name;
    }

    @Override
    public void onEvent(LongEvent event) throws Exception {
        System.out.println(this.name + ": " + event);
    }

    public static void main(String[] args){
        int bufferSize = 1024;
        Executor executor = Executors.newCachedThreadPool();
        Disruptor disruptor = new Disruptor(LongEvent.FACTORY,
                bufferSize,
                executor, ProducerType.MULTI,
                new SleepingWaitStrategy());
        LongEventHandler2 handler = new LongEventHandler2("p1-s1");
        LongEventHandler2 handler1 = new LongEventHandler2("p2-s2");
        disruptor.handleEventsWithWorkerPool(handler, handler1);

        LongEventProducer producer = new
                LongEventProducer(disruptor.getRingBuffer());
        disruptor.start();
        for (int i = 0; i &lt; 1000; i++)
        {
            producer.onData(i);
        }
    }
}
{% endcodeblock %}

这三种基本模式还可以组合到一起实现更为复杂的处理逻辑，具体的使用方面同学们可以自己琢磨，最主要的是对RingBuffer的使用， 详细的可以参考Github上的测试代码

关于等待策略WaitStrategy的选择需要特别注意，默认是选择BlockingWaitStrategy。

1. BlockingWaitStrategy:这个策略的内部适用一个锁和条件变量来控制线程的执行和等待（Java基本的同步方法）。BlockingWaitStrategy是最慢的等待策略，但也是CPU使用率最低和最稳定的选项。然而，可以根据不同的部署环境调整选项以提高性能。
2. SleepingWaitStrategy: 和BlockingWaitStrategy一样，SpleepingWaitStrategy的CPU使用率也比较低。它的方式是循环等待并且在循环中间调用LockSupport.parkNanos(1)来睡眠，（在Linux系统上面睡眠时间60µs）.然而，它的优点在于生产线程只需要计数，而不执行任何指令。并且没有条件变量的消耗。但是，事件对象从生产者到消费者传递的延迟变大了。SleepingWaitStrategy最好用在不需要低延迟，而且事件发布对于生产者的影响比较小的情况下。比如异步日志功能
3. YieldingWaitStrategy:YieldingWaitStrategy是可以被用在低延迟系统中的两个策略之一，这种策略在减低系统延迟的同时也会增加CPU运算量。YieldingWaitStrategy策略会循环等待sequence增加到合适的值。循环中调用Thread.yield()允许其他准备好的线程执行。如果需要高性能而且事件消费者线程比逻辑内核少的时候，推荐使用YieldingWaitStrategy策略。例如：在开启超线程的时候
4. BusySpinWaitStrategy:BusySpinWaitStrategy是性能最高的等待策略，同时也是对部署环境要求最高的策略。这个性能最好用在事件处理线程比物理内核数目还要小的时候。例如：在禁用超线程技术的时候。

使用的时候需要根据自己的实际需求选择。

使用队列来实现生产者消费者模式的问题：
1. 只适应单个生产者，单个消费者的情况
2. 队列头和尾，会被频繁修改
3. 因为生产者和消费者的速度不一致，队列常常是空的，或者是满的，而且这种情况经常发生。


disruptor获取一个游标后，必须要publish，放到finally代码块中，否则可能会阻塞其他producer对RingBuffer的Commit(Publish)


http://ifeve.com/disruptor/
