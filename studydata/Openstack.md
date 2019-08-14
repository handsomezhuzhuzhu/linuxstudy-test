## 一、基础环境准备

### - 1.1：虚拟机配置

- contos7.2-1511最小化安装，NATx2+仅主机x2=4块网卡，绑定为两块网卡

## 二、openstack环境准备

### - 2.1：openstack介绍

- 安装指南：<https://docs.openstack.org/ocata/install-guide-rdo/>

### - 2.2：版本介绍

```bash
#Alpha ：是内部测试版, 一般不向外部发布，通常只在软件开发者内部交流，该版本软件的Bug较多，需要继续修改。

#Dev ：在软件开发中多用于开发软件的代号，相比于beta版本，dev版本可能出现的更早，甚至还没有发布。这也就意味着，dev版本的软件通常比beta版本的软件更不稳定

#Beta ：也是测试版，这个阶段的版本会一直加入新的功能。在Alpha版之后推出。

#RC ：(Release Candidate)  就是发行候选版本,RC版不会再加入新的功能了，主要着重于除错。

#GA:General Availability, 正式发布的版本。

#Release: 该版本意味“最终版本”，在前面版本的一系列测试版之后，终归会有一个正式版本，是最终交付用户使用的一个版本。该版本有时也称为标准版
```

### - 2.3：各组件的功能

- 查看openstack yum版本

```bash
[root@control1 ~]# yum list openstack*
```

- 各服务器安装ocata的yum源

```bash
# 负载服务、数据库、memcache 、rabbitMQ服务器除外，即只安装openstack控制节点和计算节点
[root@control1 ~]# yum install -y centos-release-openstack-ocata.noarch
```

- 各服务器安装openstack客户端

```bash
[root@control1 ~]# yum install -y python-openstackclient
```

- 各服务器安装openstack SElinux管理包

```bash
#如果agent开启了selinux会自动进行selinux权限的相关设置，若selinux关闭可不用装
[root@control1 ~]# yum install openstack-selinux -y
```

### - 2.4：安装数据库服务器

- 需要先在控制端安装包

```bash
#用于控制端连接数据库
[root@control1 ~]# yum install -y mariadb python2-PyMySQL
```

- 安装mariadb-server

```bash
[root@centos ~]# yum install mariadb-server -y
[root@centos ~]# systemctl start mariadb
[root@centos ~]# mysql_secure_installation
```

- 配置数据库

```bash
[root@centos ~]# vim /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 192.168.172.5 #指定监听地址
default-storage-engine = innodb #默认引擎
innodb_file_per_table = on #开启每个表都有独立表空间
max_connections = 4096 #最大连接数
collation-server = utf8_general_ci #不区分大小写排序
character-set-server = utf8 #设置编码
```

- 配置my.cnf（若是做mysql主从则添加）

```bash
[root@centos ~]# vim /etc/my.cnf
[mysqld]
socket=/var/lib/mysql/mysql.sock
user=mysql
symbolic-links=0
datadir=/data/mysql
innodb_file_per_table=1
#skip-grant-tables
relay-log = /data/mysql
server-id=10
log-error= /data/mysql-log/mysql_error.txt
log-bin=/data/mysql-binlog/master-log
#general_log=ON
#general_log_file=/data/general_mysql.log
long_query_time=5
slow_query_log=1
slow_query_log_file= /data/mysql-log/slow_mysql.txt
max_connections=1000
bind-address=192.168.172.5
[client]
port=3306
socket=/var/lib/mysql/mysql.sock
[mysqld_safe]
log-error=/data/mysql-log/mysqld-safe.log
pid-file=/var/lib/mysql/mysql.sock

```

- 创建数据目录并授权

```bash
[root@centos ~]# mkdir -pv /data/{mysql,mysql-log,mysql-binlog}
[root@centos ~]# chown -R mysql.mysql /data/

```

- 重启mariadb并验证

```bash
[root@centos ~]# systemctl restart mariadb
[root@centos ~]# mysql -uroot -pcentos
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 10.3.10-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>

```

### - 2.5：部署keepalived+haproxy

- 部署在ubuntu系统上，ubuntu修改网卡名为eth0，

```bash
zhuzhuzhu@ubuntu1804:~$ sudo -i #切换到root用户
root@ubuntu1804:~# vim /etc/default/grub
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
root@ubuntu1804:~# vim /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: yes
#ubuntu使网卡配置生效的命令是netplan apply
#ubuntu的yum源，使用阿里云源
root@ubuntu1804:~# vim /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
root@ubuntu1804:~# reboot

```

- ubuntu上安装haproxy+keepalived(若是centos系统，切记keepalived加上vrrp_iptables)

```bash
root@ubuntu1804:~# apt-get install keepalived haproxy -y
root@ubuntu1804:~# find / -name keepalived.*
root@ubuntu1804:~# cp /usr/share/doc/keepalived/samples/keepalived.conf.sample /etc/keepalived/keepalived.conf
#配置keepalived的步骤不再赘述


#haproxy的内核配置参数
root@ubuntu1804:~# vim /etc/sysctl.conf
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
root@ubuntu1804:~# sysctl -p
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1

```

- 启动服务并验证

```bash
root@ubuntu1804:~# systemctl start haproxy
root@ubuntu1804:~# systemctl enable haproxy
root@ubuntu1804:~# ss -nlt
State      Recv-Q      Send-Q                Local Address:Port             Peer Address:Port      
LISTEN     0           128                 192.168.172.248:3306   
...

```

### - 2.6：安装rabbotMQ服务器

#### - 2.6.1：单个rabbitMQ服务部署

- 各组件通过消息发送与接收来实现组件之间的通信

```bash
[root@centos ~]# yum install rabbitmq-server -y
[root@centos ~]# systemctl start rabbitmq-server
[root@centos ~]# systemctl enable rabbitmq-server
Created symlink from /etc/systemd/system/multi-user.target.wants/rabbitmq-server.service to /usr/lib/systemd/system/rabbitmq-server.service.

#验证端口5672
[root@centos ~]# ss -nlt
State       Recv-Q Send-Q    Local Address:Port                   Peer Address:Port              
LISTEN      0      128                   *:25672                             *:*                  
LISTEN      0      168       192.168.172.4:3306                              *:*                  
LISTEN      0      128                   *:4369                              *:*                  
LISTEN      0      128                   *:22                                *:*                  
LISTEN      0      100           127.0.0.1:25                                *:*                  
LISTEN      0      128                  :::5672                             :::*                  
LISTEN      0      128                  :::22                               :::*                  
LISTEN      0      100                 ::1:25                               :::*          

```

- 添加rabbitMQ客户端用户并设置密码

```bash
[root@centos ~]# rabbitmqctl add_user openstack centos #用户名openstack，密码centos
Creating user "openstack"

```

- 赋予opstack用户读写权限

```bash
[root@centos ~]# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/"

```

- 打开rebbitMQ的web插件

```bash
[root@centos ~]# rabbitmq-plugins enable rabbitmq_management
The following plugins have been enabled:
  amqp_client
  cowlib
  cowboy
  rabbitmq_web_dispatch
  rabbitmq_management_agent
  rabbitmq_management

Applying plugin configuration to rabbit@centos... started 6 plugins.
[root@centos ~]# rabbitmq-plugins list #查看插件
...
[root@centos ~]# ss -nlt
State       Recv-Q Send-Q    Local Address:Port                   Peer Address:Port              
LISTEN      0      128                   *:25672                             *:*                  
LISTEN      0      168       192.168.172.4:3306                              *:*                  
LISTEN      0      128                   *:4369                              *:*                  
LISTEN      0      128                   *:22                                *:*                  
LISTEN      0      1024                  *:15672                             *:*                  
LISTEN      0      100           127.0.0.1:25                                *:*                  
LISTEN      0      128                  :::5672                             :::*                  
LISTEN      0      128                  :::22                               :::*                  
LISTEN      0      100                 ::1:25                               :::*  

```

- rabbitMQ各端口的作用
  - 客户端实用         5672
  - web访问端口     15672
  - 集群通信使用     25672
- 访问rabbitMQ的web界面

默认用户名密码都是guest，可以更改，web访问端口15672（打开插件后此端口自动打开）

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1560926481609.png)

#### - 2.6.2：部署rabbitMQ集群

Rabbitmq的集群是依赖于erlang的集群来工作的，所以必须先构建起erlang的集群环境。而Erlang的集群中各节点是通过一个magic cookie来实现的，这个cookie存放在 /var/lib/rabbitmq/.erlang.cookie 中，文件是400的权限

- 前提条件
  - 必须保证各节点cookie保持一致
  - 必须能解析对方的主机名，换言之若使用域名则必须配置本地解析
  - 从节点数据必须清空才可加入主节点
- 参考作品：<http://blogs.studylinux.net/?p=4266>

```bash
#首先设置相同的cookie值，无论复制主、从服务器的cookie都可以
[root@centos ~]# cat /var/lib/rabbitmq/.erlang.cookie
WYFSFWJASCIOEYGPYZBE
[root@centos ~]# scp /var/lib/rabbitmq/.erlang.cookie 192.168.172.*：/var/lib/rabbitmq/

#停止所有节点RabbitMq服务，然后使用detached参数独立运行，这步很关键，尤其增加节点停止节点后再次启动遇到无法启动都可以参照这个顺序
[root@rabbitmq-server1 ~]# systemctl  stop  rabbitmq-server
[root@rabbitmq-server2 ~]# systemctl  stop  rabbitmq-server
[root@rabbitmq-server3 ~]# systemctl  stop  rabbitmq-server
[root@rabbitmq-server1 ~]# rabbitmq-server -detached
[root@rabbitmq-server2 ~]# rabbitmq-server -detached
[root@rabbitmq-server3 ~]# rabbitmq-server -detached

#清空元数据
[root@rabbitmq-server1 ~]# rabbitmqctl   reset 

#未添加到集群之前只有自己一台节点
[root@rabbitmq-server1 ~]# rabbitmqctl  cluster_status

#将rabbitmq-server1添加到集群当中，并成为内存节点，不加--ram默认是磁盘节点
[root@rabbitmq-server1 ~]# rabbitmqctl  join_cluster rabbit@rabbitmq-server2 --ram　

#更改为镜像模式，"#"为任意0个或多个即为所有，也可以使用"^test"匹配开头，还可以使用其他正则匹配
[root@rabbitmq-server1 ~]# rabbitmqctl set_policy  ha-all "#"  '{"ha-mode":"all"}' 

```

