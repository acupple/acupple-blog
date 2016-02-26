---
title: MySQL连接问题123
date: 2016-02-26 10:52:07
tags: 数据库
---

1. 首先要排查网络问题和防火墙的问题
这个是必须的， 你要是MySQL的服务器都连不上，那还访问什么？ 怎么检查呢？
ping 192.168.0.11，如果ping 的通的话， 再去检查一下 3306端口是不是被防火墙给挡掉了，telnet 192.168.0.11 3306 或者干脆把防火墙关掉，
service iptables stop (Redhat) 或 ufw disable(ubuntu) 这一步没问题的话， 开始下一步。

2. 要排查有没有访问权限
说到访问权限， MySQL分配用户的时候会指定一个host, 比如我的 host指定为 192.168.0.5 , 那么这个账号就只能用 .5 这一台机器问，其他的机器用这个账号访问会提示没有权限。 host 指定为 % 则表示允许所有的机器访问。 一般来说出于安全方面的考虑，遵循最小权限原则， 权限的问题就不多讲了， 不会的自己查手册。可以使用如下命令：
{% codeblock %}
mysql>use mysql;
mysql>update user set host = '%' where user ='root';
mysql>select host, user from user;
{% endcodeblock %}
如果执行上述命令出现：ERROR 1062 (23000):Duplicate entry'%-root' for key 'PRIMARY' 错误，
说明host已经有了%这个值，所以直接运行命令：
{% codeblock %}
mysql>flushprivileges;
{% endcodeblock %}

确定了权限没问题的话进行下一步。
或者直接使用如下命令: grant all privileges on *.* to sa @"%" identified by "12345";为-usa -p12345用户开通所有机器的访问权限。

3. 要排查MySQL的配置
检查mysql的配置文件， Linux下MySQL的配置文件叫 my.cnf，windows下的叫 my.ini，检查这个配置项–bind-address=IP
引用手册里的一段话：
	*The IP address to bind to.Only one address can be selected. If this option is specified multiple times,the last address given is used. If no address or 0.0.0.0 is specified, the server listens on all interfaces.*
绑定的IP， 只能绑定一个IP， 如果绑定多个IP, 则以最后一个绑定的为准。如果没有绑定或绑定 0.0.0.0, 服务器监听所有的客户端。

4. Mysql忘记密码怎么办
利用vim命令打开mysql配置文件my.cnf
在mysqld进程配置文件中添加skip-grant-tables,添加完成后,执行wd保存
重启mysql server然后就可以不用密码直接登录mysql了，登录进去后
执行以下命令修改root密码：
{% codeblock %}
mysql>update mysql.user set password=password('newpassword') where user='root';
{% endcodeblock %}
#将password()中的newpassword字符更改为你自己的密码。
密码修改完成后，将my.cnf文件中添加的skip-grant-tables语句注释或删除掉，然后重启数据库即可。