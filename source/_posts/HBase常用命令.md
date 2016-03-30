---
title: HBase常用命令
date: 2016-03-23 16:19:53
tags: 大数据
---
1. 进入Hbase Shell
```bash
$HBASE_HOME/bin/hbase shell
```
2. 查看表
```bash
hbase(main)> list
```
3. 查看表结构
```bash
# 语法：describe <table>
# 例如：查看表t1的结构
hbase(main)> describe 't1'
```
4. 创建表结构
```bash
# 语法：create <table>, {NAME => <family>, VERSIONS => <VERSIONS>}
# 例如：创建dashcam_rawlog表，并创建两个列族，名称为 tag 和 content
create 'dashcam_rawlog', 'content', 'tag'
```
5. 修改表结构
```bash
# 语法：alter 't1', {NAME => 'f1'}, {NAME => 'f2', METHOD => 'delete'}，修改表结构必须先disable
# 例如：修改表test1的cf的TTL为180天
hbase(main)> disable 'test1'
hbase(main)> alter 'test1',{NAME=>'body',TTL=>'15552000'},{NAME=>'meta', TTL=>'15552000'}
hbase(main)> enable 'test1'
```
6. 查询数据
```bash
# 语法：scan <table>, {COLUMNS => [ <family:column>,.... ], LIMIT => num}
# 另外，还可以添加STARTROW、TIMERANGE和FITLER等高级功能
# 例如：扫描表t1的前5条数据
hbase(main)> scan 't1',{LIMIT=>5}
 
# 扫描所有dashcam_rawlog的所有行，慎用
scan 'dashcam_rawlog'
 
# 获取rowkey为\xEF\xAB\x37的行，注意，一定要用双引号来扩起rowkey！！！
get 'dashcam_rawlog', "\xEF\xAB\x37"
```
7. 删除表
```bash
# 两步：首先disable，然后drop
# 例如：删除表t1
hbase(main)> disable 't1'
hbase(main)> drop 't1'
```
8. 更改TTL
```bash
disable 'dashcam_host'
alter 'dashcam_host', NAME=>'hostinfo', TTL=>'604800'
enable 'dashcam_host'
```
9. 添加数据
```bash
# 语法：put <table>,<rowkey>,<family:column>,<value>,<timestamp>
# 例如：给表t1的添加一行记录：rowkey是rowkey001，family name：f1，column name：col1，value：value01，timestamp：系统默认
hbase(main)> put 't1','rowkey001','f1:col1','value01'
用法比较单一。
```
10. 删除数据
```bash
# a.删除行中的某个列值
# 语法：delete <table>, <rowkey>,  <family:column> , <timestamp>,必须指定列名
# 例如：删除表t1，rowkey001中的f1:col1的数据
hbase(main)> delete 't1','rowkey001','f1:col1'
 
# b.删除整行（所有版本数据）
# 语法：deleteall <table>, <rowkey>, <family:column> , <timestamp>，可以不指定列名，删除整行数据
# 例如：删除表t1，rowk001的数据
hbase(main)> deleteall 't1','rowkey001'
```
11. 清空表数据
```bash
# 语法： truncate <table># 其具体过程是：disable table -> drop table -> create table
# 例如：删除表t1的所有数据
hbase(main)> truncate 't1'
```
12. 查询表中数据行数
```bash
# 语法：count <table>, {INTERVAL => intervalNum, CACHE => cacheNum}
# INTERVAL设置多少行显示一次及对应的rowkey，默认1000；CACHE每次去取的缓存区大小，默认是10，调整该参数可提高查询速度
# 例如，查询表t1中的行数，每100条显示一次，缓存区为500
hbase(main)> count 't1', {INTERVAL => 100, CACHE => 500}
```