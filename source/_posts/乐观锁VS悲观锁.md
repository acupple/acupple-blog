---
title: 乐观锁VS悲观锁
date: 2016-02-29 13:55:34
tags: 数据库
---

锁机制解决的并发导致的数据更新问题:
1. 更新丢失：一个事务的更新覆盖了其他事务的更新结果.
2. 脏读：用户AB看到的值都是6，B把值改成了2，但是用户A读到的值仍为6.


为了解决这些问题，必须引入并发控制机制，及锁。 
1. 悲观锁：顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。
2. 乐观锁：顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库如果提供类似于write_condition机制的其实都是提供的乐观锁。

两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下，即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果经常产生冲突，上层应用会不断的进行retry，这样反倒是降低了性能，所以这种情况下用悲观锁就比较合适。

乐观锁的实现:
1. 使用自增长的整数表示数据版本号。更新的时候检查版本是否一致，比如数据库中的版本是6， 更新提交时version=6+1， 使用7与数据库version+1=7进行比较，如果相等，则可以更新，如果不等则有可能其他程序已更新了该记录，返回错误。
2. 使用时间戳来实现。更新时用自己拿到的时间戳和数据库中的时间戳比较，相等则更新成功并同时更新数据库中的时间戳，否则返回错误。

MySQL InnerDB的事务隔离级别:
事务的ACID特性：原子性、一致性、隔离性、持久性。这部分不多说了，任何一本讲数据库理论的书籍里边都会有讲。MySQL InnoDB通过锁来实现事务的一致性和隔离性，共实现了四种事务隔离级别：
1. READ UNCOMMITTED：某个session中的事务可以看到其他session的事务中尚未提交的更改，而该更改可能回滚，也即会出现”脏读“；
2. READ COMMITTED：某个session中的事务只可以看到其他session的事务中已经提交的更改，不会出现”脏读“， 但一个事务对同一对象的两次查询结果可能出现不一致，也即会出现“不可重复读”；
3. REPEATABLE READ：某个session中的事务不能查询到其他session的事务中未提交和已经提交的更改，不会出现”不可重复读”，但期间，其他事务却可能对数据进行更改并提交，而这些更改对前一个事务中的INSERT/UPDATE/DELETE等语句是可见的，因此可能出现”更新丢失“，另外，虽然SELECT不到，但对其进行更改操作时却真实的存在，就好像幻象一样，且更改过后可以被同一事物中的SELECT看到，也即”幻像读“（其实”不可重复读”也可理解为”幻像读“，因为同一事务中前后两次读取到的结果不一样，看个人怎么理解了，这些叫法只是一个名字而已）；
4. SERIALIZABLE：使事务串行化执行，解决上述问题。所有的读操作均为当前读，读加读锁 (S锁)，写加写锁 (X锁)Serializable隔离级别下，读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用。

总而言之: 同一记录上的更新/删除需要串行执行的约束。

http://www.cnblogs.com/zhaoyl/p/4121010.html