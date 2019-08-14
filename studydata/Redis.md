## 一、缓存的概念

- 加速CPU访问硬盘数据，提高CPU的工作效率

#### ![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560253446775.png)

### - 1.1：系统缓存

#### - 1.1.1：buffer与cache

- buffer：写缓冲，一般用于写操作，将数据先写入内存再写入磁盘，服务器突然断电会丢失内存中的部分数据
- cache：读缓存，一般用于读操作，CPU将需要频繁读取的数据放在距离自己最近的缓存区，加快读取速度

#### - 1.1.2：cache的特性

- 自动过期：给缓存的数据加上有效时间，超出时间后自动过期删除
- 过期时间：强制过期，原网站更新图片后CDN是不会更新的，需要强制将缓存的图片过期
- 命中率：缓存读取命中率

### - 1.2：用户层缓存

#### - 1.2.1：DNS缓存

- 默认为60秒，即60秒之内再访问同一个域名就不再进行DNS解析

#### - 1.2.2：浏览器缓存

#### - 1.2.3：浏览器缓存过期机制

- 最后修改时间：系统调用会获取稳健的最后修改时间，如果没有发生变化就返回给浏览器304的状态码，表明没有发生变化，然后浏览器就使用本地的缓存展示资源
- Etag标记：基于Etag标记是否一致判断页面是否发生变化
- 过期时间：第一次请求资源额时候带一个资源的过期时间，默认为30天

### - 1.3：CDN缓存

#### - 1.3.1：CDN

- 内容分发网络（Content Delivery Network），通过将服务内容分发至全网加速节点，利用全球调度系统使用户能够就近获取，有效降低访问延迟，提升服务可用性
  - 降低机房的使用带宽，很多资源通过CDN就直接返回用户了
  - 解决不同运营商之间的关联，各自的网络访问各自的网络，加速用户访问
  - 解决用户访问的地域问题，就近返回用户资源

#### - 1.3.2：用户请求CDN流程

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560255690304.png)

#### - 1.3.3：CDN优势

- 提前对静态内容进行预缓存，避免大量的请求回源，导致主站网络带宽被打满而导致数据无法更新，另外 CDN 可以将数据根据访问的热度不同而进行不同级别的缓存，例如访问量最高的资源访问 CDN边缘节点的内存，其次的放在 SSD 或者 SATA，再其次的放在云存储，这样兼顾了速度与成本
- 调度准确：将用户调度到最近的边缘节点
- 性能优化：专门用于缓存，响应速度快
- 安全相关：可以抵御攻击
- 节省带宽：用户请求由边缘节点响应，大幅降低了到源站的带宽

### - 1.4：应用层缓存

- Nginx、PHP 等 web 服务可以设置应用缓存以加速响应用户请求

### - 1.5：其他层面缓存

- CPU缓存：L1的数据缓存和指令缓存、二级缓存、三级缓存

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560256019917.png)

## 二、redis部署与使用

### - 2.1：redis基础

#### - 2.1.1：redis对比memcached

- 支持数据的持久化：可以将内存中的数据保存在磁盘中，重启redis服务后可以从备份文件中恢复数据到内存中继续使用
- 支持更多的数据类型：支持string、hash、list、set（集合）、zet（有序集合）
- 支持数据备份：支持主从同步、快照+AOF
- 支持更大的value数据：memcached单个key value最大支持1MB，redis最大支持512MB
- 线程与并发：redis是单线程，memcached是多线程，单机情况下没有memcached并发高，但redis支持分布式集群以实现更高的并发，单redis也可以实现数万并发
- 支持集群横向扩展：基于redis cluster的横向扩展，可以实现分布式集群，大幅提升性能和数据安全性
- 都是基于c语言开发

#### - 2.1.2：应用场景

- session共享：常见于 web 集群中的 Tomcat 或者 PHP 中多 web 服务器 session 共享
- 消息队列：ELK的日志缓存、部分业务的订阅发布系统
- 计数器：访问排行榜、商品浏览数等和次数相关的数值统计场景
- 缓存：数据查询、电商网站商品信息、新闻内容
- 微博/微信社交场合：共同好友、点赞评论

### - 2.2：Redis安装与使用

#### - 2.2.1：yum安装

```bash
#需要epel源
[root@centos ~]$yum list redis
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
Installed Packages
redis.x86_64                                                           3.2.12-2.el7     
[root@centos ~]$yum install redis -y
```

#### - 2.2.2：编译安装

```bash
[root@centos ~]$cd /usr/local/src/
[root@centos ~]$wget http://download.redis.io/releases/redis-4.0.14.tar.gz
[root@centos src]$tar xvf redis-4.0.14.tar.gz 
[root@centos src]$cd redis-4.0.14/deps/
[root@centos deps]$yum install gcc jemalloc-devel -y
[root@centos deps]$make hiredis jemalloc linenoise lua
drwxr-xr-x. 2 root root 134 Jun 11 21:53 bin
[root@centos deps]$cd ..
[root@centos redis-4.0.14]$make PREFIX=/apps/redis install
[root@centos redis-4.0.14]$ll /apps/redis/
total 0
drwxr-xr-x. 2 root root 134 Jun 11 21:53 bin
[root@centos ~]$mkdir /apps/redis/{etc,run,data,logs}
[root@centos ~]$cp /usr/local/src/redis-4.0.14/redis.conf /apps/redis/etc/ #复制配置文件
[root@centos redis]$ln -sv bin/redis-* /usr/sbin/ #创建命令软链接
‘/usr/sbin/redis-benchmark’ -> ‘bin/redis-benchmark’ #性能测试工具
‘/usr/sbin/redis-check-aof’ -> ‘bin/redis-check-aof’ #AOF文件检查功能
‘/usr/sbin/redis-check-rdb’ -> ‘bin/redis-check-rdb’ #RDB文件检查工具
‘/usr/sbin/redis-cli’ -> ‘bin/redis-cli’ #redis客户端工具
‘/usr/sbin/redis-sentinel’ -> ‘bin/redis-sentinel’ #哨兵
‘/usr/sbin/redis-server’ -> ‘bin/redis-server’ #redis服务端
#查看帮助和启动测试
[root@centos redis]$/apps/redis/bin/redis-server -v #查看版本信息
Redis server v=4.0.14 sha=00000000:0 malloc=jemalloc-4.0.3 bits=64 build=aeae6c8087d21a8e
[root@centos redis]$/apps/redis/bin/redis-server -h #查看帮助
Usage: ./redis-server [/path/to/redis.conf] [options]
       ./redis-server - (read config from stdin)
       ./redis-server -v or --version
       ./redis-server -h or --help
       ./redis-server --test-memory <megabytes>

Examples:
       ./redis-server (run the server with default conf)
       ./redis-server /etc/redis/6379.conf
       ./redis-server --port 7777
       ./redis-server --port 7777 --slaveof 127.0.0.1 8888
       ./redis-server /etc/myredis.conf --loglevel verbose

Sentinel mode:
       ./redis-server /etc/sentinel.conf --sentinel
```

- 服务启动及警告解决

```bash
[root@centos redis]$/apps/redis/bin/redis-server /apps/redis/etc/redis.conf #启动后默认前端执行
...#启动后出现警告
11362:M 12 Jun 09:12:18.976 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11362:M 12 Jun 09:12:18.976 # Server initialized
11362:M 12 Jun 09:12:18.976 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
11362:M 12 Jun 09:12:18.976 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
11362:M 12 Jun 09:12:18.976 * Ready to accept connections

#解决警告1
临时解决： echo 511 > proc/sys/net/core/somaxconn 
永久解决：将 net.core.somaxconn = 511 写入/etc/sysctl.conf
#解决警告2
临时解决：echo 1 > /proc/sys/vm/overcommit_memory
永久解决：将 overcommit_memory=1 写入/etc/sysctl.conf
#解决警告3
临时解决：echo never > /sys/kernel/mm/transparent_hugepage/enabled
永久解决：将 echo never > /sys/kernel/mm/transparent_hugepage/enabled 写入/etc/re.local
```

- 创建启动脚本和用户

```bash
#创建启动脚本
[root@centos ~]$vim /usr/lib/systemd/system/redis.service
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
#ExecStart=/usr/bin/redis-server /etc/redis.conf --supervised systemd
ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Type=notify
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
[Install]
WantedBy=multi-user.target

[root@centos ~]$useradd -r -s /sbin/nologin redis
[root@centos ~]$chown -R redis.redis /apps/redis/
#使用命令启动测试
[root@centos ~]$systemctl start redis
[root@centos ~]$ss -nlt
State       Recv-Q Send-Q                     Local Address:Port                                    Peer Address:Port              
LISTEN      0      511                            127.0.0.1:6379                                               *:*                  
LISTEN      0      128                                    *:22                                                 *:*                  
LISTEN      0      100                            127.0.0.1:25                                                 *:*                  
LISTEN      0      128                                   :::22                                                :::*                  
LISTEN      0      100                                  ::1:25                                                :::*                  
[root@centos ~]$redis-cli
127.0.0.1:6379> info
...
```