### - 2.7：安装memcached

用于缓存openstack各服务的身份认证令牌信息

- 安装memcached

```bash
[root@centos ~]# yum install memcached -y
[root@centos ~]# vim /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="512"
OPTIONS="-l 0.0.0.0,::1"

#启动服务
[root@centos ~]# systemctl start memcached
[root@centos ~]# systemctl enable memcached
Created symlink from /etc/systemd/system/multi-user.target.wants/memcached.service to /usr/lib/systemd/system/memcached.service.

#验证端口11211
[root@centos ~]# ss -nlt
State       Recv-Q Send-Q    Local Address:Port                   Peer Address:Port              
LISTEN      0      128                   *:25672                             *:*                  
LISTEN      0      168       192.168.172.4:3306                              *:*                  
LISTEN      0      1024    192.168.172.248:11211                             *:*                  
LISTEN      0      128                   *:4369                              *:*                  
LISTEN      0      128                   *:22                                *:*                  
LISTEN      0      1024                  *:15672                             *:*                  
LISTEN      0      100           127.0.0.1:25                                *:*                  
LISTEN      0      128                  :::5672                             :::*                  
LISTEN      0      128                  :::22                               :::*                  
LISTEN      0      100                 ::1:25                               :::*  

```

- 控制端安装通信包

```bash
[root@control1 ~]# yum install python-memcached -y

```

## 三、部署认证服务keystone

### - 3.1：keystone介绍

keystone中主要涉及以下几个概念：User、Tenant、Role、Token

- User：使用openstack的用户
- Tenant：租户，可以理解为一个人、项目或者组织拥有的资源的合集，在一个租户中可以拥有很多个用户，这些用户可以根据权限的划分使用租户中的资源
- Role：角色，用于分配操作的权限，角色可以被指定给用户，是的该用户获得角色对应的操作权限
- Token：指的是一串比特值或者字符串，用来补作为访问资源的记号。Token中含有可访问资源的范围和有效时间

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1560950863682.png)

### - 3.2：keystone数据库配置

- 创建数据库

```bash
#在mysql服务器上创建
[root@centos ~]# mysql -uroot -pcentos
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 10.3.10-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database keystone;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> grant all on keystone.* to 'keystone'@'%' identified by 'centos';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> 

```

- 验证数据库

```bash
#验证可以从openstack控制端使用keyston访问数据库
[root@control1 ~]# mysql -ukeystone -pcentos -h192.168.172.248
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 10.3.10-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.003 sec)

```

- 配置haproxy代理

```bash
root@ubuntu1804:~# vim /etc/haproxy/haproxy.cfg
#======================================================
listen openstack_mysql
bind 192.168.172.248:3306
mode tcp
server 192.168.172.5 192.168.172.5:3306 check
#======================================================
listen openstack_memcached
bind 192.168.172.248:11211
mode tcp
server 192.168.172.5 192.168.172.5:11211 check
#======================================================
listen openstack_rabbitmq
bind 192.168.172.248:5672
mode tcp
server 192.168.172.5 192.168.172.5:5672 check

#重启服务验证端口
root@ubuntu1804:~# systemctl restart haproxy
root@ubuntu1804:~# ss -nlt
State      Recv-Q      Send-Q               Local Address:Port              Peer Address:Port      
LISTEN     0           128                192.168.172.248:3306                   0.0.0.0:*         
LISTEN     0           128                192.168.172.248:11211                  0.0.0.0:*         
LISTEN     0           128                192.168.172.248:5672                   0.0.0.0:*  
...

#验证vip访问memcached
[root@control1 ~]# telnet 192.168.172.248 11211
Trying 192.168.172.248...
Connected to 192.168.172.248.
Escape character is '^]'.

```

### - 3.3：安装keystone

```bash
#openstack-keystone是keystone服务，http是web服务，mod_wsgi是python的通用网关
[root@control1 ~]# yum install -y openstack-keystone httpd mod_wsgi python-memcached

```

- 编辑keystone配置文件

```bash
[root@control1 ~]# openssl rand -hex 10 #生成临时token
3ade64c5dfbaefb7915e
[root@control1 ~]# vim /etc/keystone/keystone.conf
713 connection = mysql+pymysql://keystone:centos@192.168.172.248/keystone #//账户：密码@VIP（或域名）/数据库名称
17 admin_token = 3ade64c5dfbaefb7915e
675 provider = fernet

```

- 初始化并验证数据库

```bash
#会在数据库创建默认表等操作
[root@control1 ~]# su -s /bin/sh -c "keystone-manage db_sync" keystone

#在数据库查看是否生成表
[root@centos mnesia]# mysql -uroot -pcentos
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 19
Server version: 10.3.10-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use keystone
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [keystone]> show tables;
+------------------------------------+
| Tables_in_keystone                 |
+------------------------------------+
| access_rule                        |
| access_token                       |
| application_credential             |
| application_credential_access_rule |
| application_credential_role        |
| assignment                         |
...
38个表
#查看keystone日志文件
[root@control1 ~]# ll /var/log/keystone/keystone.log 
-rw-rw---- 1 root keystone 76746 6月  20 01:01 /var/log/keystone/keystone.log

```

- 初始化证书并验证

```bash
[root@control1 ~]# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
[root@control1 ~]# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
[root@control1 ~]# ll /etc/keystone/fernet-keys/
总用量 8
-rw------- 1 keystone keystone 44 6月  19 17:16 0
-rw------- 1 keystone keystone 44 6月  19 17:16 1

```

### - 3.4：配置keystone

通过apache代理python

- 编辑apache配置文件

```bash
[root@control1 ~]# vim /etc/httpd/conf/httpd.conf
95 ServerName 192.168.172.4:80

```

- 软链接配置文件

```bash
[root@control1 ~]# vim /usr/share/keystone/wsgi-keystone.conf
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

[root@control1 ~]# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

```

- 启动apache服务并验证端口

```bash
[root@control1 ~]# systemctl start httpd
[root@control1 ~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.

#验证端口
[root@control1 ~]# ss -nlt
State       Recv-Q Send-Q    Local Address:Port                   Peer Address:Port                 
LISTEN      0      511                  :::5000                             :::*                  
LISTEN      0      511                  :::80                               :::*                  
LISTEN      0      511                  :::35357                            :::* 

```

### - 3.5：创建域、用户、项目和角色

- 通过admin的token设置环境变量进行操作（复制一个新的ssh渠道来执行）

```bash
[root@control1 ~]# export OS_TOKEN=3ade64c5dfbaefb7915e
[root@control1 ~]# export OS_URL=http://192.168.172.4:35357/v3
[root@control1 ~]# export OS_IDENTITY_API_VERSION=3

```

- 创建默认域

一定要在上一步设置完成环境变量的前提下方可操作成功，否则会提示未认证

```bash
#命令格式：openstack domain create --description " 描述信息"  域名
[root@control1 ~]# openstack domain create --description "Default Domain" default
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Default Domain                   |
| enabled     | True                             |
| id          | ff3fa2a78082485ba524f9273fb6f106 |
| name        | default                          |
| tags        | []                               |
+-------------+----------------------------------+

```

- 创建一个admin的项目

```bash
#命令格式： openstack project --domain  域 --description " 描述" 项目名
[root@control1 ~]# openstack project create --domain default --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| domain_id   | ff3fa2a78082485ba524f9273fb6f106 |
| enabled     | True                             |
| id          | 96bb8e810de74cd082fb8b6c235824b7 |
| is_domain   | False                            |
| name        | admin                            |
| parent_id   | ff3fa2a78082485ba524f9273fb6f106 |
| tags        | []                               |
+-------------+----------------------------------+

```

- 创建admin用户并设置密码为admin

```bash
[root@control1 ~]# openstack user create --domain default --password-prompt admin
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | ff3fa2a78082485ba524f9273fb6f106 |
| enabled             | True                             |
| id                  | 833c4226afd14c0697d0c17587ba214f |
| name                | admin                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

- 创建admin角色

一个项目里面可以有多个角色，目前角色只能创建在/etc/keystone/policy.json 文件中定义好的角色

```bash
[root@control1 ~]# openstack role create admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | 3584b80c46ec4bee8e6834f706a0b78a |
| name        | admin                            |
+-------------+----------------------------------+

```

- 给admin用户授权

将admin用户授予admin项目的admin角色，即给admin项目添加一个用户叫admin，并将其添加至admin角色，角色是权限的一种集合

```bash
[root@control1 ~]# openstack role add --project admin --user admin admin

```

### - 3.6：创建demo项目

该项目可用于演示和测试等

- 创建demo项目

```bash
[root@control1 ~]# openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | ff3fa2a78082485ba524f9273fb6f106 |
| enabled     | True                             |
| id          | 980cd5b150c44aaba175a8f4ecd20f07 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | ff3fa2a78082485ba524f9273fb6f106 |
| tags        | []                               |
+-------------+----------------------------------+

```

- 创建demo用户并设置密码为demo

```bash
[root@control1 ~]# openstack user create --domain default --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | ff3fa2a78082485ba524f9273fb6f106 |
| enabled             | True                             |
| id                  | 47c94047e0284bfaa3fa54f8be539af1 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

- 创建一个user角色

角色目前有user和admin

```bash
[root@control1 ~]# openstack role create user
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | 25b8aa832c0a42fcb5894d01fc06d2b3 |
| name        | user                             |
+-------------+----------------------------------+

```

- 把demo用户添加到demo项目然后赋予user权限

```bash
[root@control1 ~]# openstack role add --project demo --user demo user

```

### - 3.7：创建一个service项目

各服务之间与keystone进行访问和认证，service用于给服务创建用户

- 创建service项目

```bash
[root@control1 ~]# openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | ff3fa2a78082485ba524f9273fb6f106 |
| enabled     | True                             |
| id          | 50c7fd61e6b84a62915e0f78dc6dcb09 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | ff3fa2a78082485ba524f9273fb6f106 |
| tags        | []                               |
+-------------+----------------------------------+

```

### - 3.8：服务注册

将keystone服务地址注册到openstart

- 创建一个keystone认证服务

