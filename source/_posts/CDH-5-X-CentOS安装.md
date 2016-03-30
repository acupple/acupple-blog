---
title: CDH_5.X_CentOS安装
date: 2016-03-16 22:54:07
tags: 大数据
---
## 概述
Cloudera企业级数据中心的安装主要分为4个步骤：
1. 集群服务器配置，包括安装操作系统、关闭防火墙、同步服务器时钟等；
2. 安装Cloudera管理器；
3. 安装CDH集群；
4. 集群完整性检查，包括HDFS文件系统、MapReduce、Hive等是否可以正常运行。

## 准备工作
1. 集群规模5个节点
```bash
Cloudera管理器节点:
    172.31.46.113
CDH节点:
    172.31.34.88
    172.31.34.89
    172.31.34.90
    172.31.34.91
```
2. 操作系统版本：CentOS 6
3. CDH版本：CDH 5.x
4. 采用root对集群进行部署

## 服务器配置
1. 安装操作系统CentOS 6
2. 如果不能连接互联网，先创建CentOS的repository，以便yum可以访问OS镜像
3. 网络配置：为了令集群中各个节点之间互相通信，需要对以下文件进行修改：
	（以cm节点为例）
/etc/sysconfig/network-scripts/ifcfg-eth0
```bash
DEVICE=eth0
ONBOOT=yes
BOOTPROTO=static
IPADDR=172.31.46.113
NETMASK=255.255.240.0
```
/etc/selinux/config
```bash
SELINUX=disabled
```
重新启动网络服务，初始化网络
```bash
$> chkconfig iptables off
$> /etc/init.d/network restart
$> init 6
```
安装需要的软件与服务
```bash
$> yum install ntp
#（如果节点不能连接互联网，需要先将CM节点设置为本地NTP服务器，
# 其他节点的NTP需要设置为与CM节点同步）
$> service ntpd start
$> chkconfig ntpd on

# Only For Cloudera管理器节点
$> yum install createrepo
$> yum install httpd
$> service httpd start
$> chkconfig httpd on
```
4.禁止交换（可选）
内存页面交换在某些情况下会导致CDH性能下降，建议关闭
```bash
$>vim/etc/sysctl.conf
#增加一行：vm.swappiness=0
$>sudo sysctl vm.swappiness=0
```
修改transparent_hugepage参数，这一参数默认值可能会导致CDH性能下降（可选）
```bash
#在/init.d/rc.local中增加一行：
$>echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag
```
## CDH软件下载与配置（Cloudera管理器节点）
1. 下载Cloudera管理器需要的rpm包
```bash
wget -c -r -nd -np -k -L -A rpmhttp://archive-
primary.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/
```
2. 下载Parcel包（包含了CDH中的Hadoop组件）
从以下地址选择合适版本的parcel包：
http://archive-primary.cloudera.com/cdh5/parcels/latest
下载manifest.json文件：
http://archive-primary.cloudera.com/cdh5/parcels/latest/manifest.json

3. 下载后将下载的软件放置为如下结构（该步骤不是必须的，只是为了后续说明的方便）
```bash
[root@ip-172-31-46-113]# ls
CDH-5.2.0-1.cdh5.2.0.p0.36-el6.parcel	cm	manifest.json
[root@ip-172-31-46-113]# ls cm
cloudera-manager-agent-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
cloudera-manager-daemons-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
cloudera-manager-server-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
cloudera-manager-server-db-2-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
enterprise-debuginfo-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
jdk-6u31-linux-amd64.rpm
oracle-j2sdk1.7-1.7.0+update67-1.x86_64.rpm
```
4. 创建repo文件以支持本地yum的操作
```bash
$> cd cm
$> createrepo .
Spawning worker 0 with 7 pkgs
Workers Finished
Gathering worker results

Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete

$> ls
cloudera-manager-agent-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
cloudera-manager-daemons-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
cloudera-manager-server-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
cloudera-manager-server-db-2-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
enterprise-debuginfo-5.2.0-1.cm520.p0.60.el6.x86_64.rpm
jdk-6u31-linux-amd64.rpm
oracle-j2sdk1.7-1.7.0+update67-1.x86_64.rpm
repodata
```
执行完后，在cm目录下生成目录repodata

5.将文件移动到特定的目录，确保可以通过HTTP协议进行访问
```bash
$> ls
CDH-5.2.0-1.cdh5.2.0.p0.36-el6.parcel  cm  manifest.json
$> mkdir -p /var/www/html/cdh5/parcels/5.2.0/
$> mv CDH-5.2.0-1.cdh5.2.0.p0.36-el6.parcel /var/www/html/cdh5/parcels/5.2.0/
$> mv manifest.json /var/www/html/cdh5/parcels/5.2.0/
$> mv cm /var/www/html/
$> chmod -R ugo+rX /var/www/html
```
现在，我们可以使用浏览器对相关目录进行访问：
http://172.31.46.113/cdh5/parcels/5.2.0/
http://172.31.46.113/cm/

6.新建文件/etc/yum.repos.d/myrepo.repo
```bash
[myrepo]
name=repo
baseurl=http://172.31.46.113/cm/
enabled=true
gpgcheck=false
```

## 安装Cloudera管理器
1.安装JDK
```
yum install oracle-j2sdk1.7
```
2.安装Cloudera管理器服务器
```
yum install cloudera-manager-daemons cloudera-manager-server
```
3.安装内置数据库
```
yum installcloudera-manager-server-db-2
```
4.启动内置数据库
```
service cloudera-scm-server-db start
```
5.启动Cloudera管理器服务器
```
service cloudera-scm-server start
```
启动后就可以访问Cloudera管理器页面了, Cloudera管理器的监听端口为7180
http://172.31.46.113:7180/smf/login