#### - 2.2.3：redis配置文件

```bash
bind 0.0.0.0 #监听地址，可用空格隔开后监听多个IP
protected-mode yes #在没有设置bind IP和密码的时候只允许访问127.0.0.1:6379
port 6379 #监听端口
tcp-backlog 511 #三次握手的时候server端收到client ack确认号之后的队列值
timeout 0 #客户端和redis服务器端的连接超时时间，默认为0用不超时
tcp-keepalive 300 #tcp会话保持时间
daemonize yes #认情况下redis不是作为守护进程运行的，如果你想让它在后台运行，你就把它改成yes,当redis作为守护进程运行的时候，它会写一个pid到 /var/run/redis.pid文件里面
supervised no #和操作系统相关参数，可以设置通过upstart和systemd管理Redis守护进程，centos7以后都使用systemd
pidfile /var/run/redis_6379.pid #pid文件路径
loglevel notice #日志级别
logfile "/apps/redis/logs/redis_6379.log""" #日志路径，按实际情况配置
databases 16 #设置db库数量，默认16个库
always-show-logo yes #在启动redis时是否显示 log
save 900 1 #在900秒内有一个键内容发生更改就出就快照机制
save 300 10
save 60 10000
stop-writes-on-bgsave-error no #快照出错时是否禁止redis写入操作，此选项一定要设置为no！
rdbcompression yes #持久化到RDB文件时，是否压缩，"yes"为压缩，"no"则反之
rdbchecksum yes #是否开启RC64校验，默认是开启
dbfilename dump.rdb #快照文件名
dir /apps/redis/data #快照文件保存路径
replica-serve-stale-data yes #当从库同主库失去连接或者复制正在进行，从机库有两种运行方式：1) 如果 replica-serve-stale-data 设置为 yes(默认设置)，从库会继续响应客户端的读请求。2) 如果replica-serve-stale-data设置为no，除去指定的命令之外的任何请求都会返回一个错误"SYNC with master inprogress"。
replica-read-only yes #是否设置从库只读
repl-diskless-sync no #是否使用socket方式复制数据，目前redis复制提供两种方式，disk和socket，如果新的 slave连上来或者重连的slave无法部分同步，就会执行全量同步，master会生成rdb文件，有2种方式：disk方式是 master创建一个新的进程把rdb文件保存到磁盘，再把磁盘上的rdb文件传递给slave，socket是master创建一个新的进程，直接把rdb文件以socket的方式发给slave，disk方式的时候，当一个rdb保存的过程中，多个slave都能共享这个 rdb文件，socket的方式就是一个个slave顺序复制，只有在磁盘速度缓慢但是网络相对较快的情况下才使用socket方式，否则使用默认的disk方式
repl-diskless-sync-delay 30 #diskless复制的延迟时间，设置0为关闭，一旦复制开始还没有结束之前，maste 节点不会再接收新slave的复制请求，直到下一次开始
repl-ping-slave-period 10 #slave 根据 master 指定的时间进行周期性的 PING 监测
repl-timeout 60 #复制链接超时时间，需要大于 repl-ping-slave-period，否则会经常报超时
repl-disable-tcp-nodelay no #在 socket 模式下是否在 slave 套接字发送 SYNC 之后禁用 TCP_NODELAY，如果你选择“yes”Redis 将使用更少的 TCP 包和带宽来向 slaves 发送数据。但是这将使数据传输到 slave上有延迟，Linux 内核的默认配置会达到 40 毫秒，如果你选择了 "no" 数据传输到 salve 的延迟将会减少但要使用更多的带宽
repl-backlog-size 1mb #复制缓冲区大小，只有在 slave 连接之后才分配内存。
repl-backlog-ttl 3600 #多次时间 master 没有 slave 连接，就清空 backlog 缓冲区。
replica-priority 100 #当 master 不可用，Sentinel 会根据 slave 的优先级选举一个 master。最低的优先级的 slave，当选 master。而配置成 0，永远不会被选举。
requirepass foobared #设置 redis 连接密码
rename-command #重命名一些高危命令
maxclients 10000 #最大连接客户端
maxmemory #最大内存，单位为bytes 字节，8G内存的计算方式 8(G)*1024(MB)*1024(KB)*1024(Kbyte)，需要注意的是 slave 的输出缓冲区是不计算在 maxmemory 内。
appendonly no #是否开启 AOF 日志记录， 默认 redis 使用的是 rdb 方式持久化，这种方式在许多应用中已经足够用了。但是 redis 如果中途宕机，会导致可能有几分钟的数据丢失，根据 save 来策略进行持久化，Append Only File 是另一种持久化方式，可以提供更好的持久化特性。Redis 会把每次写入的数据在接收后都写入 appendonly.aof 文件，每次启动时 Redis 都会先把这个文件的数据读入内存里，先忽略 RDB 文件。
appendfilename "appendonly.aof" #AOF 文件名
appendfsync everysec #aof 持久化策略的配置 , no表示不执行fsync,由操作系统保证数据同步到磁盘,always 表示每次写入都执行 fsync，以保证数据同步到磁盘,everysec 表示每秒执行一次 fsync，可能会导致丢失这 1s 数据。no-appendfsync-on-rewrite no 在 aof rewrite 期间,是否对 aof 新记录的 append 暂缓使用文件同步策略,主要考虑磁盘 IO 开支和请求阻塞时间。默认为 no,表示"不暂缓",新的 aof 记录仍然会被立即同步，Linux 的默认 fsync 策略是 30 秒，如果为 yes 可能丢失 30 秒数据，但由于 yes 性能较好而且会避免出现阻塞因此比较推荐。
auto-aof-rewrite-percentage 100 # 当 Aof log 增长超过指定百分比例时，重写 log file， 设置为 0 表示不自动重写 Aof 日志，重写是为了使 aof 体积保持最小，而确保保存最完整的数据。
auto-aof-rewrite-min-size 64mb #触发 aof rewrite 的最小文件大小
aof-load-truncated yes #是否加载由于其他原因导致的末尾异常的 AOF 文件(主进程被 kill/断电等)
aof-use-rdb-preamble yes #redis4.0 新增 RDB-AOF 混合持久化格式，在开启了这个功能之后，AOF 重写产生的文件将同时包含 RDB 格式的内容和 AOF 格式的内容，其中 RDB 格式的内容用于记录已有的数据，而 AOF 格式的内存则用于记录最近发生了变化的数据，这样 Redis 就可以同时兼有 RDB 持久化和AOF 持久化的优点（既能够快速地生成重写文件，也能够在出现问题时，快速地载入数据）。
lua-time-limit 5000 #lua 脚本的最大执行时间，单位为毫秒
cluster-enabled yes #是否开启集群模式，默认是单机模式
cluster-config-file nodes-6379.conf #由 node 节点自动生成的集群配置文件
cluster-node-timeout 15000 #集群中 node 节点连接超时时间
cluster-replica-validity-factor 10 #在执行故障转移的时候可能有些节点和master断开一段时间数据比较旧，这些节点就不适用于选举为 master，超过这个时间的就不会被进行故障转移
cluster-migration-barrier 1 #一个主节点拥有的至少正常工作的从节点，即如果主节点的slave节点故障后会将多余的从节点分配到当前主节点成为其新的从节点。
cluster-require-full-coverage no #集群槽位覆盖，如果一个主库宕机且没有备库就会出现集群槽位不全，那么 yes 情况下 redis 集群槽位验证不全就不再对外提供服务，而 no 则可以继续使用但是会出现查询数据查不到的情况(因为有数据丢失)。
#Slow log 是Redis用来记录查询执行时间的日志系统，slow log保存在内存里面，读写速度非常快，因此你可以放心地使用它，不必担心因为开启 slow log 而损害 Redis 的速度。
slowlog-log-slower-than 10000 #以微秒为单位的慢日志记录，为负数会禁用慢日志，为0会记录每个命令操作。
slowlog-max-len 128 #记录多少条慢日志保存在队列，超出后会删除最早的，以此滚动删除
```

### - 2.3：redis持久化