```bash
[root@control1 ~]# openstack service list #查看当前服务
[root@control1 ~]# openstack service create --name keystone --description "Openstack Identity" identity
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Openstack Identity               |
| enabled     | True                             |
| id          | c9de2eee565148ae92c15f9f484de1c2 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
[root@control1 ~]# openstack service list #验证服务创建成功与否
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| c9de2eee565148ae92c15f9f484de1c2 | keystone | identity |
+----------------------------------+----------+----------+

```

- 创建endpoint

如果创建错误或多创建了，就要全部删除再重新注册，因为不知道具体那一个是对的哪一个是错误的，所以只能全部删除然后重新注册，注册的IP地址写keepalived的VIP，稍后配置haproxy

```bash
[root@control1 ~]# openstack endpoint create --region RegionOne identity public http://192.168.172.248:5000/v3 #公共端点
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | f859ac1133df462182106434c90b95d7 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | c9de2eee565148ae92c15f9f484de1c2 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.172.248:5000/v3   |
+--------------+----------------------------------+

[root@control1 ~]# openstack endpoint create --region RegionOne identity internal http://192.168.172.248:5000/v3 #私有端点
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | df8dad31228b414684718c0dad1aba32 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | c9de2eee565148ae92c15f9f484de1c2 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.172.248:5000/v3   |
+--------------+----------------------------------+

[root@control1 ~]# openstack endpoint create --region RegionOne identity admin http://192.168.172.248:35357/v3 #管理端点
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | f3a05f01640745948a6a07f246886093 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | c9de2eee565148ae92c15f9f484de1c2 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.172.248:35357/v3  |
+--------------+----------------------------------+

```

- 配置haproxy

```bash
root@ubuntu1804:~# vim /etc/haproxy/haproxy.cfg 
...
#============================================================
listen keystone_public_url
bind 192.168.172.248:5000
mode tcp
server 192.168.172.4 192.168.172.4:5000 check
#============================================================
listen keystone_admin_url
bind 192.168.172.248:35357
mode tcp
server 192.168.172.4 192.168.172.4:35357 check

```

- 重启并验证访问

```bash
root@ubuntu1804:~# telnet 192.168.172.4 5000
Trying 192.168.172.4...
Connected to 192.168.172.4.
Escape character is '^]'.

root@ubuntu1804:~# telnet 192.168.172.4 35357
Trying 192.168.172.4...
Connected to 192.168.172.4.
Escape character is '^]'.

```

- 测试keystone是否可以做用户验证

验证admin用户，密码admin，新打开一个窗口并进行以下操作：

```bash
[root@control1 ~]# export OS_IDENTITY_API_VERSION=3
[root@control1 ~]# openstack --os-auth-url http://192.168.172.4:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
...

#验证demo用户，密码为demo
[root@control1 ~]# openstack --os-auth-url http://192.168.172.4:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
Password: 
...

```

- 使用脚本设置环境变量

```bash
#admin用户脚本内容
[root@control1 ~]# vim admin-ocata.sh
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.172.4:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
[root@control1 ~]# chmod a+x admin-ocata.sh

#demo用户脚本内容
[root@control1 ~]# vim demo-ocata.sh
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://192.168.172.4:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
[root@control1 ~]# chmod a+x demo-ocata.sh

```

- 测试脚本是否可以正常使用

```bash
#admin用户脚本测试
[root@control1 ~]# source admin-ocata.sh
[root@control1 ~]# openstack --os-auth-url http://192.168.172.4:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
...

#demo用户脚本验证
[root@control1 ~]# source demo-ocata.sh
[root@control1 ~]# openstack --os-auth-url http://192.168.172.4:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
...

```

## 四、部署镜像服务glance

Glance 是 OpenStack 镜像服务组件，glance 服务默认监听在 9292 端口，其接收 REST API 请求，然后通过其他模块（glance-registry 及 image store）来完成诸如镜像的获取、上传、删除等操作，Glance 提供 restful API 可以查询虚拟机镜像的 metadata，并且可以获得镜像，通过Glance，虚拟机镜像可以被存储到多种存储上，比如简单的文件存储或者对象存储（比如OpenStack 中 swift 项目）是在创建虚拟机的时候，需要先把镜像上传到 glance，对镜像的列出镜像、删除镜像和上传镜像都是通过 glance 进行理

- glance的两个主要服务
  - glance-api：监听端口9292，接收镜像的删除、上传和读取
  - glance-Registry：监听端口9191，负责与mysql数据交互，用于存储或获取镜像的元数据（metadata），提供镜像元数据相关的REST接口，通过glance-registry可以向数据库中写入或获取镜像的各种数据，glance 数据库中有两张表，一张是 glance 表，一张是 imane property 表，image 表保存了镜像格式、大小等信息，image property 表保存了镜像的定制化信息
- image store 是一个存储的接口层，通过这个接口 glance 可以获取镜像，image store 支持的存储有 Amazon 的 S3、openstack 本身的 swift、还有 ceph、glusterFS、sheepdog 等分布式存储，image store 是镜像保存与读取的接口，但是它只是一个接口，具体的实现需要外部的支持，glance 不需要配置消息队列，但是需要配置数据库和 keystone

### - 4.1：控制端安装glance

```bash
[root@control1 ~]# yum install openstack-glance -y

```

### - 4.2：创建glance数据库并初始化

```bash
[root@centos ~]# mysql -uroot -pcentos 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 86
Server version: 10.3.10-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database glance;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> grant all on glance.* to 'glance'@'%' identified by 'centos';
Query OK, 0 rows affected (0.062 sec)

```

- 控制端登录验证

```bash
[root@control1 ~]# mysql -uglance -pcentos -h 192.168.172.248
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 87
Server version: 10.3.10-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| glance             |
| information_schema |
+--------------------+
2 rows in set (0.005 sec)

```

### - 4.3：创建glance账户

创建glance密码用户并设置密码为glance

```bash
[root@control1 ~]# openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | ff3fa2a78082485ba524f9273fb6f106 |
| enabled             | True                             |
| id                  | d103b6c29e3a4b1bb02b11378dc14014 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

- 对glance用户授权

把glance和neutron用户添加到service项目并授予admin角色

```bash
[root@control1 ~]# openstack role add --project service --user glance admin

```

### - 4.4：编辑配置文件并同步数据库

```bash
[root@control1 ~]# vim /etc/glance/glance-api.conf
[database]
connection = mysql+pymysql://glance:centos@192.168.172.248/glance #数据库名和数据库密码

[keystone_authtoken]
auth_uri = http://192.168.172.248:5000
auth_url = http://192.168.172.248:35357
memcached_servers = 192.168.172.248:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

[root@control1 ~]# vim /etc/glance/glance-registry.conf
[database]
connection = mysql+pymysql://glance:centos@192.168.172.248/glance

[keystone_authtoken]
auth_uri = http://192.168.172.248:5000
auth_url = http://192.168.172.248:35357
memcached_servers = 192.168.172.248:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone

#创建images目录
[root@control1 ~]# mkdir /var/lib/glance/images
[root@control1 ~]# chown -R glance.glance /var/lib/glance/images
[root@control1 ~]# ll /var/lib/glance/images
总用量 0

```

- 配置haproxy端口转发

```bash
root@ubuntu1804:~# vim /etc/haproxy/haproxy.cfg
...
listen glance_api
bind 192.168.172.248:9292
mode tcp
server 192.168.172.4 192.168.172.4:9292 check
#===========================================================
listen glance
bind 192.168.172.248:9191
mode tcp
server 192.168.172.4 192.168.172.4:9191 check
#===========================================================

#重启haproxy验证
root@ubuntu1804:~# systemctl restart haproxy
root@ubuntu1804:~# ss -nlt
State      Recv-Q      Send-Q               Local Address:Port              Peer Address:Port      
LISTEN     0           128                192.168.172.248:9191                   0.0.0.0:*         
LISTEN     0           128                192.168.172.248:5000                   0.0.0.0:*         
LISTEN     0           128                192.168.172.248:5672                   0.0.0.0:*         
LISTEN     0           128                192.168.172.248:3306                   0.0.0.0:*         
LISTEN     0           128                192.168.172.248:9292                   0.0.0.0:*         
LISTEN     0           128                192.168.172.248:11211                  0.0.0.0:*         
LISTEN     0           128                  127.0.0.53%lo:53                     0.0.0.0:*         
LISTEN     0           128                        0.0.0.0:22                     0.0.0.0:*         
LISTEN     0           128                      127.0.0.1:6010                   0.0.0.0:*         
LISTEN     0           128                192.168.172.248:35357                  0.0.0.0:*         
LISTEN     0           128                           [::]:22                        [::]:*         
LISTEN     0           128                          [::1]:6010                      [::]:* 

```

- 初始化glance数据库并验证

```bash
[root@control ~]#su -s /bin/sh -c "glance-manage db_sync" glance
...
[root@control ~]#[root@localhost ~]# mysql -uroot -pcentos
MariaDB [(none)]> use glance
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [glance]> show tables;
+----------------------------------+
| Tables_in_glance                 |
+----------------------------------+
| alembic_version                  |
| artifact_blob_locations          |
| artifact_blobs                   |
| artifact_dependencies            |
| artifact_properties              |
| artifact_tags                    |
| artifacts                        |
| image_locations                  |
...

```

- 启动glance服务

```bash
[root@control1 ~]# systemctl  start  openstack-glance-api.service  openstack-glance-registry.service
[root@control1 ~]# systemctl  enable  openstack-glance-api.service  openstack-glance-registry.service
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-glance-api.service to /usr/lib/systemd/system/openstack-glance-api.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-glance-registry.service to /usr/lib/systemd/system/openstack-glance-registry.service.
[root@control1 ~]# ss -nlt
State       Recv-Q Send-Q         Local Address:Port              Peer Address:Port              
LISTEN      0      4096            *:9292                                *:*                  
LISTEN      0      128             *:22                                  *:*                  
LISTEN      0      100             127.0.0.1:25                          *:*                  
LISTEN      0      4096            *:9191                                *:*               
LISTEN      0      511             :::5000                               :::*                  
LISTEN      0      511             :::80                                 :::*                  
LISTEN      0      128             :::22                                 :::*                  
LISTEN      0      100             ::1:25                                :::*                  
LISTEN      0      511             :::35357                              :::*   

```

- glance服务注册

```bash
[root@control ~]# source admin-ocata.sh 
[root@control ~]# openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 14cadd5f8f6342539bfca2d36a576b90 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

```

- 创建endpoint

```bash
#创建公有endpoint
[root@control ~]# openstack endpoint create --region RegionOne image public http://192.168.172.248:9292

