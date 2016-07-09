---
title: centos安装最新版本git
date: 2016-05-13 15:09:49
tags: Java
---

在使用jenkins的时候，写了一个plugin，由于git版本的问题，出现莫名其妙的问题，但是官方的最新版本只有1.7，咋办？只能考虑源码安装了
步骤如下
1. 下载编译工具
```bash
yum groupinstall “Development Tools”
```
2. 下载依赖包
```bash
yum install zlib-devel perl-ExtUtils-MakeMaker asciidoc xmlto openssl-devel
```
3. 下载 git 最新版本的源代码
```bash
wget http://www.codemonkey.org.uk/projects/git-snapshots/git/git-latest.tar.gz
```
4. 解压源文件
```bash
tar -zxvf git-latest.tar.gz
```
5. 编译安装
```bash
autoconf
 ./configure
 make -jn && make -jn install
 ## 其中make -j n中的n为指定线程数，对于多核处理器这样可以加快编译安装的速度
 ```
6. 添加link
```bash
ln -s /usr/local/bin/git /usr/bin/
```
7. 检查版本号
```bash
git --version
```