​	redis是一个内存级别的缓存程序，即redis是使用内存进行数据的缓存的，但是其可以将内存中的数据按照一定的策略保存到硬盘上，从而实现数据持久保存的目的，redis支持两种不同方式的数据持久化保存机制，分别是RDB和AOF

#### - 2.3.1：RDB模式

- RDB：基于时间的快照，只保留当前最新的一次快照，特点实质性速度快，缺点是可能丢失上次快照到当前快照未完成时之间的数据
- 实现过程：redis从主进程先fork出一个子进程，使用写时复制机制，子进程将内存的数据保存为一个临时文件，当数据保存完成后再将上一次保存的RDB文件替换掉，然后关闭子进程，这样可以使每一次做RDB快照时保存的数据都是完整的，因为直接替换掉RDB文件的时候可能会出现突然断电等问题而导致RDB文件还没有保存完就关机停止保存导致数据丢失的情况。也可以手动将每次生成的RDB文件进程备份，这样可以最大化保存历史数据

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560309604853.png)

- 优点：
  - RDB快照保存了某个时间点的数据，可以通过脚本执行 bgsave(非阻塞)或者 save(阻塞)命令自定义时间点备份，可以保留多个备份，当出现问题可以恢复到不同时间点的版本
  - 可以最大化 IO 的性能，因为父进程在保存 RDB 文件的时候唯一要做的是 fork 出一个子进程，然后的操作都会有这个子进程操作，父进程无需任何的 IO 操作
  - RDB在大量数据的恢复速度上比AOF快
- 缺点：
  - 不能实时的保存数据，会丢失自上一次执行 RDB 备份到当前的内存数据
  - 数据量非常大的时候，从父进程 fork 的时候需要一点时间，可能是毫秒或者秒

#### - 2.3.2：AOF模式

- AOF：按照操作顺序依次将操作添加到指定的日志文件当中，特点是数据安全性相对较高，缺点是即使有些操作是重复的也会全部记录
- AOF 和 RDB 一样使用了写时复制机制，AOF 默认为每秒钟 fsync 一次，即将执行的命令保存到 AOF 文件当中，这样即使 redis 服务器发生故障的话顶多也就丢失 1 秒钟之内的数据，也可以设置不同的 fsync策略，或者设置每次执行命令的时候执行 fsync，fsync 会在后台执行线程，所以主线程可以继续处理用户的正常请求而不受到写入 AOF 文件的 IO 影响
- 优缺点：
  - AOF 的文件大小要大于 RDB 格式的文件
  - 根据所使用的 fsync 策略(fsync 是同步内存中 redis 所有已经修改的文件到存储设备)，默认是appendfsync everysec 即每秒执行一次 fsync

### - 2.4：redis数据类型

http://www.redis.cn/topics/data-types.htm

#### - 2.4.1：字符串

- redis 中所有的 key 的类型都是字符串，命令不区分大小写

```bash
#添加一个key
127.0.0.1:6379> set key1 value1
OK
#查看key的类型
127.0.0.1:6379> type key1
string
#设置过期时间
127.0.0.1:6379> set key1 value1 ex 3 #为键值设置自动过期时间
OK
#获取一个key的内容
127.0.0.1:6379> get key1
"value1"
#删除一个key
127.0.0.1:6379> DEL key2
(integer) 1
#批量设置多个key
127.0.0.1:6379> MSET key1 value1 key2 value2 
OK
#批量查看多个key
127.0.0.1:6379> MGET key1 key2
1) "value1"
2) "value2"
#追加数据
127.0.0.1:6379> APPEND key1 hello
(integer) 11
127.0.0.1:6379> get key1
"value1hello"
#数值递增
127.0.0.1:6379> set num 1
OK
127.0.0.1:6379> INCR num
(integer) 2
127.0.0.1:6379> INCR num
(integer) 3
127.0.0.1:6379> INCR num
(integer) 4
#数值递减
127.0.0.1:6379> DECR num
(integer) 3
127.0.0.1:6379> DECR num
(integer) 2
127.0.0.1:6379> DECR num
(integer) 1
#返回字符串key的长度
127.0.0.1:6379> STRLEN key1
(integer) 11
```

#### - 2.4.2：列表

- 列表是一个双向可读写的管道，其头部是左侧尾部是右侧，一个列表最多可以包含2^32-1个元素

```bash
#生成列表并插入数据
127.0.0.1:6379> LPUSH list1 jack rose tom jerry
(integer) 4
127.0.0.1:6379> type list1
list
#向列表追加数据
127.0.0.1:6379> LPUSH list1 javis #从左侧追加即头部追加
(integer) 5
127.0.0.1:6379> RPUSH list1 alice #从右侧追加即尾部追加
(integer) 6
#获取列表长度
127.0.0.1:6379> LLEN list1
(integer) 6
#移除列表数据
127.0.0.1:6379> RPOP list1 #移除最后一个
"alice"
127.0.0.1:6379> LPOP list1 #移除第一个
"javis"
```

#### - 2.4.3：集合（set）

- set时string类型的无序集合，集合成员是唯一的，不能出现重复的数据

```bash
#生成集合key
127.0.0.1:6379> SADD set1 v1
(integer) 1
127.0.0.1:6379> SADD set2 v2 v4
(integer) 2
127.0.0.1:6379> type set1
set
#追加数值：追加的时候不能追加已经存在的数值
127.0.0.1:6379> SADD set1 v2 v3 v4
(integer) 3
127.0.0.1:6379> SADD set1 v2 #追加已存在的数据则显示不成功
(integer) 0
#查看集合的所有数据
127.0.0.1:6379> SMEMBERS set1
1) "v2"
2) "v3"
3) "v1"
4) "v4"
#获取集合的差集：属于集合A而不属于集合B的元素
127.0.0.1:6379> SDIFF set1 set2 #查看属于set1不属于set2的元素
1) "v1"
2) "v3"
127.0.0.1:6379> SDIFF set2 set1 #查看属于set1不属于set1的元素
(empty list or set)
#获取集合的交集：既属于集合A又属于集合B的元素
127.0.0.1:6379> SINTER set1 set2
1) "v2"
2) "v4"
#获取集合的并集：集合A和集合B中的全部元素
127.0.0.1:6379> SUNION set1 set2
1) "v3"
2) "v1"
3) "v4"
4) "v2"
```

#### - 2.4.4：有序集合（sorted set）

​	Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员，不同的是每个元素都会关联一个 double(双精度浮点型)类型的分数，redis 正是通过分数来为集合中的成员进行从小到大的排序，有序集合的成员是唯一的,但分数(score)却可以重复，集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)， 集合中最大的成员数为 2^32 - 1

```bash
#生成有序集合
127.0.0.1:6379> ZADD zset1 1 v1
(integer) 1
127.0.0.1:6379> ZADD zset1 2 v2
(integer) 1
127.0.0.1:6379> ZADD zset1 3 v3
(integer) 1
127.0.0.1:6379> type zset1
zset
#排行案例：
127.0.0.1:6379> ZADD paihangbang 10 key1 20 key2 30 key33
(integer) 3
127.0.0.1:6379> ZREVRANGE paihangbang 0 -1 withscores #显示指定集合内所有key和得分情况，0 -1表示显示集合内所有值
1) "key3"
2) "30"
3) "key2"
4) "20"
5) "key1"
6) "10"
#批量添加多个数值
127.0.0.1:6379> ZADD zset2 1 v1 2 v2 3 v3 
(integer) 3
#获取集合的长度
127.0.0.1:6379> ZCARD zset2
(integer) 3
127.0.0.1:6379> ZCARD zset1
(integer) 3
#基于索引返回数值
127.0.0.1:6379> ZRANGE zset1 1 3
1) "v2"
2) "v3"
127.0.0.1:6379> ZRANGE zset1 0 2
1) "v1"
2) "v2"
3) "v3"
#返回某个数值的索引
127.0.0.1:6379> ZRANk zset1 v2
(integer) 1
127.0.0.1:6379> ZRANk zset1 v3
(integer) 2
127.0.0.1:6379> ZRANk zset1 v1
(integer) 0
```

#### - 2.4.5：哈希（hash）

​	hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象,Redis 中每个 hash 可以存储 2^32 - 1个 键值对

```bash
#生成hash key
127.0.0.1:6379> HSET hset1 name javis age 20
(integer) 2
127.0.0.1:6379> type hset1
hash
#获取hash key字段值
127.0.0.1:6379> HGET hset1 name
"javis"
127.0.0.1:6379> HGET hset1 age
"20"
#删除一个hash key的字段
127.0.0.1:6379> HDEL hset1 age
(integer) 1
#获取所有hash表中的字段
127.0.0.1:6379> HSET hset1 name tom age 20
(integer) 1
127.0.0.1:6379> HKEYS hset1
1) "name"
2) "age"
```