#创建私有endpoint
[root@control ~]# openstack endpoint create --region RegionOne image internal http://192.168.172.248:9292

#创建管理endpoint
[root@control ~]# openstack endpoint create --region RegionOne image admin http://192.168.172.248:9292

#验证endpoint
[root@control ~]# openstack endpoint list
...

```

- 测试glance上传镜像

```bash
[root@control ~]# wget  http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
[root@control ~]# source admin-ocata.sh
[root@control ~]# openstack image create "cirros" --file /root/cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public

```

- 验证glance镜像

```bash
[root@control ~]# glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| 1fc48636-8605-44b9-870d-9b761a1a7b53 | cirros |
+--------------------------------------+--------+
[root@control ~]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 1fc48636-8605-44b9-870d-9b761a1a7b53 | cirros | active |
+--------------------------------------+--------+--------+

#查看指定镜像信息
[root@control ~]# openstack image show cirros

```

## 五、部署nova控制节点与计算节点

nova 是 openstack 最早的组件之一，nova 分为控制节点和计算节点，计算节点通过 novacomputer 进行虚拟机创建，通过 libvirt 调用 kvm 创建虚拟机，nova 之间通信通过 rabbitMQ队列进行通信，其组件和功能如下：

- API：负责接收和响应外部请求。

```bash
Nova-api 组件实现了 restful API 的功能，接收和响应来自最终用户的计算 API 请求，接收外部的请求并通过 message queue 将请求发动给其他服务组件，同时也兼容 EC2 API，所以也可以使用 EC2 的管理工具对 nova 进行日常管理

```

- Scheduler：负责调度虚拟机所在的物理机。

```bash
nova scheduler 模块在 openstack 中的作用是决策虚拟机创建在哪个主机（计算节点）上。决策一个虚拟机应该调度到某物理节点，需要分为两个步骤：
#1.过滤（filter），过滤出可以创建虚拟机的主机。首先得到未经过滤的主机列表，然后根据过滤特性，选择符合条件的计算节点主机
#2.计算权值（weight），根据权重大进行分配，默认根据资源可用空间进行权重排序。经过主机过滤后，需要对主机进行权值的计算，根据策略选择相应的某一台主机（对于每一个要创建的虚拟机而言）

```



- Conductor：计算节点访问数据库的中间件。
- Consoleauth：用于控制台的授权认证。
- Novncproxy：VNC 代理，用于显示虚拟机操作终端

官方部署文档：https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/common/get_started_compute.html

### - 5.1：安装并配置nova控制节点

- 安装nova控制端

```bash
[root@control ~]# yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy  openstack-nova-scheduler openstack-nova-placement-api -y

