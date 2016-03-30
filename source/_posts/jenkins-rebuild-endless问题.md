---
title: jenkins_rebuild_endless问题
date: 2016-03-22 14:02:10
tags: Java
---
问题描述： 当git的branch配置成"*/dev"的时候， 构建的时候会出现多个后续版本
```bash
15:03:54 Multiple candidate revisions
15:03:54 Scheduling another build to catch up with Test_Git
```
然后就会无限的自动重复构建。
问题分析：从构建日志中可以看到：
```bash
> /usr/local/bin/git rev-parse refs/remotes/origin/dev^{commit} # timeout=10
> /usr/local/bin/git rev-parse refs/remotes/origin/origin/dev^{commit} # timeout=10
```
merge的时候存在两个有歧义的分支: refs/remotes/origin/dev和refs/remotes/origin/origin/dev
而用*/dev这种通配符形式的branch，就会出现上面的描述的问题

解决办法：
1. 直接使用“dev”而不是“*/dev”
2. 使用“/resf/heads/dev”这种最安全的方法。 Git client plugin文档上也推荐使用这种方式。

> [JENKINS-21464](https://issues.jenkins-ci.org/browse/JENKINS-21464)
> Workaround 1:
> 	use master as branch specifier rather than */master
> Workaround 2:
> 	force recloning the jenkins repo by triggering a build with wipe out & force clone option turned on then you can even remove the wipe out option