### - 2.5：消息队列

- 分为生产者消费者模式和发布者订阅者模式

#### - 2.5.1：生产者消费者模式

- 模式原理：

​	在生产者消费者(Producer/Consumer)模式下，上层应用接收到的外部请求后开始处理其当前步骤的操作，在执行完成后将已经完成的操作发送至指定的频道(channel)当中，并由其下层的应用监听该频道并继续下一步的操作，如果其处理完成后没有下一步的操作就直接返回数据给外部请求，如果还有下一步的操作就再将任务发布到另外一个频道，由另外一个消费者继续监听和处理

- 模式介绍：

​	生产者消费者模式下，多个消费者同时监听一个队列，但是一个消息只能被最先抢到消息的消费者消费，即消息任务是一次性读取和处理，此模式在分布式业务架构中非常常用，比较常用的软件还有RabbitMQ、Kafka、RocketMQ、ActiveMQ 等

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560322803438.png)

- 队列介绍：

​	队列当中的 消息由不同的生产者写入也会有不同的消费者取出进行消费处理，但是一个消息一定是只能被取出一次也就是被消费一次

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560322904708.png)

```bash
#生产者发布消息
127.0.0.1:6379> LPUSH channel1 msg1 #从管道左侧写入
(integer) 1
127.0.0.1:6379> LPUSH channel1 msg2
(integer) 2
127.0.0.1:6379> LPUSH channel1 msg3
(integer) 3
127.0.0.1:6379> LPUSH channel1 msg4
(integer) 4
#查看队列消息
127.0.0.1:6379> LRANGE channel1 0 -1
1) "msg4"
2) "msg3"
3) "msg2"
4) "msg1"
#消费者消费消息
127.0.0.1:6379> RPOP channel1 #从管道的右侧消费
"msg1"
127.0.0.1:6379> RPOP channel1
"msg2"
127.0.0.1:6379> RPOP channel1
"msg3"
127.0.0.1:6379> RPOP channel1
"msg4"
127.0.0.1:6379> RPOP channel1 #队列中的消息已被消费完毕
(nil)
```

#### - 2.5.2：发布者订阅者模式

- 模式介绍：

​	在发布者订阅者模式下，发布者将消息发布到指定的 channel 里面，凡是监听该 channel 的消费者都会收到同样的一份消息，这种模式类似于是收音机模式，即凡是收听某个频道的听众都会收到主持人发布的相同的消息内容。
​	此模式常用语群聊天、群通知、群公告等场景。
Subscriber：订阅者
Publisher：发布者
Channel：频道

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560323192059.png)

```bash
#消费者监听频道
127.0.0.1:6379> SUBSCRIBE channel1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel1"
3) (integer) 1
#发布者发布消息
127.0.0.1:6379> PUBLISH channel1 test1
(integer) 1
127.0.0.1:6379> PUBLISH channel1 test2
(integer) 1
#订阅者验证消息
127.0.0.1:6379> SUBSCRIBE channel1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel1"
3) (integer) 1
1) "message"
2) "channel1"
3) "test1"
1) "message"
2) "channel1"
3) "test2"
#订阅多个频道
127.0.0.1:6379> SUBSCRIBE channel1 channel2
#订阅所有频道
127.0.0.1:6379> SUBSCRIBE *
#订阅匹配到的频道
127.0.0.1:6379> SUBSCRIBE channel*
```

#### - 2.5.3：redis其他命令

##### - 2.5.3.1：CONFIG

​	config用于查看当前redis配置、以及不重启更改redis配置等

```bash
#更改最大内存
127.0.0.1:6379> CONFIG set maxmemory 8589934592
OK
127.0.0.1:6379> CONFIG get maxmemory
1) "maxmemory"
2) "8589934592"
#设置连接密码，设置完成后立即生效
127.0.0.1:6379> CONFIG SET requirepass centos
OK
127.0.0.1:6379> CONFIG GET requirepass
1) "requirepass"
2) "centos"
#查看当前配置
127.0.0.1:6379> CONFIG GET *
...
```

##### - 2.5.3.2：INFO

​	显示当前节点redis运行状态显示信息

```bash
127.0.0.1:6379> info 
...
```

##### - 2.5.3.3：SELECT

```bash
127.0.0.1:6379> SELECT 1 
OK
```

##### - 2.5.3.4：KEYS

```bash
#查看当前库下的所有key
127.0.0.1:6379> KEYS *
 1) "hset1"
 2) "set1"
 3) "key1"
 4) "key2"
 5) "zset1"
 6) "paihangbang"
 7) "set2"
 8) "num"
 9) "list1"
10) "zset2"
```

##### - 2.5.3.5：BGSAVE

```bash
#手动在后台执行RDB持久化操作
127.0.0.1:6379> BGSAVE
Background saving started
```

##### - 2.5.3.6：DBSIZE

```bash
#返回当前库下的所有key数量
127.0.0.1:6379> DBSIZE
(integer) 10
```

##### - 2.5.3.7：FLUSHDB

​	强制清空当前库中的所有key

##### - 2.5.3.8：FLUSHALL

​	强制清空当前redis服务器所有数据库中的所有key，即删除所有数据

## 三、redis高可用与集群

​	解决redis的单点宕机问题

### - 3.1：redis主从配置

​	程序端连接到高可用负载的 VIP，然后连接到负载服务器设置的 Redis 后端 real server，此模式不需要在程序里面配置 Redis 服务器的真实 IP 地址，当后期 Redis 服务器 IP 地址发生变更只需要更改 redis相应的后端 real server 即可，可避免更改程序中的 IP 地址设置

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560326813169.png)

#### - 3.1.1：slave主要配置

​	redis slave也要开启持久化并设置和 master 同样的连接密码，因为后期 slave 会有提升为 master 的可能,Slave 端切换 master 同步后会丢失之前的所有数据

​	一旦某个 Slave 成为一个 master 的 slave，Redis Slave 服务会清空当前 redis 服务器上的所有数据并将master 的数据导入到自己的内存，但是断开同步关系后不会删除当前已经同步过的数据

```bash
#命令行配置
#当前状态为master，需要转换为slave角色并执行master服务器的IP+PORT+PASSWORD
127.0.0.1:6379> SLAVEOF 192.168.30.66 6379 
OK
127.0.0.1:6379> CONFIG SET masterauth centos
OK
#查看日志同步过程
[root@centos ~]$tail /apps/redis/logs/redis_6379.log 
8664:S 12 Jun 16:45:53.637 * Connecting to MASTER 192.168.30.66:6379
8664:S 12 Jun 16:45:53.638 * MASTER <-> SLAVE sync started
8664:S 12 Jun 16:45:53.638 * Non blocking connect for SYNC fired the event.
8664:S 12 Jun 16:45:53.639 * Master replied to PING, replication can continue...
8664:S 12 Jun 16:45:53.640 * Partial resynchronization not possible (no cached master)
8664:S 12 Jun 16:45:53.641 * Full resync from master: 1278ee21a9ef33ec30c3db7cf8cfdd166149cdb5:1022
8664:S 12 Jun 16:45:53.674 * MASTER <-> SLAVE sync: receiving 459 bytes from master
8664:S 12 Jun 16:45:53.674 * MASTER <-> SLAVE sync: Flushing old data
8664:S 12 Jun 16:45:53.674 * MASTER <-> SLAVE sync: Loading DB in memory
8664:S 12 Jun 16:45:53.674 * MASTER <-> SLAVE sync: Finished with success
#查看当前slave状态
127.0.0.1:6379> INFO replication
# Replication
role:slave
master_host:192.168.30.66
master_port:6379
master_link_status:up
```

```bash
#保存配置到redis.conf
[root@centos ~]$vim /apps/redis/etc/redis.conf 
286 replicaof 192.168.30.66 6379
293 masterauth centos #master的密码
#重启验证
[root@centos ~]$systemctl restart redis
127.0.0.1:6379> INFO replication
# Replication
role:slave
master_host:192.168.30.66
master_port:6379
master_link_status:up
...
```

```bash
#验证slave数据
127.0.0.1:6379> KEYS *
 1) "list1"
 2) "paihangbang"
 3) "zset1"
 4) "key1"
 5) "set2"
 6) "set1"
 7) "num"
 8) "zset2"
 9) "hset1"
10) "key2"
#默认状态下slave状态只读无法写入数据
127.0.0.1:6379> set k1 v2
(error) READONLY You can't write against a read only slave. #可以修改配置文件的slave-read-only yes为no
```

