---
title: tomcat启动123
date: 2016-05-23 17:05:02
tags: Java
---
# 让Tomcat支持引用软连接资源
默认情况下想通过在Tomcat下建立软连接来使tomcat上的应用引用该资源是不行的，会出现类似错误：
```java
java.lang.IllegalStateException: ContainerBase.addChild: start: LifecycleException:  
start: :  java.io.IOException: Failed to access resource XXX
```
这时候需要打开支持引用软连接资源的开关：allowLinking="true"，比如在context标签下加入allowLinking="true"：
```xml
<Context path="/XXX" override="true" allowLinking="true"></Context>
```
或者在resources标签下加入allowLinking="true"
```xml
<Resources className="xoxo" allowLinking="true"></Resources >
```
##DefaultCharset
对defaultCharset,在jvm里的语言就是初始话在启动的时候，而且不可被更改，你只能修改系统的charset,或者jvm的启动参数里加上 -Dfile.encoding="UTF-8"

设置Dfile.encoding后文件内容正常了，但是创建文件名或，中文目录或者文件打包的时候，仍然出现乱码，这个时候需要设置另一个参数: -Dsun.jnu.encoding=UTF-8
这两个参数的区别也就在这里了：

1. -Dfile.encoding设置文件内容的编码
2. -Dsun.jnu.encoding设置文件名称的编码

PS: 搞不清楚为什么把文件内容和文件名的编码，分开来设置，知道的朋友可以告诉我哦:)

##Saltstack执行cmd.run重启tomcat后出现日志乱码
Salt在执行cmd.run前会将minion端的字符集默认设置为“C”，而目前大部分tomcat应用使用的是UTF-8字符集，所以Salt执行cmd.run重启tomcat后会出现日志乱码。
解决办法：这些salt命令的时候将LC_ALL设置为空，或者UTF-8
```bash
salt '10.0.10.100' cmd.run  'locale' env='{"LC_ALL": ""}'    #增加参数env='{"LC_ALL": ""}'  
```
参考： http://blog.csdn.net/hnhuangyiyang/article/details/50421738