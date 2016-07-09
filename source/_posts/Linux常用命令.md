---
title: Linux常用命令
date: 2016-04-06 11:21:15
tags: 大数据
---

## 端口占用查看
```bash
$>netstat –apn | grep 8080
```
## 磁盘占用情况
```bash
$>df -h
$>du -h --max-depth=1 /
```
## 关闭页交互空间
```bash
##方法1
$>vim /etc/sysctl.conf
$>sudo sysctl vm.swappiness=0
##方法2
$>vim /etc/fstab
##注释掉LABEL_lswap行
$>swapoff -a
```
## 查看rpm安装软件的版本
```bash
rpm -qa |grep xxxx
```
## 卸载rpm安装的软件
```bash
rpm -e xxxx ##通过-qa查找到的版本
```
## yum 查找版本
```bash
yum info xxx
```