```bash
#查看master日志
[root@centos ~]$tail /apps/redis/logs/redis_6379.log 
12221:M 12 Jun 16:45:53.641 * Full resync requested by slave 192.168.30.86:6379
12221:M 12 Jun 16:45:53.641 * Starting BGSAVE for SYNC with target: disk
12221:M 12 Jun 16:45:53.641 * Background saving started by pid 13123
13123:C 12 Jun 16:45:53.642 * DB saved on disk
13123:C 12 Jun 16:45:53.643 * RDB: 0 MB of memory used by copy-on-write
12221:M 12 Jun 16:45:53.673 * Background saving terminated with success
12221:M 12 Jun 16:45:53.674 * Synchronization with slave 192.168.30.86:6379 succeeded
12221:M 12 Jun 16:54:54.386 # Connection with slave 192.168.30.86:6379 lost.
12221:M 12 Jun 16:54:54.402 * Slave 192.168.30.86:6379 asks for synchronization
12221:M 12 Jun 16:54:54.403 * Partial resynchronization request from 192.168.30.86:6379 accepted. Sending 756 bytes of backlog starting from offset 1023.
```

#### - 3.1.2：主从复制过程

​	Redis 支持主从复制分为全量同步和增量同步，首次同步是全量同步，主从同步可以让从服务器从主服务器备份数据，而且从服务器还可以再有从服务器，即另外一台 redis 服务器可以从一台从服务器进行数据同步，redis 的主从同步是非阻塞的，其收到从服务器的 sync(2.8 版本之前是 PSYNC)命令会fork 一个子进程在后台执行 bgsave 命令，并将新写入的数据写入到一个缓冲区里面，bgsave 执行完成之后并生成的将 RDB 文件发送给客户端，客户端将收到后的 RDB 文件载入自己的内存，然后主 redis将缓冲区的内容在全部发送给从 redis，之后的同步从服务器会发送一个 offset 的位置(等同于 MySQL的 binlog 的位置)给主服务器，主服务器检查后位置没有错误将此位置之后的数据包括写在缓冲区的积压数据发送给 redis 从服务器，从服务器将主服务器发送的积压数据写入内存，这样一次完整的数据同步，再之后再同步的时候从服务器只要发送当前的 offset 位 置给主服务器，然后主服务器根据响应的位置将之后的数据发送给从服务器保存到其内存即可

​	Redis 全量复制一般发生在 Slave 初始化阶段，这时 Slave 需要将 Master 上的所有数据都复制一份。具体步骤如下：
1）从服务器连接主服务器，发送 SYNC 命令；
2）主服务器接收到 SYNC 命令后，开始执行 BGSAVE 命令生成 RDB 快照文件并使用缓冲区记录此后执行的所有写命令；
3）主服务器 BGSAVE 执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令；
4）从服务器收到快照文件后丢弃所有旧数据，载入收到的快照；
5）主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令；
6）从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；
7）后期同步会先发送自己 slave_repl_offset 位置，只同步新增加的数据，不再全量同步。

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560330997098.png)

#### - 3.1.3：主从同步优化

```bash
repl-diskless-sync yes #yes为支持disk，master将RDB文件先保存到磁盘在发送给slave，no为maste直接将RDB文件发送给 slave，默认即为使用no，Master RDB件不需要与磁盘交互
repl-diskless-sync-delay 5 #Master准备好RDB文件后等待传输时间
repl-ping-slave-period 10 #slave端向server端发送ping的时间区间设置，默认为10秒
repl-timeout 60 #设置超时时间
repl-disable-tcp-nodelay no #是否启用TCP_NODELAY，如设置成yes，则redis会合并小的TCP包从而节省带宽，但会增加同步延迟（40ms），造成 master与slave数据不一致，假如设置成no，则redismaster会立即发送同步数据，没有延迟，前者关注性能，后者关注一致性
repl-backlog-size 1mb #master的写入数据缓冲区，用于记录自上一次同步后到下一次同步过程中间的写入命令，计算公式：repl-backlog-size=允许从节点最大中断时长 * 主实例offset每秒写入量，比如master每秒最大写入64mb，最大允许 60 秒，那么就要设置为 64mb*60 秒=3840mb(3.8G)
repl-backlog-ttl 3600 #如果一段时间后没有slave连接到master，则backlog size的内存将会被释放。如果值为0则表示永远不释放这部份内存。
slave-priority 100 #slave 端的优先级设置，值是一个整数，数字越小表示优先级越高。当master故障时将会按照优先级来选择slave端进行恢复，如果值设置为0，则表示该slave永远不会被选择。
#min-slaves-to-write 0 #
#min-slaves-max-lag 10 #设置当一个master端的可用slave少于N个，延迟时间大于M秒时，不接收写操作。
Master的重启会导致master_replid发生变化，slave之前的master_replid就和master不一致从而会引发所有slave的全量同步
```

#### - 3.1.4：salave切换master

```bash
#取消slave状态即变为master
127.0.0.1:6379> SLAVEOF no one
OK
#停止slave之后查看当前状态
127.0.0.1:6379> INFO replication
# Replication
role:master
connected_slaves:0
master_replid:20faae1ad737a2ed5a6c5c873481cb839e491d01
master_replid2:2cc621f855ee6b281e0d3f6099b8d360e24c490a
master_repl_offset:1470
second_repl_offset:1471
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1470
#测试能否写入数据
127.0.0.1:6379> set key3 value3
OK
```

#### - 3.1.5：创建slave节点的slave节点

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Redis/1560338373811.png)

```bash
#在有slave差的"master"查看状态
127.0.0.1:6379> INFO replication
# Replication
role:slave
master_host:192.168.30.66
master_port:6379
master_link_status:up
master_last_io_seconds_ago:8 #最后一次与master通信已过去多少秒
master_sync_in_progress:0 #是否正在与master通信
slave_repl_offset:2702 #当前同步的偏移量
slave_priority:100 #slave优先级，master故障后值越小越优先同步
slave_read_only:1
connected_slaves:1
slave0:ip=192.168.30.76,port=6379,state=online,offset=2702,lag=0
master_replid:2cc621f855ee6b281e0d3f6099b8d360e24c490a
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2702
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:2702
```

### - 3.2：redis集群

redis集群解决了单台redis无法解决的两个核心问题：

- master和slave角色的无缝切换，让业务无感知从而不影响业务使用
- 可以横向动态扩展 Redis 服务器，从而实现多台服务器并行写入以实现更高并发的目的

redis集群实现方式：

- 客户端分片 
- 代理分片
- redis cluster

#### - 3.2.1：Sentinel（哨兵）

- 编辑配置文件（前提是已经为redis配置好了主从同步）

```bash
#master配置
[root@centos ~]$vim /usr/local/src/redis-4.0.14/sentinel.conf
bind 0.0.0.0
port 26379
daemonize yes
pidfile "/apps/redis/run/redis-sentinel.pid"
logfile "/apps/redis/run/sentinel_26379.log"
dir "/apps/redis"
sentinel monitor mymaster 192.168.30.66 6379 2
sentinel auth-pass mymaster centos
sentinel down-after-milliseconds mymaster 30000 #(SDOWN)主观下线的时间,30s
sentinel parallel-syncs mymaster 1 #发生故障转移时候同时向新master同步数据的slave数量，数字越小总同步时间越长
sentinel failover-timeout mymaster 180000 #所有slaves指向新的master所需的超时时间
sentinel deny-scripts-reconfig yes

#slave1配置(slave2配置相同)
[root@centos ~]$vim /usr/local/src/redis-4.0.14/sentinel.conf
bind 192.168.30.76
port 26379
daemonize yes
pidfile "/apps/redis/run/redis-sentinel.pid"
logfile "/apps/redis/run/sentinel_26379.log"
dir "/apps/redis"
sentinel deny-scripts-reconfig yes
sentinel monitor mymaster 192.168.30.66 6379 2
sentinel auth-pass mymaster centos
```

- 启动哨兵

```bash
#三个哨兵需要同时启动，用shell发送全部会话功能
redis-sentinel /usr/local/src/redis-4.0.14/sentinel.conf
```

- 验证端口

```bash
#master端口
[root@centos redis-4.0.14]$ss -nlt
State       Recv-Q Send-Q          Local Address:Port                         Peer Address:Port              
LISTEN      0      511                         *:26379                                   *:*                  
LISTEN      0      511                         *:6379                                    *:*                  
LISTEN      0      128                         *:22                                      *:*                  
LISTEN      0      100                 127.0.0.1:25                                      *:*                  
LISTEN      0      128                        :::22                                     :::*                  
LISTEN      0      100                       ::1:25                                     :::*  
```