```

- 准备数据库

```bash
[root@localhost ~]# mysql -uroot -pcentos 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 282
Server version: 5.5.60-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database nova_api;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'centos';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> create database nova_cell0;
Query OK, 1 row affected (0.00 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'centos';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> create database nova;
Query OK, 1 row affected (0.00 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'centos';
Query OK, 0 rows affected (0.00 sec)

```

- 验证数据库

```bash
[root@control ~]# mysql -unova -pcentos -h192.168.172.248
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 292
Server version: 5.5.60-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
| nova_api           |
| nova_cell0         |
+--------------------+
4 rows in set (0.00 sec)

```

### - 5.2：创建nova用户

将nova用户添加到service项目并授予admin权限

- 创建nova用户并设置密码为nova

```bash
[root@control1 ~]# openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | ff3fa2a78082485ba524f9273fb6f106 |
| enabled             | True                             |
| id                  | 0cadd2fbbdac4ac1a3cde2dd1076e121 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

- 为nova用户授权

```bash
[root@control1 ~]# openstack role add --project service --user nova admin

```

- 创建nova服务

```bash
[root@control ~]# openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 06b66b8181f94c02bce9231386e2b7fd |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

```

- 创建endpoint

```bash
#创建公有endpoint
[root@control ~]# openstack endpoint create --region RegionOne compute public http://192.168.172.248:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6b23daa369944f4a9bcdecded745494f |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 06b66b8181f94c02bce9231386e2b7fd |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.172.248:8774/v2.1 |
+--------------+----------------------------------+

#创建私有endpoint
[root@control ~]# openstack endpoint create --region RegionOne compute internal http://192.168.172.248:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 18b752ff7f1c4c99ae8315b3804abed0 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 06b66b8181f94c02bce9231386e2b7fd |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.172.248:8774/v2.1 |
+--------------+----------------------------------+

#创建管理endpoint
[root@control ~]# openstack endpoint create --region RegionOne compute admin http://192.168.172.248:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | db6e5d86b21a4727a59e038255ad901e |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 06b66b8181f94c02bce9231386e2b7fd |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.172.248:8774/v2.1 |
+--------------+----------------------------------+

```

- 创建placement用户并授权

```bash
#创建placement用户密码placement
[root@control ~]# openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | a7b0d0798ee74bb29795fca945b80a48 |
| enabled             | True                             |
| id                  | 84d570bee46348069b2b81f92a2837b9 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

#为placement用户授予admin权限
[root@control ~]# openstack role add --project service --user placement admin

```

- 创建placement API并注册

```bash
[root@control ~]# openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | dcf04807a9bc4f40b6987d88ae320b0b |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

#创建公有placement
[root@control ~]# openstack endpoint create --region RegionOne placement public http://192.168.172.248:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7dcd82bde3704fea890135c97168c23e |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | dcf04807a9bc4f40b6987d88ae320b0b |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.172.248:8778      |
+--------------+----------------------------------+

#创建私有endpoint
[root@control ~]# openstack endpoint create --region RegionOne placement internal http://192.168.172.248:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a0b32fc6e95d4dec9ad501c3a3709733 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | dcf04807a9bc4f40b6987d88ae320b0b |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.172.248:8778      |
+--------------+----------------------------------+

#创建管理endpoint
[root@control ~]# openstack endpoint create --region RegionOne placement admin http://192.168.172.248:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | df615e121b6a44598afb023b2bc84083 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | dcf04807a9bc4f40b6987d88ae320b0b |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.172.248:8778      |
+--------------+----------------------------------+

```

- 在haproxy配置端口转发

```bash
root@ubuntu1804:~# vim /etc/haproxy/haproxy.cfg
#===========================================================
listen nova_8774
bind 192.168.172.248:8774
mode tcp
server 192.168.172.4 192.168.172.4:8774 check
#===========================================================
listen nova_8778
bind 192.168.172.248:8778
mode tcp
server 192.168.172.4 192.168.172.4:8778 check
#===========================================================

#重启服务查看端口8778和8774是否监听

```

- 编辑配置文件

```bash
#编辑nova.conf
[root@control ~]# vim /etc/nova/nova.conf
2306 use_neutron=true
2465 firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
2629 enabled_apis=osapi_compute,metadata
3021 transport_url=rabbit://openstack:centos@192.168.172.5 #rabbit服务器IP，不支持代理
3085 auth_strategy=keystone
3379 [api_database]
3380 connection = mysql+pymysql://nova:centos@192.168.172.248/nova_api
4396 [database]
4397 connection = mysql+pymysql://nova:centos@192.168.172.248/nova
4954 [glance]
4955 api_servers=http://192.168.172.248:9292
5596 [keystone_authtoken]
5597 auth_uri = http://192.168.172.248:5000
5598 auth_url = http://192.168.172.248:35357
5599 memcached_servers = 192.168.172.248:11211
5600 auth_type = password
5601 project_domain_name = default
5602 user_domain_name = default
5603 project_name = service
5604 username = nova
5605 password = nova #nova用户的密码，不是数据库的密码
7311 lock_path=/var/lib/nova/tmp
8142 [placement]
8143 os_region_name = RegionOne
8144 project_domain_name = Default
8145 project_name = service
8146 auth_type = password
8147 user_domain_name = Default
8148 auth_url = http://192.168.172.248:35357/v3
8149 username = placement
8150 password = placement
9729 [vnc]
9730 enabled=true
9761 vncserver_listen=192.168.172.4
9773 vncserver_proxyclient_address=192.168.172.4

```

- 配置apache允许访问placement API

```bash
[root@control ~]# vim /etc/httpd/conf.d/00-nova-placement-api.conf
#最下方添加以下配置
<Directory /usr/bin>
  <IfVersion >= 2.4>
    Require all granted
  </IfVersion>
  <IfVersion < 2.4>
    Order allow,deny
    Allow from all
  </IfVersion>
</Directory>

#配置完成后重启httpd
[root@control ~]# systemctl restart httpd

```

- 初始化数据库

```bash
#nova_api数据库
[root@control ~]# su -s /bin/sh -c "nova-manage api_db sync" nova

#nova数据库
[root@control ~]# su -s /bin/sh -c "nova-manage db sync" nova
WARNING: cell0 mapping not found - not syncing cell0. 

#nova cell0数据库
[root@control ~]# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

#nova cell1数据库
[root@control ~]# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
45e0a5f7-e655-4b92-a2a3-afb864b47810

```

- 验证nova cell0和nova cell1是否正常注册

```bash
[root@control ~]# nova-manage cell_v2 list_cells
+-------+--------------------------------------+
|  Name |                 UUID                 |
+-------+--------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |
| cell1 | 45e0a5f7-e655-4b92-a2a3-afb864b47810 |
+-------+--------------------------------------+

```

- 启动并将nova服务设置为开机启动

```bash
[root@control ~]# systemctl start openstack-nova-api.service  openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
[root@control ~]# systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service

```

- 重启nova控制脚本

```bash
[root@control ~]# vim nova-restat.sh
#！/bin/bash
systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service

[root@control ~]# chmod +x nova-restat.sh

```

- 查看nova日志

```bash
[root@control ~]# tail /var/log/nova/nova-novncproxy.log 
2019-06-22 04:22:09.984 6427 WARNING oslo_reports.guru_meditation_report [-] Guru meditation now registers SIGUSR1 and SIGUSR2 by default for backward compatibility. SIGUSR1 will no longer be registered in a future release, so please use SIGUSR2 to generate reports.
2019-06-22 04:22:09.985 6427 INFO nova.console.websocketproxy [-] WebSocket server settings:
2019-06-22 04:22:09.985 6427 INFO nova.console.websocketproxy [-]   - Listen on 0.0.0.0:6080
2019-06-22 04:22:09.986 6427 INFO nova.console.websocketproxy [-]   - Flash security policy server
2019-06-22 04:22:09.986 6427 INFO nova.console.websocketproxy [-]   - Web server (no directory listings). Web root: /usr/share/novnc
2019-06-22 04:22:09.986 6427 INFO nova.console.websocketproxy [-]   - No SSL/TLS support (no cert file)
2019-06-22 04:22:09.987 6427 INFO nova.console.websocketproxy [-]   - proxying from 0.0.0.0:6080 to None:None

```

- 查看rabbitMQ连接

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561120540809.png)

- 验证nova控制端

```bash
[root@control ~]# nova service-list
...

```

### - 5.3：部署nova计算节点

在nova计算节点部署服务

- 安装nova计算服务

```bash
[root@compute ~]# yum install openstack-nova-compute -y

```

- 配置nova.conf

```bash
[root@compute ~]# vim /etc/nova/nova.conf
1 [DEFAULT]
2306 use_neutron=true
2465 firewall_driver=nova.virt.firewall.NoopFirewallDriver
2629 enabled_apis=osapi_compute,metadata
3021 transport_url=rabbit://openstack:centos@192.168.172.5
3085 auth_strategy=keystone
3367 [api_database]
3368 connection = mysql+pymysql://nova:centos@192.168.172.248/nova_api
4370 [database]
4371 connection = mysql+pymysql://nova:centos@192.168.172.248/nova
4938 [glance]
4939 api_servers=http://192.168.172.248:9292
5598 [keystone_authtoken]
5599 auth_uri = http://192.168.172.248:5000
5600 auth_url = http://192.168.172.248:35357
5601 memcached_servers = 192.168.172.248:11211
5602 auth_type = password
5603 project_domain_name = default
5604 user_domain_name = default
5605 project_name = service
5606 username = nova
5607 password = nova
7298 [oslo_concurrency]
7299 lock_path=/var/lib/nova/tmp
8144 [placement]
8145 os_region_name = RegionOne
8146 project_domain_name = Default
8147 project_name = service
8148 auth_type = password
8149 user_domain_name = Default
8150 auth_url = http://192.168.172.248:35357/v3
8151 username = placement
8152 password = placement
9731 [vnc]
9732 enabled=true
9771 vncserver_listen=0.0.0.0
9781 vncserver_proxyclient_address=192.168.172.6 #自身IP
9800 novncproxy_base_url=http://192.168.172.4:6080/vnc_auto.html #此处为控制端IP

```

- 确认计算节点是否开启硬件加速

```bash
[root@compute nova]# egrep -c '(vmx|svm)' /proc/cpuinfo
2

```

- 启动nova计算服务并设置为开机启动

```bash
[root@compute ~]# systemctl enable libvirtd.service openstack-nova-compute.service
[root@compute ~]# systemctl start libvirtd.service openstack-nova-compute.service

```

- 将/etc/nova下文件打包发送至计算节点compute2

```bash
#在compute2安装nova服务
yum install openstack-nova-compute -y
#修改/etc/nova/nova.conf文件
vncserver_proxyclient_address=192.168.172.66
#启动服务并设置为开机启动

```



- 添加计算节点到cell数据库

```bash
[root@compute ~]# source /admin-ocata.sh #和控制节点一样的声明变量脚本
[root@compute ~]# openstack hypervisor list
+----+---------------------+-----------------+----------------+-------+
| ID | Hypervisor Hostname | Hypervisor Type | Host IP        | State |
+----+---------------------+-----------------+----------------+-------+
|  1 | compute             | QEMU            | 192.168.172.6  | up    |
|  2 | compute2            | QEMU            | 192.168.172.66 | up    |
+----+---------------------+-----------------+----------------+-------+

```

### - 5.4：主动发现计算节点（在控制节点上执行）

```bash
[root@compute ~]# source /admin-ocata.sh 
[root@compute ~]# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting compute nodes from cell 'cell1': 45e0a5f7-e655-4b92-a2a3-afb864b47810
Found 1 computes in cell: 45e0a5f7-e655-4b92-a2a3-afb864b47810
Checking host mapping for compute host 'compute': f49be8da-1d8b-4713-98f4-596118d48c0f
Creating host mapping for compute host 'compute': f49be8da-1d8b-4713-98f4-596118d48c0f

#若要定期主动发现，则修改配置文件如下
[root@compute ~]# vim /etc/nova/nova.conf
8638 [scheduler]
8745 discover_hosts_in_cells_interval=300 #间隔300s检查一次

```

- 验证计算节点

```bash
[root@control ~]# nova host-list
+-----------+-------------+----------+
| host_name | service     | zone     |
+-----------+-------------+----------+
| control   | scheduler   | internal |
| control   | conductor   | internal |
| control   | consoleauth | internal |
| compute   | compute     | nova     |
| compute2  | compute     | nova     |
+-----------+-------------+----------+

[root@control ~]# nova service-list
+----+------------------+----------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host     | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+----------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-scheduler   | control  | internal | enabled | up    | 2019-06-22T08:25:55.000000 | -               |
| 2  | nova-conductor   | control  | internal | enabled | up    | 2019-06-22T08:25:55.000000 | -               |
| 3  | nova-consoleauth | control  | internal | enabled | up    | 2019-06-22T08:25:54.000000 | -               |
| 9  | nova-compute     | compute  | nova     | enabled | up    | 2019-06-22T08:26:00.000000 | -               |
| 10 | nova-compute     | compute2 | nova     | enabled | up    | 2019-06-22T08:25:54.000000 | -               |
+----+------------------+----------+----------+---------+-------+----------------------------+-----------------+

[root@control ~]# nova image-list
WARNING: Command image-list is deprecated and will be removed after Nova 15.0.0 is released. Use python-glanceclient or openstackclient instead
+--------------------------------------+--------+--------+--------+
| ID                                   | Name   | Status | Server |
+--------------------------------------+--------+--------+--------+
| 1fc48636-8605-44b9-870d-9b761a1a7b53 | cirros | ACTIVE |        |
+--------------------------------------+--------+--------+--------+

[root@control ~]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 1fc48636-8605-44b9-870d-9b761a1a7b53 | cirros | active |
+--------------------------------------+--------+--------+

[root@control ~]# openstack compute service list #列出服务组件是否注册成功
+----+------------------+----------+----------+---------+-------+----------------------------+
| ID | Binary           | Host     | Zone     | Status  | State | Updated At                 |
+----+------------------+----------+----------+---------+-------+----------------------------+
|  1 | nova-scheduler   | control  | internal | enabled | up    | 2019-06-22T08:26:55.000000 |
|  2 | nova-conductor   | control  | internal | enabled | up    | 2019-06-22T08:27:05.000000 |
|  3 | nova-consoleauth | control  | internal | enabled | up    | 2019-06-22T08:27:04.000000 |
|  9 | nova-compute     | compute  | nova     | enabled | up    | 2019-06-22T08:27:00.000000 |
| 10 | nova-compute     | compute2 | nova     | enabled | up    | 2019-06-22T08:27:04.000000 |
+----+------------------+----------+----------+---------+-------+----------------------------+

[root@control ~]# nova-status upgrade check #检查cells和placement API是否正常
+---------------------------------------------------------------------+
| Upgrade Check Results                                               |
+---------------------------------------------------------------------+
| Check: Cells v2                                                     |
| Result: 成功                                                        |
| Details: None                                                       |
+---------------------------------------------------------------------+
| Check: Placement API                                                |
| Result: 成功                                                        |
| Details: None                                                       |
+---------------------------------------------------------------------+
| Check: Resource Providers                                           |
| Result: Warning            |
| Details: There are 1 compute resource providers and 2 compute nodes |
|   in the deployment. Ideally the number of compute resource         |
|   providers should equal the number of enabled compute nodes        |
|   otherwise the cloud may be underutilized. See                     |
|   http://docs.openstack.org/developer/nova/placement.html           |
|   for more details.                                                 |
+---------------------------------------------------------------------+
#上面的警告绝对不能忽略，不然后面虚拟机创建不成功，
#具体解决方法：
1.neutron agent-list查看是否有xx，有则删除重新执行命令检查
2.在计算节点的hosts文件中写上自己的IP和hostname，在控制端hosts文件中也写上计算节点的IP和hostname，确保重启libvirtd.service openstack-nova-compute.service服务是日志中没有错误和警告
#正确示范
[root@control ~]# nova-status upgrade check
+---------------------------+
| Upgrade Check Results     |
+---------------------------+
| Check: Cells v2           |
| Result: 成功              |
| Details: None             |
+---------------------------+
| Check: Placement API      |
| Result: 成功              |
| Details: None             |
+---------------------------+
| Check: Resource Providers |
| Result: 成功              |
| Details: None             |
+---------------------------+

[root@control ~]# openstack catalog list #列出keystone服务中的端点，以验证keystone的连通性
+-----------+-----------+----------------------------------------------+
| Name      | Type      | Endpoints                                    |
+-----------+-----------+----------------------------------------------+
| nova      | compute   | RegionOne                                    |
|           |           |   internal: http://192.168.172.248:8774/v2.1 |
|           |           | RegionOne                                    |
|           |           |   public: http://192.168.172.248:8774/v2.1   |
|           |           | RegionOne                                    |
|           |           |   admin: http://192.168.172.248:8774/v2.1    |
|           |           |                                              |
| glance    | image     | RegionOne                                    |
|           |           |   public: http://192.168.172.248:9292        |
|           |           | RegionOne                                    |
|           |           |   internal: http://192.168.172.248:9292      |
|           |           | RegionOne                                    |
|           |           |   admin: http://192.168.172.248:9292         |
|           |           |                                              |
| keystone  | identity  | RegionOne                                    |
|           |           |   internal: http://192.168.172.248:5000/v3   |
|           |           | RegionOne                                    |
|           |           |   public: http://192.168.172.248:5000/v3     |
|           |           | RegionOne                                    |
|           |           |   admin: http://192.168.172.248:5000/v3      |
|           |           |                                              |
| placement | placement | RegionOne                                    |
|           |           |   public: http://192.168.172.248:8778        |
|           |           | RegionOne                                    |
|           |           |   internal: http://192.168.172.248:8778      |
|           |           | RegionOne                                    |
|           |           |   admin: http://192.168.172.248:8778         |
|           |           |                                              |
+-----------+-----------+----------------------------------------------+

```

## 六、部署网络服务neutron

neutron 是 openstack 的网络组件，是 OpenStack 的网络服务

- 网络：

```bash
在显示的网络环境中我们使用交换机将多个计算机连接起来从而形成了网络，而在neutron 的环境里，网络的功能也是将多个不同的云主机连接起来

```

- 子网：

```bash
是现实的网络环境下可以将一个网络划分成多个逻辑上的子网络，从而实现网络隔离，在 neutron 里面子网也是属于网络

```

- 端口：

```bash
计算机连接交换机通过网线连，而网线插在交换机的不同端口，在 neutron 里面端口属于子网，即每个云主机的子网都会对应到一个端口

```

- 路由器：

```bash
用于连接不同的网络或者子网

```

- 网络类型：

```bash
提供者网络：虚拟机桥接到物理机，并且虚拟机必须和物理机在同一个网络范围内
自服务网络：可以自己创建网络，最终会通过虚拟路由器连接外网

```

### - 6.1：数据库准备

- MySQL服务器创建数据库并授权

```bash
[root@localhost ~]# mysql -uroot -pcentos
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 37
Server version: 5.5.60-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database neutron;
Query OK, 1 row affected (0.01 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'centos';
Query OK, 0 rows affected (0.00 sec)


```

- 控制端连接测试

```bash
[root@control ~]# mysql -uneutron -pcentos -h192.168.172.248
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 51
Server version: 5.5.60-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| neutron            |
+--------------------+
2 rows in set (0.00 sec)

```

### - 6.2：创建neutron用户并设置密码为neutron

```bash
[root@control1 ~]# openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | ff3fa2a78082485ba524f9273fb6f106 |
| enabled             | True                             |
| id                  | d50ab8ade825447d85d664ce7422ca02 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

- 给neutron用户授权admin

```bash
[root@control1 ~]# openstack role add --project service --user neutron admin

```

### - 6.3：创建neutron服务并注册

- 创建neutron服务：

```bash
[root@control ~]# openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | dec6392cb6bf447e97a34ea53e1b0660 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

```

- 注册endpoint端点：

```bash
#注册公共endpoint
[root@control ~]# openstack endpoint create --region RegionOne network public http://192.168.172.248:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | babc3060a54842798a1d523ee84da704 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | dec6392cb6bf447e97a34ea53e1b0660 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.172.248:9696      |
+--------------+----------------------------------+

#注册私有endpoint
[root@control ~]# openstack endpoint create --region RegionOne network internal http://192.168.172.248:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e799327eca9d4f519887bb8fb2b88203 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | dec6392cb6bf447e97a34ea53e1b0660 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.172.248:9696      |
+--------------+----------------------------------+

#注册管理endpoint
[root@control ~]# openstack endpoint create --region RegionOne network admin http://192.168.172.248:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 465aa3d83402447aa10512ac2fd2de65 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | dec6392cb6bf447e97a34ea53e1b0660 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.172.248:9696      |
+--------------+----------------------------------+

```

- 验证端点是否添加成功

```bash
[root@control ~]# openstack endpoint list
...

```

- 配置haproxy负载转发

```bash
root@ubuntu1804:~# vim /etc/haproxy/haproxy.cfg
listen neutron
bind 192.168.172.248:9696
mode tcp
server 192.168.172.4 192.168.172.4:9696 check

#配置完成后重启服务验证端口
root@ubuntu1804:~# systemctl restart haproxy
root@ubuntu1804:~# ss -nlt
State      Recv-Q      Send-Q               Local Address:Port              Peer Address:Port      
LISTEN     0           128                192.168.172.248:9696                   0.0.0.0:* 
...

```

### - 6.4：部署neutron控制端

- 控制端安装neutron

```bash
[root@control ~]# yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y

```

- 编辑neutron配置文件

```bash
[root@control ~]# vim /etc/neutron/neutron.conf
27 auth_strategy = keystone
30 core_plugin = ml2
33 service_plugins =
99 notify_nova_on_port_status_changes = true
103 notify_nova_on_port_data_changes = true
570 transport_url = rabbit://openstack:centos@192.168.172.5
765 connection = mysql+pymysql://neutron:centos@192.168.172.248/neutron
845 [keystone_authtoken]
846 auth_uri = http://192.168.172.248:5000
847 auth_url = http://192.168.172.248:35357
848 memcached_servers = 192.168.172.248:11211
849 auth_type = password
850 project_domain_name = default
851 user_domain_name = default
852 project_name = service
853 username = neutron
854 password = neutron
1073 [nova]
1074 auth_url = http://192.168.172.248:35357
1075 auth_type = password
1076 project_domain_name = default
1077 user_domain_name = default
1078 region_name = RegionOne
1079 project_name = service
1080 username = nova
1081 password = nova
1195 lock_path = /var/lib/neutron/tmp

```

- Modular Layer 2 

```bash
#ML2插件使用Linuxbridge机制来为实例创建layer-2虚拟网络基础设施
[root@control ~]# vim /etc/neutron/plugins/ml2/ml2_conf.ini
122 type_drivers = flat,vlan
127 tenant_network_types =
131 mechanism_drivers = linuxbridge
136 extension_drivers = port_security
172 flat_networks = internal
249 enable_ipset = true

```

- 配置linuxbridge代理

```bash
[root@control ~]# vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
155 physical_interface_mappings = internal:eth0 #内部网络
168 firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
173 enable_security_group = true #开启安全组功能，需要在web端设置路由规则
188 enable_vxlan = false

```

- 配置DHCP代理

```bash
[root@control ~]# vim /etc/neutron/dhcp_agent.ini
16 interface_driver = linuxbridge
32 dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
33 #/usr/lib/python2.7/site-packages/neutron/agent/linux/iptables_firewall.py
41 enable_isolated_metadata = true

```

- 配置元数据代理

```bash
[root@control ~]# vim /etc/neutron/metadata_agent.ini
22 nova_metadata_ip = 192.168.172.248
34 metadata_proxy_shared_secret = 20190622

```

- 配置nova调用neutron

```bash
[root@control ~]# vim /etc/nova/nova.conf
6992 [neutron]
6993 url = http://192.168.172.248:9696
6994 auth_url = http://192.168.172.248:35357
6995 auth_type = password
6996 project_domain_name = default
6997 user_domain_name = default
6998 region_name = RegionOne
6999 project_name = service
7000 username = neutron
7001 password = neutron
7061 service_metadata_proxy=true
7072 metadata_proxy_shared_secret = 20190622

```

- 创建软链接

```bash
[root@control ~]# ln -sv /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
"/etc/neutron/plugin.ini" -> "/etc/neutron/plugins/ml2/ml2_conf.ini"

```

- 初始化数据库

```bash
[root@control ~]# su -s /bin/sh  -c  "neutron-db-manage  --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
  正在对 neutron 运行 upgrade...
...

```

- 重启nova API服务

```bash
[root@control ~]# systemctl restart openstack-nova-api.service

#重启后验证nova API日志有没有报错
[root@control ~]# tail -n200 /var/log/nova/nova-api.log
...

```

- 配置haproxy代理

```bash
root@ubuntu1804:~# vim /etc/haproxy/haproxy.cfg
listen nova_api
bind 192.168.172.248:8775
mode tcp
server 192.168.172.4 192.168.172.4:8775 check

#重启服务验证端口监听
root@ubuntu1804:~# systemctl restart haproxy
root@ubuntu1804:~# ss -nlt
State      Recv-Q      Send-Q     Local Address:Port      PeerAddress:Port           
LISTEN     0           128        192.168.172.248:8775       0.0.0.0:* 
...

```

- 启动neutron服务并设置为开机启动

```bash
[root@control ~]# systemctl enable neutron-server.service  neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-server.service to /usr/lib/systemd/system/neutron-server.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-dhcp-agent.service to /usr/lib/systemd/system/neutron-dhcp-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-metadata-agent.service to /usr/lib/systemd/system/neutron-metadata-agent.service.

[root@control ~]# systemctl start neutron-server.service  neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service

#验证neutron日志不能有报错，全是INFO则为正确
[root@control ~]# tail -f /var/log/neutron/*.log
...

```

- 验证neutron控制端是否注册成功（此步骤要求所有服务器时间必须一致）

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561210097370.png)

- neutron控制端重启脚本

```bash
[root@control ~]# vim neutron-restart.sh
#!/bin/bash
systemctl restart openstack-nova-api.service neutron-server.service neutron-linuxbridgeagent.service neutron-dhcp-agent.service neutron-metadata-agent.service
[root@control ~]# chmod +x neutron-restart.sh

```

### - 6.5：部署neutron计算节点

```bash
#安装服务
[root@compute ~]# yum install openstack-neutron-linuxbridge ebtables ipset -y

```

- 编辑配置文件

```bash
[root@compute ~]# vim /etc/neutron/neutron.conf
27 auth_strategy = keystone
570 transport_url = rabbit://openstack:centos@192.168.172.5
845 [keystone_authtoken]
846 auth_uri = http://192.168.172.248:5000
847 auth_url = http://192.168.172.248:35357
848 memcached_servers = 192.168.172.248:11211
849 auth_type = password
850 project_domain_name = default
851 user_domain_name = default
852 project_name = service
853 username = neutron
854 password = neutron
1188 lock_path = /var/lib/neutron/tmp

```

- 配置linuxbridge代理

```bash
[root@compute ~]# vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
156 physical_interface_mappings = internal:eth0
169 firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
174 enable_security_group = true
189 enable_vxlan = false
#209 local_ip = 172.20.30.6 #若启动服务报错有关local_ip，则添加修改此项

```

- 配置nova调用使用网络

```bash
[root@compute ~]# vim /etc/nova/nova.conf
6992 [neutron]
6993 url = http://192.168.172.248:9696
6994 auth_url = http://192.168.172.248:35357
6995 auth_type = password
6996 project_domain_name = default
6997 user_domain_name = default
6998 region_name = RegionOne
6999 project_name = service
7000 username = neutron
7001 password = neutron

#配置完成后重启nova服务
[root@compute ~]# systemctl restart openstack-nova-compute.service

```

- 启动neutron并设置为开机启动

```bash
[root@compute ~]# systemctl enable neutron-linuxbridge-agent.service
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.
[root@compute ~]# systemctl start neutron-linuxbridge-agent.service

#验证neutron日志，不报错即为正常
[root@compute ~]# tail -n100 /var/log/neutron/linuxbridge-agent.log

```

- neutron控制端验证计算节点是否注册成功

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561211290906.png)

- 验证neutron server进程是否正常运行

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561211420793.png)

## 七、仪表盘horizon

horizon 是 openstack 的管理其他组件的图形显示和操作界面，通过 API 和其他服务进行通讯，如镜像服务、计算服务和网络服务等结合使用，horizon 基于 python django 开发，通过Apache 的 wsgi 模块进行 web 访问通信，Horizon 只需要更改配置文件连接到 keyston 即可

### - 7.1：控制端安装horizon

```bash
[root@control ~]# yum install openstack-dashboard -y

```

### - 7.2：修改配置文件

```bash
[root@control ~]# vim /etc/openstack-dashboard/local_settings
28 ALLOWED_HOSTS = ['*',]
159 OPENSTACK_HOST = "192.168.172.248"
#配置memcached会话保持
129 SESSION_ENGINE = 'django.contrib.sessions.backends.cache' #新增加
130 CACHES = {  #注释之前的配置，切记
131      'default': {
132         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
133         'LOCATION': '192.168.172.248:11211',
134        },
135 }
#启用第三版API认证
167 OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
#启用对域的支持
66 OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
#配置API版本
54 OPENSTACK_API_VERSIONS = {
55     "data-processing": 1.1,
56     "identity": 3,
57     "image": 2,
58     "volume": 2,
59 #    "compute": 2,
60 }
#设置默认域：
73 OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
#配置web界面创建的用户默认权限：
168 OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
#单一扁平网络模式下，禁用第三层网络
288 OPENSTACK_NEUTRON_NETWORK = {
289     'enable_router': False,
290     'enable_quotas': False,
291     'enable_ipv6': False,
292     'enable_distributed_router': False,
293     'enable_ha_router': False,
294     'enable_lb': False,
295     'enable_firewall': False,
296     'enable_vpn': False,
297     'enable_fip_topology_check': False,
#配置时区：
423 TIME_ZONE = "Asia/Shanghai"

```

- 配置完成后重启httpd服务

```bash
[root@control ~]# systemctl restart httpd

```

### - 7.3：配置haproxy代理

```bash
#配置haproxy代理horizon
root@ubuntu1804:~# vim /etc/haproxy/haproxy.cfg
listen horizon
bind 192.168.172.248:80
mode tcp
server 192.168.172.4 192.168.172.4:80 check

#配置完成后重启服务
root@ubuntu1804:~# systemctl restart haproxy.service
root@ubuntu1804:~# ss -nlt
State      Recv-Q      Send-Q               Local Address:Port              Peer Address:Port      
...       
LISTEN     0           128                192.168.172.248:80                     0.0.0.0:* 

```

### - 7.4：验证访问web界面

- 登录界面

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561213299633.png)

- 登录后界面

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561213495956.png)

## 八、创建虚拟机

启动虚拟机之前需要先做一些前期准备，比如网络和IP地址分配，虚拟机；类型创建等

### - 8.1：网络规划及IP划分

官方文档：<https://docs.openstack.org/ocata/zh_CN/install-guide-rdo/launch-instance.html#id1>

- 桥接网络IP划分，要求虚拟机与物理机必须在同一个相同子网的网络内

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561251723292.png)

- 创建桥接网络

```bash
#语法：
openstack network reate --在项目之间共享 --外部网络 --provider-physical-network  --配置文件名称 --provider-network-type flat --自定义网络名称
#/etc/neutron/plugins/ml2/ml2_conf.ini #控制端自有
#/etc/neutron/plugins/ml2/linuxbridge_agent.ini #控制端和计算节点共有

[root@control ~]# openstack network create --share --external --provider-physical-network internal --provider-network-type flat internal-net
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2019-06-23T02:51:52Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 7a689c6e-54ba-4cdf-a8b6-5a112fb32d1f |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| mtu                       | 1500                                 |
| name                      | internal-net                         |
| port_security_enabled     | True                                 |
| project_id                | e6ac9e68df504ce387db705ca3c45bda     |
| provider:network_type     | flat                                 |
| provider:physical_network | internal                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 4                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| updated_at                | 2019-06-23T02:51:52Z                 |
+---------------------------+--------------------------------------+

```

- 创建子网

```bash
#语法：
openstack subnet create --network 上一步定义的网络名称 --allocation-pool start=开始IP,end=结束IP --dns-nameserver DNS --gateway 网关 --subnet-range IP/掩码 自定义名称

[root@control ~]# openstack subnet create --network internal-net --allocation-pool start=192.168.172.150,end=192.168.172.200  --dns-nameserver  192.168.172.2  --gateway 192.168.172.2 --subnet-range 192.168.172.0/24 internal
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 192.168.172.101-192.168.172.150      |
| cidr              | 192.168.172.0/24                     |
| created_at        | 2019-06-23T02:59:39Z                 |
| description       |                                      |
| dns_nameservers   | 192.168.172.2                        |
| enable_dhcp       | True                                 |
| gateway_ip        | 192.168.172.2                        |
| host_routes       |                                      |
| id                | 4a54fad3-2f80-4742-b672-b8740fce672f |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | internal                             |
| network_id        | 7a689c6e-54ba-4cdf-a8b6-5a112fb32d1f |
| project_id        | e6ac9e68df504ce387db705ca3c45bda     |
| revision_number   | 2                                    |
| segment_id        | None                                 |
| service_types     |                                      |
| subnetpool_id     | None                                 |
| updated_at        | 2019-06-23T02:59:39Z                 |
+-------------------+--------------------------------------+

```

- 验证网络

```bash
[root@control ~]# openstack network list
+--------------------------------------+--------------+--------------------------------------+
| ID                                   | Name         | Subnets                              |
+--------------------------------------+--------------+--------------------------------------+
| 7a689c6e-54ba-4cdf-a8b6-5a112fb32d1f | internal-net | 4a54fad3-2f80-4742-b672-b8740fce672f |
+--------------------------------------+--------------+--------------------------------------+

#查看子网
[root@control ~]# openstack subnet list
+---------------------------------+----------+---------------------------------+------------------+
| ID                              | Name     | Network                         | Subnet           |
+---------------------------------+----------+---------------------------------+------------------+
| 4a54fad3-2f80-4742-b672-b8740fc | internal | 7a689c6e-54ba-4cdf-             | 192.168.172.0/24 |
| e672f                           |          | a8b6-5a112fb32d1f               |                  |
+---------------------------------+----------+---------------------------------+------------------

[root@control ~]# neutron net-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------+--------------+--------------------------+----------------------------+
| id                       | name         | tenant_id                | subnets                    |
+--------------------------+--------------+--------------------------+----------------------------+
| 7a689c6e-54ba-4cdf-      | internal-net | e6ac9e68df504ce387db705c | 4a54fad3-2f80-4742-b672-b8 |
| a8b6-5a112fb32d1f        |              | a3c45bda                 | 740fce672f                 |
|                          |              |                          | 192.168.172.0/24           |
+--------------------------+--------------+--------------------------+----------------------------

[root@control ~]# neutron subnet-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+---------------------+----------+---------------------+------------------+-----------------------+
| id                  | name     | tenant_id           | cidr             | allocation_pools      |
+---------------------+----------+---------------------+------------------+-----------------------+
| 4a54fad3-2f80-4742- | internal | e6ac9e68df504ce387d | 192.168.172.0/24 | {"start":             |
| b672-b8740fce672f   |          | b705ca3c45bda       |                  | "192.168.172.101",    |
|                     |          |                     |                  | "end":                |
|                     |          |                     |                  | "192.168.172.150"}    |
+---------------------+----------+---------------------+------------------+-----------------------

```

- web端验证网络

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561259229972.png)

### - 8.2：创建虚拟机类型

```bash
#测试cirros镜像
[root@control ~]# openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
+----------------------------+---------+
| Field                      | Value   |
+----------------------------+---------+
| OS-FLV-DISABLED:disabled   | False   |
| OS-FLV-EXT-DATA:ephemeral  | 0       |
| disk                       | 1       |
| id                         | 0       |
| name                       | m1.nano |
| os-flavor-access:is_public | True    |
| properties                 |         |
| ram                        | 64      |
| rxtx_factor                | 1.0     |
| swap                       |         |
| vcpus                      | 1       |
+----------------------------+---------+

```

- web端验证虚拟机类型

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561259436836.png)

### - 8.3：实现免密码登录

- 生成key

```bash
[root@control ~]# ssh-keygen -q -N ""
[root@control ~]# ll /root/.ssh/
总用量 12
-rw------- 1 root root 1679 6月  23 11:13 id_rsa
-rw-r--r-- 1 root root  394 6月  23 11:13 id_rsa.pub
-rw-r--r-- 1 root root  175 6月  21 21:49 known_hosts

```

- 添加公钥

```bash
[root@control ~]# openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | 79:10:e8:40:af:88:45:27:9b:0a:ec:00:89:58:37:92 |
| name        | mykey                                           |
| user_id     | 30fa771d71fa47d9a908f548e6e0cd77                |
+-------------+-------------------------------------------------+

#验证key
[root@control ~]# openstack keypair list
+-------+-------------------------------------------------+
| Name  | Fingerprint                                     |
+-------+-------------------------------------------------+
| mykey | 79:10:e8:40:af:88:45:27:9b:0a:ec:00:89:58:37:92 |
+-------+-------------------------------------------------+

```

- web端验证公钥

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561259803461.png)

### - 8.4：安全组

- 创建安全组

```bash
[root@control ~]# openstack security group rule create --proto icmp default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2019-06-23T03:21:28Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | cf345da0-f501-4347-8c5c-0b1af4b3d86d |
| name              | None                                 |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | e6ac9e68df504ce387db705ca3c45bda     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | 0bd1b9fd-5e5a-46f4-b4ae-33db2e43efbb |
| updated_at        | 2019-06-23T03:21:28Z                 |
+-------------------+--------------------------------------+

#若是提示有多个安全组存在而导致不能创建安全组，则删除一个即可
[root@control ~]# openstack security group list
+--------------------------------------+---------+-------------+----------------------------------+
| ID                                   | Name    | Description | Project                          |
+--------------------------------------+---------+-------------+----------------------------------+
| 0bd1b9fd-5e5a-46f4-b4ae-33db2e43efbb | default | 缺省安全组  | e6ac9e68df504ce387db705ca3c45bda |
| 1022171d-8261-4871-9f0f-948f808a8625 | default | 缺省安全组  | cba6c8e57c0f4789a6a7764b69486f78 |
+--------------------------------------+---------+-------------+----------------------------------+
[root@control ~]# openstack security group delete 1022171d-8261-4871-9f0f-948f808a8625

```

- 添加规则

```bash
[root@control ~]# openstack security group rule create --proto tcp --dst-port 22 default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2019-06-23T03:22:39Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | 410d5305-4782-41fa-9201-48576885e690 |
| name              | None                                 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | e6ac9e68df504ce387db705ca3c45bda     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | 0bd1b9fd-5e5a-46f4-b4ae-33db2e43efbb |
| updated_at        | 2019-06-23T03:22:39Z                 |
+-------------------+--------------------------------------+

```

- web端验证

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561260220225.png)

- 最终验证

```bash
#列出虚拟机类型
[root@control ~]# openstack flavor list
+----+---------+-----+------+-----------+-------+-----------+
| ID | Name    | RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+---------+-----+------+-----------+-------+-----------+
| 0  | m1.nano |  64 |    1 |         0 |     1 | True      |
+----+---------+-----+------+-----------+-------+-----------+

#列出可用镜像
[root@control ~]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 1fc48636-8605-44b9-870d-9b761a1a7b53 | cirros | active |
+--------------------------------------+--------+--------+

#列出可用网络
[root@control ~]# openstack network list
+--------------------------------------+--------------+--------------------------------------+
| ID                                   | Name         | Subnets                              |
+--------------------------------------+--------------+--------------------------------------+
| 7a689c6e-54ba-4cdf-a8b6-5a112fb32d1f | internal-net | 4a54fad3-2f80-4742-b672-b8740fce672f |
+--------------------------------------+--------------+--------------------------------------+

#列出安全组
[root@control ~]# openstack security group list
+--------------------------------------+---------+-------------+----------------------------------+
| ID                                   | Name    | Description | Project                          |
+--------------------------------------+---------+-------------+----------------------------------+
| 0bd1b9fd-5e5a-46f4-b4ae-33db2e43efbb | default | 缺省安全组  | e6ac9e68df504ce387db705ca3c45bda |
+--------------------------------------+---------+-------------+----------------------------------

#当所有验证无误方可执行以下步骤

```

### - 8.5:：命令行启动虚拟机

- 创建虚拟机

```bash
#语法
openstack server create --flavor 虚拟机类型 --image 镜像名称 --nic net-id=network-ID --security-group 安全组名 --key-name key 名称 虚拟机名

[root@control ~]# openstack server create --flavor m1.nano --image cirros --nic net-id=7a689c6e-54ba-4cdf-a8b6-5a112fb32d1f --security-group default --key-name mykey javis
+-------------------------------------+-----------------------------------------------+
| Field                               | Value                                         |
+-------------------------------------+-----------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                        |
| OS-EXT-AZ:availability_zone         |                                               |
| OS-EXT-SRV-ATTR:host                | None                                          |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                          |
| OS-EXT-SRV-ATTR:instance_name       |                                               |
| OS-EXT-STS:power_state              | NOSTATE                                       |
| OS-EXT-STS:task_state               | scheduling                                    |
| OS-EXT-STS:vm_state                 | building                                      |
| OS-SRV-USG:launched_at              | None                                          |
| OS-SRV-USG:terminated_at            | None                                          |
| accessIPv4                          |                                               |
...

```

- 查看虚拟机

```bash
[root@control ~]# openstack server list
+--------------------------------------+-------+--------+------------------------------+------------+
| ID                                   | Name  | Status | Networks                     | Image Name |
+--------------------------------------+-------+--------+------------------------------+------------+
| ce6a4114-8dda-4249-ab7a-75bcc3772a19 | javis | ACTIVE | internal-net=192.168.172.159 | cirros     |
+--------------------------------------+-------+--------+------------------------------+------------+

```

- 查看虚拟机访问地址

```bash
[root@control ~]# openstack console url show  javis
+-------+------------------------------------------------------------------------------------+
| Field | Value                                                                              |
+-------+------------------------------------------------------------------------------------+
| type  | novnc                                                                              |
| url   | http://192.168.172.4:6080/vnc_auto.html?token=9646e1a0-e1d2-4b9f-a8ed-120d3ccf0bb8 |
+-------+------------------------------------------------------------------------------------+

```

- 使用浏览器访问虚拟机URL(ocata需要用centos7.2-1511操作，否则会产生版本冲突)

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561365231593.png)

- 图进行界面创建虚拟机（此步骤不再赘述）
- 验证虚拟机运行正常

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561379593162.png)

## 九、快速添加新计算节点

当后期添加新物理服务器作为计算节点，如果按照上面的过程安装配置会特别慢，但可以通过复制配置文件的方式快速添加

### - 9.1：计算节点安装服务

```bash
#提前将yum仓库、防火墙、selinux、主机名、时间同步等配置完毕
[root@compute ~]# yum install -y net-tools vim lrzsz tree screen lsof tcpdump
[root@compute ~]# yum install centos-release-openstack-ocata.noarch -y
[root@compute ~]# yum install -y python-openstackclient openstack-selinux openstack-neutron-linuxbridge ebtables ipset #安装全部相关rpm包
[root@compute ~]# 

```

## 十、实现内外网结构

类似于阿里云ECS主机的内外网（双网卡不同网段）的结构，最终实现内外网区分隔离

### - 10.1：各虚拟机添加网卡并配置IP

- 添加仅主机模式网卡
- 各虚拟机配置IP

```bash
#控制端：10.20.30.4
#haproxy：10.20.30.3
#mysql：10.20.30.5
#计算节点：10.20.30.6

#配置完成后验证可以互相ping通

```

### - 10.2：控制节点配置

- 编辑配置文件linuxbridge_agent.ini

```bash
[root@control ~]# vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
155 physical_interface_mappings = internal:eth0,external:eth1

#全部配置：
[root@control ~]# vim /etc/neutron/plugins/ml2/ml2_conf.ini
[root@control ~]# grep  "^[a-Z]" /etc/neutron/plugins/ml2/linuxbridge_agent.ini
physical_interface_mappings = internal:eth0,external:eth1
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
enable_security_group = true
enable_vxlan = false

```

- 编辑配置文件ml2_conf.ini

```bash
[root@control ~]# vim /etc/neutron/plugins/ml2/ml2_conf.ini
172 flat_networks = internal,external

#全部配置：
[root@control ~]# grep  "^[a-Z]" /etc/neutron/plugins/ml2/ml2_conf.ini
type_drivers = flat,vlan
tenant_network_types = 
mechanism_drivers = linuxbridge
extension_drivers = port_security
flat_networks = internal,external
enable_ipset = true

```

- 重启neutron服务

```bash
[root@control ~]# systemctl  restart neutron-linuxbridge-agent
[root@control ~]# systemctl  restart neutron-server

```

### - 10.3：计算节点配置

- 编辑配置文件

```bash
[root@compute ~]# vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
155 physical_interface_mappings = internal:eth0,external:eth1

#全部配置：
[root@compute ~]# grep "^[a-Z]" /etc/neutron/plugins/ml2/linuxbridge_agent.ini
physical_interface_mappings = internal:eth0,external:eth1
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
enable_security_group = true
enable_vxlan = false

```

- 重启neutron服务

```bash
[root@compute ~]# systemctl  restart neutron-linuxbridge-agent

```

### - 10.4：创建网络并认证

```bash
[root@control ~]# neutron net-create --shared --provider:physical_network external  --provider:network_type flat external-net
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2019-06-24T19:56:19Z                 |
| description               |                                      |
| id                        | bd66d25d-1cae-4fc2-a805-40b4eac605c9 |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| mtu                       | 1500                                 |
| name                      | external-net                         |
| port_security_enabled     | True                                 |
| project_id                | e6ac9e68df504ce387db705ca3c45bda     |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| revision_number           | 3                                    |
| router:external           | False                                |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | e6ac9e68df504ce387db705ca3c45bda     |
| updated_at                | 2019-06-24T19:56:19Z                 |
+---------------------------+--------------------------------------+

```

- 创建子网

```bash
[root@control ~]# neutron subnet-create --name external-subnet   --allocation-pool start=10.20.30.100,end=10.20.30.150 external-net 10.20.30.0/24
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "10.20.30.100", "end": "10.20.30.150"} |
| cidr              | 10.20.30.0/24                                    |
| created_at        | 2019-06-24T20:03:08Z                             |
| description       |                                                  |
| dns_nameservers   |                                                  |
| enable_dhcp       | True                                             |
| gateway_ip        | 10.20.30.1                                       |
| host_routes       |                                                  |
| id                | 3e1c8896-8729-42ab-b23c-1cb43f1c0ea1             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | external-subnet                                  |
| network_id        | bd66d25d-1cae-4fc2-a805-40b4eac605c9             |
| project_id        | e6ac9e68df504ce387db705ca3c45bda                 |
| revision_number   | 2                                                |
| service_types     |                                                  |
| subnetpool_id     |                                                  |
| tags              |                                                  |
| tenant_id         | e6ac9e68df504ce387db705ca3c45bda                 |
| updated_at        | 2019-06-24T20:03:08Z                             |
+-------------------+--------------------------------------------------+

```

- 验证子网

```bash
[root@control ~]# neutron net-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+---------------------------------+--------------+---------------------------------+---------------------------------+
| id                              | name         | tenant_id                       | subnets                         |
+---------------------------------+--------------+---------------------------------+---------------------------------+
| 31088730-af0d-4342-a45c-        | internal-net | e6ac9e68df504ce387db705ca3c45bd | e66bd43c-c424-4f8b-             |
| 00671b7934d4                    |              | a                               | 87d2-7bac33c2ccb6               |
|                                 |              |                                 | 192.168.172.0/24                |
| bd66d25d-1cae-                  | external-net | e6ac9e68df504ce387db705ca3c45bd | 3e1c8896-8729-42ab-b23c-        |
| 4fc2-a805-40b4eac605c9          |              | a                               | 1cb43f1c0ea1 10.20.30.0/24      |
+---------------------------------+--------------+---------------------------------+---------------------------------+

```

### - 10.5：创建虚拟机

- 在网卡界面添加两个网卡

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561379348644.png)

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Openstack/1561379451030.png)
