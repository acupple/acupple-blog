---
title: P-Unit结合JUnit做并发性能测试
date: 2016-02-25 13:31:18
tags: Java
---

JUnit大家一定都很熟悉了，这里就不做多的介绍。我们重点说一下p-unit的应用。
p-unit是一款开源的测试框架，支持在多线程中跑同样的测试用例.
比如你有如下JUnit的测试代码:
{% codeblock lang:java%}
public class sampleTest {
    @Test
    public void test1() {
      //some logic and assert
    }
    @Test
    public void test2() {
      // do something
    }
}
{% endcodeblock %}

1. 并发的跑一个unittest中的所有test cases。
{% codeblock lang:java%}
SoloRunner runner = new SoloRunner();
//如果你是用JUnit 的@Test annotation, 这个是必须的。
runner.setConvention(new JUnitAnnotationConvention());
runner.run(sampleTest.class);
{% endcodeblock %}

此时test1和test2会并发执行
2. 对一个unittest中的每个方法用多个线程去执行。

{% codeblock lang:java%}
ConcurrentRunner crunner = new ConcurrentRunner(100);
crunner.setConvention(new JUnitAnnotationConvention());
crunner.run(sampleTest.class);
{% endcodeblock %}

此时分别会有100个线程并发的执行test1和test2

Source code: https://svn.code.sf.net/p/p-unit/code/trunk/
里面有源码和简单的测试代码和说明，可供参考
http://www.ibm.com/developerworks/cn/java/j-lo-punit/