- 哨兵日志

```bash
#master日志
[root@centos ~]$tail /apps/redis/run/sentinel_26379.log 
14507:X 13 Jun 20:24:05.592 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
14507:X 13 Jun 20:24:05.592 # Redis version=4.0.14, bits=64, commit=00000000, modified=0, pid=14507, just started
14507:X 13 Jun 20:24:05.592 # Configuration loaded
14509:X 13 Jun 20:24:05.594 * Increased maximum number of open files to 10032 (it was originally set to 1024).
14509:X 13 Jun 20:24:05.595 * Running mode=sentinel, port=26379.
14509:X 13 Jun 20:24:05.595 # Sentinel ID is b34e084539e169c4eb76140984de540d434f2d5d
14509:X 13 Jun 20:24:05.595 # +monitor master mymaster 192.168.30.66 6379 quorum 2 #监控成功，2表示投票选举超过两人同意时才选举成功
14510:X 13 Jun 20:24:05.597 * Increased maximum number of open files to 10032 (it was originally set to 1024).
14510:X 13 Jun 20:24:05.597 # Creating Server TCP listening socket 0.0.0.0:26379: bind: Address already in use #监听端口成功
14509:X 13 Jun 20:24:07.661 * +sentinel sentinel 167356ace4af05481aa402cd35ca9a016e4a9d1c 192.168.30.86 26379 @ mymaster 192.168.30.66 6379
```

- 当前redis状态

```bash
#master状态
[root@centos ~]$redis-cli 
127.0.0.1:6379> AUTH centos
OK
127.0.0.1:6379> INFO replication
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.30.86,port=6379,state=online,offset=304956,lag=0
slave1:ip=192.168.30.76,port=6379,state=online,offset=304956,lag=1
master_replid:2cc621f855ee6b281e0d3f6099b8d360e24c490a
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:304956
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:304956
```

- 当前sentinel状态

```bash
#以master为例
[root@centos ~]$redis-cli -p 26379
127.0.0.1:26379> INFO sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=192.168.30.66:6379,slaves=2,sentinels=3 #最后一行包括了masterIP，slave数量，sentinel数量
```

- 停止redis master测试故障转移

```bash
#停止master服务
[root@centos ~]$systemctl stop redis

#查看哨兵信息
192.168.30.76:26379> INFO sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=192.168.30.86:6379,slaves=2,sentinels=3

#查看故障转移日志
[root@centos ~]$tail /apps/redis/run/sentinel_26379.log 
...
14509:X 13 Jun 21:00:12.658 # +switch-master mymaster 192.168.30.66 6379 192.168.30.86 6379
14509:X 13 Jun 21:00:12.658 * +slave slave 192.168.30.76:6379 192.168.30.76 6379 @ mymaster 192.168.30.86 6379
14509:X 13 Jun 21:00:12.658 * +slave slave 192.168.30.66:6379 192.168.30.66 6379 @ mymaster 192.168.30.86 6379
14509:X 13 Jun 21:00:42.683 # +sdown slave 192.168.30.66:6379 192.168.30.66 6379 @ mymaster 192.168.30.86 6379
```

- 故障转移后的redis配置文件

```bash
#故障转移后redis.conf中的replicaof行的master IP被修改
[root@centos ~]$grep "^[a-z]" /apps/redis/etc/redis.conf
...
slaveof 192.168.30.86 6379

#故障转以后sentinel.conf中的sentinel monitor IP会被修改
...
sentinel leader-epoch mymaster 1
sentinel known-slave mymaster 192.168.30.66 6379
sentinel known-slave mymaster 192.168.30.76 6379
sentinel known-sentinel mymaster 192.168.30.86 26379 167356ace4af05481aa402cd35ca9a016e4a9d1c
sentinel known-sentinel mymaster 192.168.30.76 26379 824563bfc2d4a0663210fbfbe70d55a45964d33a
sentinel current-epoch 1
```

- 故障转移后的redis状态

```bash
127.0.0.1:6379> INFO replication
# Replication
role:slave
master_host:192.168.30.86 #故障转移后新的master IP
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:653035
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:48a3b72e0910f990246cba25d16eda1825073d2f #故障转移后的当前master_replid
master_replid2:2cc621f855ee6b281e0d3f6099b8d360e24c490a #故障转移后之前的master_replid
master_repl_offset:653035
second_repl_offset:477138
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2633
repl_backlog_histlen:650403
```

#### - 3.2.2：Redis Cluster

- 实验准备：6台主机，全新环境，全部编译安装redis

- 创建cluster集群的前提

```bash
1.每个redis node节点采用相同的硬件配置，相同的密码
2.每个节点必须开启的参数
cluster-enabled yes #必须开启集群状态，开启后redis进程会有cluster显示
cluster-config-file nodes-6380.conf #此文件有redis cluster集群自动创建和维护，不需要任何手动操作
3.所有redis服务器必须没有任何数据
4.先启动为单机redis且没有任何key value
```

- 开启服务后查看端口

```bash
[root@centos ~]$ss -nlt
State       Recv-Q Send-Q    Local Address:Port                               Peer Address:Port              
LISTEN      0      128       *:6379  #客户端通信端口                                *:*                  
LISTEN      0      128       *:22                                                  *:*                  
LISTEN      0      100       127.0.0.1:25                                          *:*                  
LISTEN      0      128      *:16379 #服务器端通信端口                               *:*                  
LISTEN      0      128        :::22                                                :::*                  
LISTEN      0      100       ::1:25                                                :::*   
```

- 正在一台主机上安装Ruby

```bash
#yum安装的ruby版本太低，需要编译安装
[root@centos src]$wget https://cache.ruby-lang.org/pub/ruby/2.5/ruby-2.5.5.tar.gz
[root@centos src]$yum install zlib-devel openssl-devel -y
[root@centos src]$ls
ruby-2.5.5.tar.gz
[root@centos src]$tar xvf ruby-2.5.5.tar.gz
[root@centos src]$cd ruby-2.5.5
[root@centos src]$./configure
[root@centos src]$make && make install
[root@centos ruby-2.5.5]$gem install redis
Fetching: redis-4.1.2.gem (100%)
Successfully installed redis-4.1.2
Parsing documentation for redis-4.1.2
Installing ri documentation for redis-4.1.2
Done installing documentation for redis after 0 seconds
1 gem installed
[root@centos ~]$ln -s /usr/local/src/redis-4.0.14/src/redis-trib.rb /usr/sbin/
```

- redis-trib.rb的命令选项

```bash
[root@centos ~]$redis-trib.rb
Usage: redis-trib <command> <options> <arguments ...>

  create          host1:port1 ... hostN:portN #创建集群
                  --replicas <arg> #指定master的副本数量
  check           host:port #检查集群信息
  info            host:port #查看集群主机信息
  fix             host:port #修复集群
                  --timeout <arg>
  reshard         host:port #在线热迁移集群指定主机的slots数据
                  --from <arg>
                  --to <arg>
                  --slots <arg>
                  --yes
                  --timeout <arg>
                  --pipeline <arg>
  rebalance       host:port #平衡集群中各主机的slot数量
                  --weight <arg>
                  --auto-weights
                  --use-empty-masters
                  --timeout <arg>
                  --simulate
                  --pipeline <arg>
                  --threshold <arg>
  add-node        new_host:new_port existing_host:existing_port #添加主机到集群
                  --slave
                  --master-id <arg>
  del-node        host:port node_id #删除主机
  set-timeout     host:port milliseconds #设置节点的超时时间
  call            host:port command arg arg .. arg #在集群的所有节点上执行命令
  import          host:port #导入外部redis服务器的数据到当前集群
                  --from <arg>
                  --copy
                  --replace
  help            (show this help)

For check, fix, reshard, del-node, set-timeout you can specify the host and port of any working node in the cluster.
```

- 创建集群

```bash
#修改登录密码（若不修改或者修改后的密码未加引号则会造成创建失败）
[root@centos ~]$vim /usr/local/lib/ruby/gems/2.5.0/gems/redis-4.1.2/lib/redis/client.rb
:password => centos,

#创建cluster集群
[root@centos ~]$redis-trib.rb create --replicas 1 192.168.30.36:6379 192.168.30.46:6379 192.168.30.56:6379 192.168.30.66:6379 192.168.30.76:6379 192.168.30.86:6379
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
192.168.30.36:6379
192.168.30.46:6379
192.168.30.56:6379
Adding replica 192.168.30.76:6379 to 192.168.30.36:6379
Adding replica 192.168.30.86:6379 to 192.168.30.46:6379
Adding replica 192.168.30.66:6379 to 192.168.30.56:6379
M: 1b78ce017f9e4ab67292c47bc471b865a6cc229a 192.168.30.36:6379
   slots:0-5460 (5461 slots) master
M: 61d719e1ee539f8530b0a40e8018b1a934f409b6 192.168.30.46:6379
   slots:5461-10922 (5462 slots) master
M: 61d719e1ee539f8530b0a40e8018b1a934f409b6 192.168.30.56:6379
   slots:10923-16383 (5461 slots) master
S: 61d719e1ee539f8530b0a40e8018b1a934f409b6 192.168.30.66:6379
   replicates 61d719e1ee539f8530b0a40e8018b1a934f409b6
S: 61d719e1ee539f8530b0a40e8018b1a934f409b6 192.168.30.76:6379
   replicates 1b78ce017f9e4ab67292c47bc471b865a6cc229a
S: 61d719e1ee539f8530b0a40e8018b1a934f409b6 192.168.30.86:6379
   replicates 61d719e1ee539f8530b0a40e8018b1a934f409b6
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join.....
>>> Performing Cluster Check (using node 192.168.30.36:6379)
M: e13c423bd000ec875377845da32ec5bea315057e 192.168.30.36:6379
   slots:0-5460 (5461 slots) master #已经分配的槽位
   1 additional replica(s) #分配了一个slave
M: 19a67a5143b61ff447462c829706d855eb504f7d 192.168.30.56:6379
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: 6b10f4b2042c3a994a6f5b10513cce7adc41800d 192.168.30.86:6379
   slots: (0 slots) slave #没有为slave分配槽位
   replicates e357a26c2a3c938862919ce1d9395770099b8ccf
M: e357a26c2a3c938862919ce1d9395770099b8ccf 192.168.30.46:6379
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: 939ca2c6979260801cbf16c4eea1d40df783b028 192.168.30.76:6379
   slots: (0 slots) slave
   replicates e13c423bd000ec875377845da32ec5bea315057e
S: 7326469fb11a322d9fc88961328a10af85c6ab65 192.168.30.66:6379
   slots: (0 slots) slave
   replicates 19a67a5143b61ff447462c829706d855eb504f7d
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

#报错[ERR] Node 192.168.30.46:6379 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0.
解决方法：登录redis-cli执行flushall命令  清空nodes-6379.conf文件后重启服务
```

- 验证主从服务器的状态

```bash
#master的状态
[root@centos redis]$redis-cli 
127.0.0.1:6379> AUTH centos
OK
127.0.0.1:6379> INFO replication
# Replication
role:master
connected_slaves:1
slave0:ip=192.168.30.76,port=6379,state=online,offset=1148,lag=1
master_replid:cd9315f158907aa1a2fba723f149a616c24297fc
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1148
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1148

#slave的状态
[root@centos redis]$redis-cli 
127.0.0.1:6379> AUTH centos
OK
127.0.0.1:6379> INFO replication
# Replication
role:slave
master_host:192.168.30.36
master_port:6379
master_link_status:up
master_last_io_seconds_ago:8
master_sync_in_progress:0
slave_repl_offset:1288
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:cd9315f158907aa1a2fba723f149a616c24297fc
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1288
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1288

#查看集群状态
127.0.0.1:6379> CLUSTER INFO
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
...

#查看主从对应关系
127.0.0.1:6379> CLUSTER NODES
19a67a5143b61ff447462c829706d855eb504f7d 192.168.30.56:6379@16379 master - 0 1560482481000 3 connected 10923-16383
6b10f4b2042c3a994a6f5b10513cce7adc41800d 192.168.30.86:6379@16379 slave e357a26c2a3c938862919ce1d9395770099b8ccf 0 1560482481561 6 connected
e357a26c2a3c938862919ce1d9395770099b8ccf 192.168.30.46:6379@16379 master - 0 1560482483581 2 connected 5461-10922
939ca2c6979260801cbf16c4eea1d40df783b028 192.168.30.76:6379@16379 myself,slave e13c423bd000ec875377845da32ec5bea315057e 0 1560482483000 5 connected
7326469fb11a322d9fc88961328a10af85c6ab65 192.168.30.66:6379@16379 slave 19a67a5143b61ff447462c829706d855eb504f7d 0 1560482481000 4 connected
e13c423bd000ec875377845da32ec5bea315057e 192.168.30.36:6379@16379 master - 0 1560482482571 1 connected 0-5460
```

- 验证集群的写入与主从复制

```bash
#在管理节点测试写入
127.0.0.1:6379> SET key1 value1 #经过算法计算，当前key的槽位需要写入指定的node
(error) MOVED 9189 192.168.30.46:6379 #槽位不再当前node所以无法写入

#切换至有9189的槽位写入
127.0.0.1:6379> set key1 value1 #在指定的node就可以写入
OK
127.0.0.1:6379> keys *
1) "key1"

#在写入主机的slave上查看数据是否同步
127.0.0.1:6379> KEYS *
1) "key1"
127.0.0.1:6379> GET key1
(error) MOVED 9189 192.168.30.46:6379 #slave不提供读写功能，只提供数据备份和选举功能
```

- 集群状态监控

```bash
#查看集群当前状态
[root@centos ~]$redis-trib.rb check 192.168.30.36:6379

#查看集群数据库情况
[root@centos ~]$redis-trib.rb info 192.168.30.36:6379
192.168.30.36:6379 (e13c423b...) -> 0 keys | 5461 slots | 1 slaves.
192.168.30.56:6379 (19a67a51...) -> 0 keys | 5461 slots | 1 slaves.
192.168.30.46:6379 (e357a26c...) -> 1 keys | 5462 slots | 1 slaves.
[OK] 1 keys in 3 masters.
0.00 keys per slot on average.
```

#### - 3.2.3：redis cluster集群节点维护

- 集群维护包括增加 Redis node 节点、减少节点、节点迁移、更换服务器等。
- 增加节点和删除节点会涉及到已有的槽位重新分配及数据迁移

##### - 3.2.3.1：动态添加新的master节点

增加 Redis node 节点，需要与之前的 Redis node 版本相同、配置一致，然后分别启动两台 Redis node，因为一主一从

```bash
#启动一台新主机，在这台主机上同时配置主从
[root@centos ~]$/apps/redis/bin/redis-server /apps/redis/etc/redis.conf 
[root@centos ~]$/apps/redis/bin/redis-server /apps/redis/etc/redis1.conf
[root@centos ~]$ss -nlt
State      Recv-Q Send-Q            Local Address:Port                      Peer Address:Port              
LISTEN     0      128                      *:6379                                 *:*                  
LISTEN     0      128                      *:6380                                 *:*                                                    
LISTEN     0      128                      *:16379                                *:*                  
LISTEN     0      128                      *:16380                                *:*               
...
```

- 添加节点命令

  redis-trib.rb 	add-node  	new_host:new_port 	existing_host:existing_port
  要添加的新redis节点IP和端口添加到的集群中的master IP:端口，加到集群之后默认是master节点但是没有
  slots数据，需要重新分配。

```bash
#添加节点到集群
[root@centos ~]$redis-trib.rb add-node 192.168.30.26:6379 192.168.30.36:6379
...
#查看此时的集群状态
[root@centos ~]$redis-trib.rb check 192.168.30.36:6379
M: 61d719e1ee539f8530b0a40e8018b1a934f409b6 192.168.30.26:6379
   slots: (0 slots) master 
   0 additional replica(s) #新加入集群的主机未分配到槽位和slave节点
#更直观的查看
[root@centos ~]$redis-trib.rb info 192.168.30.36:6379
192.168.30.36:6379 (e13c423b...) -> 0 keys | 5461 slots | 1 slaves.
192.168.30.56:6379 (19a67a51...) -> 0 keys | 5461 slots | 1 slaves.
192.168.30.46:6379 (e357a26c...) -> 1 keys | 5462 slots | 1 slaves.
192.168.30.26:6379 (61d719e1...) -> 0 keys | 0 slots | 0 slaves.
[OK] 1 keys in 4 masters.
0.00 keys per slot on average.
```

- 为新加入的节点分配槽位

添加新主机之后需要对添加至几群的新主机重新分片否则分片为0

```bash
[root@centos ~]$redis-trib.rb reshard 192.168.30.26:6379 #为新节点分配槽位
How many slots do you want to move (from 1 to 16384)? 4096 #分配多少个槽位，一般是平均分配
What is the receiving node ID? 192.168.30.26:
61d719e1ee539f8530b0a40e8018b1a934f409b6 #接收slot的服务器node号
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:all #将哪些源主机的槽位分配给新节点，all时自动在所有master选择划分，如果是从集群中删除主机可以使用此方式将主机上的槽位全部移到其他master主机上
...
Do you want to proceed with the proposed reshard plan (yes/no)? yes #再次确认是否分配

#分配完毕后查看集群状态
[root@centos ~]$redis-trib.rb info 192.168.30.26:6379
192.168.30.26:6379 (61d719e1...) -> 0 keys | 4096 slots | 0 slaves.
192.168.30.46:6379 (e357a26c...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.56:6379 (19a67a51...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.36:6379 (e13c423b...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
```

##### - 3.2.3.2：为新master添加slave节点

- 命令格式

  redis-trib.rb	add-node 	slave_ip:port	 master_ip:port

```bash
[root@centos ~]$redis-trib.rb add-node 192.168.30.26:6380 192.168.30.26:6379
>>> Adding node 192.168.30.26:6380 to cluster 192.168.30.26:6379
>>> Performing Cluster Check (using node 192.168.30.26:6379)
```

- 更改新节点状态为slave

```bash
#需要手动将其指定为某个master的slave，否则其默认角色为master
[root@centos ~]$redis-cli -h 192.168.30.26 -p 6380 #登录到新的slave节点
192.168.30.26:6380> CLUSTER NODES #查看当前集群所有节点，找到目标master节点的ID
...
192.168.30.26:6380> CLUSTER REPLICATE 61d719e1ee539f8530b0a40e8018b1a934f409b6  #将其设置为slave，命令格式为CLUSTER REPLICATE masterID
OK
[root@centos ~]$redis-trib.rb info 192.168.30.26:6379 #查看集群状态是否设置成功
192.168.30.26:6379 (61d719e1...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.46:6379 (e357a26c...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.56:6379 (19a67a51...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.36:6379 (e13c423b...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
```

##### - 3.2.3.3：动态删除集群中的一个节点

删除节点的操作与添加节点的操作正好相反，是先将被删除的 Redis node 上的槽位迁移到集群中的其他 Redis node 节点上，然后再将其删除。如果一个 Redis node 节点上的槽位没有被完全迁移，删除该 node 的时候会提示有数据且无法删除

- 将master的槽位迁移至其他master（被迁移的redis服务器必须保证没有数据）

```bash
[root@centos ~]$redis-trib.rb reshard 192.168.30.36:6379 #登录任一节点分配槽位
How many slots do you want to move (from 1 to 16384)? 4096 #分多少
What is the receiving node ID? e13c423bd000ec875377845da32ec5bea315057e #分给谁
Please enter all the source node IDs. #从哪分，以done结束
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
  Source node #1:61d719e1ee539f8530b0a40e8018b1a934f409b6
Source node #2:done #结束
Do you want to proceed with the proposed reshard plan (yes/no)? yes #再次确认

#查看集群状态
[root@centos ~]$redis-trib.rb info 192.168.30.36:6379
192.168.30.36:6379 (e13c423b...) -> 0 keys | 8192 slots | 2 slaves. #槽位翻番，slave+1
192.168.30.26:6379 (61d719e1...) -> 0 keys | 0 slots | 0 slaves. #已经没有槽位了
192.168.30.56:6379 (19a67a51...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.46:6379 (e357a26c...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.

[root@centos ~]$redis-trib.rb check 192.168.30.36:6379
>>> Performing Cluster Check (using node 192.168.30.36:6379)
M: e13c423bd000ec875377845da32ec5bea315057e 192.168.30.36:6379
   slots:0-6826,10923-12287 (8192 slots) master
   2 additional replica(s) #不仅槽位翻番，而且市区master的slave也附庸在此
M: 61d719e1ee539f8530b0a40e8018b1a934f409b6 192.168.30.26:6379
   slots: (0 slots) master
   0 additional replica(s) #虽然还是master，但是已经没有槽位了
```

- 从集群删除槽位迁移完的服务器

- 命令格式

  redis-trib.rb del-node	 IP:Port 	ID

```bash
[root@centos ~]$redis-trib.rb del-node 192.168.30.26:6379 61d719e1ee539f8530b0a40e8018b1a934f409b6
>>> Removing node 61d719e1ee539f8530b0a40e8018b1a934f409b6 from cluster 192.168.30.26:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.

#验证是否被删除
[root@centos ~]$redis-trib.rb info 192.168.30.36:6379
192.168.30.36:6379 (e13c423b...) -> 0 keys | 8192 slots | 2 slaves.
192.168.30.56:6379 (19a67a51...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.46:6379 (e357a26c...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 3 masters.
0.00 keys per slot on average.
```

- 验证集群Master与Slave 对应关系

若是每台机器上面都有一主一从，则Redis Slave节点一定不能和它的master在同一个服务器，必须为跨主机交叉备份模式，避免主机故障后主备全部挂掉，如果出现Redis Slave与Redis master在同一台Redis node的情况，则需要按照以下步骤重新进行slave分配，直到不相互交叉备份为止

```bash
#登录到slave账号，修改主从对应关系命令语法如下
cluster  replicate masterID
```

- 模拟master宕机

```bash
#Redis Master服务停止之后，其对应的slave会被选举为master继续处理数据的读写操作（将192.168.30.36宕机，检查它的slave192.168.30.76是否变为master）
[root@centos ~]$redis-trib.rb info 192.168.30.76:6379
192.168.30.76:6379 (939ca2c6...) -> 0 keys | 8192 slots | 1 slaves.
192.168.30.56:6379 (19a67a51...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.46:6379 (e357a26c...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 3 masters.
0.00 keys per slot on average.

#宕机后的主机启动后会自动变为此时master的slave
[root@centos ~]$redis-trib.rb info 192.168.30.76:6379
192.168.30.76:6379 (939ca2c6...) -> 0 keys | 8192 slots | 2 slaves.
192.168.30.56:6379 (19a67a51...) -> 0 keys | 4096 slots | 1 slaves.
192.168.30.46:6379 (e357a26c...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 3 masters.
0.00 keys per slot on average.

#验证slave日志，可以看到主从转移的具体过程
[root@centos ~]$tail -f /apps/redis/run/redis_6379.log
```

##### - 3.2.3.4：导入现有的redis数据

导入数据需要redis cluster不能与被导入的数据有重复的key名称，否则导入不成功或中断

- 基础环境准备

导入数据之前需要关闭各redis服务器的密码，包括集群中的各node和源Redis server，避免认证带来的环境不一致从而无法导入，可以加参数--cluster-replace强制替换Redis cluster已有的 key

```bash
[root@redis-s1 ~]# redis-cli -h 192.168.7.102 -p 6379 -a 123456
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
192.168.7.102:6379> CONFIG SET requirepass ""
OK
192.168.7.104:6379> exit
```

- 执行数据导入

将源Redis server的数据直接导入之redis cluster

```bash
#命令格式
[root@s1 ~]#redis-trib.rb import --from 172.18.200.107:6382 --replace 172.18.200.107:6379
```

#### - 3.2.4：其他集群扩展方案

- codis

Codis 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到Codis Proxy和连接原生的Redis Server没有显著区别 (令不支持的命列表), 上层应用可以像使用单机的 Redis 一样使用, Codis底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务

Github 地址：https://github.com/CodisLabs/codis/blob/release3.2/doc/tutorial_zh.md

- twemproxy

Twemproxy 双向代理客户端实现分片，即代替用户将数据分片并到不同的后端服务器进行读写，其还支持 memcached，可以为 proxy 配置算法，缺点为 twemproxy 是瓶颈，不支持数据迁移

官方github 地址 https://github.com/twitter/twemproxy/

## 四、memcached

​	memcache本身没有像 redis 所具备的数据持久化功能，比如RDB和AOF都没有，但是可以通过做集群同步的方式，让各memcache服务器的数据进行同步，从而实现数据的一致性，即保证各memcache的数据是一样的，即使有任何一台 memcache发生故障，只要集群种有一台memcache可用就不会出现数据丢失，当其他memcache重新加入到集群的时候可以自动从有数据的memcache当中自动获取数据并提供服务

​	Memcache 支持最大的内存存储对象为1M，超过1M的数据可以使用客户端压缩或拆分报包放到多个key中，比较大的数据在进行读取的时候需要消耗的时间比较长，memcache最适合保存用户的 session实现 session 共享，Memcached 存储数据时, Memcached会去申请1MB的内存, 把该块内存称为一个slab, 也称为一个page
