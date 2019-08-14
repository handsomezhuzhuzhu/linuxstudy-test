## 一、负载均衡介绍

### - 1.1：什么是负载均衡（Load Blance）

一种服务或基于硬件设备等实现的高可用反向代理技术，负载均衡将特定的业务(web服务、网络流量等)分担给指定的一个或多个后端特定的服务器或设备，从而提高了公司业务的并发处理能力、保证了业务的高可用性、方便了业务后期的水平动态扩展

### - 1.2：为什么使用负载均衡

- Web服务器的动态水平扩展，对用户无感知
- 增加业务并发访问及处理能， 解决单服务器瓶颈问题
- 节约公网IP地址，降低IT支出成本
- 隐藏内部服务器IP，提高内部服务器安全性
- 配置简单，固定格式的配置文件
- 功能丰富，支持四层和七层，支持动态下线主机
- 性能较强，并发数万甚至数十万

### - 1.3：常见的软件负载均衡

- 四层：

  LVS

  HAProxy

  Nginx

- 七层：

  HAProxy

  Nginx

### - 1.4：应用场景

- 四层：Redis、Mysql、RabbitMQ、Memcache
- 七层：Nginx、Tomcat、PHP、图片、动静分离、API等

## 二、HAProxy

### - 2.1：HAProxy介绍

#### - 2.1.1：HAProxy功能

- HAProxy是TCP/HTTP反向代理服务器，尤其适合高可用性高并发性环境

  可以针对http请求添加cookie，进行路由后端服务器

  可平衡负载至后端服务器，并支持持久连接

  支持基于cookie进行调度

  支持所有主服务器故障切换至备用服务器

  支持专用端口实现监控服务

  支持不影响现有连接情况下停止接收新连接请求

  可以在双向添加，修改或删除http报文首部

  支持基于pattern实现连接请求的访问控制

  通过特定的URI为授权的用户提供详细的状态信息

#### - 2.1.2：HAProxy应用

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559563486918.png)

### - 2.2：HAProxy安装

#### - 2.2.1：yum安装

```bash
[root@centos ~]$yum install haproxy -y
[root@centos ~]$haproxy -v 
HA-Proxy version 1.5.18 2016/05/10
Copyright 2000-2016 Willy Tarreau <willy@haproxy.org>
```

#### - 2.2.2：编译安装

官网下载：http://www.haproxy.org/download/1.8/src/haproxy-1.8.20.tar.gz

```bash
#安装前准备
[root@centos src]$ yum install gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools vim iotop bc zip unzip zlib-devel lrzsz tree screen lsof tcpdump wget ntpdate -y
[root@centos src]$cd /usr/local/src/
[root@centos src]$ls
haproxy-1.8.20.tar.gz
#开始安装
[root@centos src]$tar xvf haproxy-1.8.20.tar.gz
[root@centos src]$cd haproxy-1.8.20/
[root@centos haproxy-1.8.20]$make ARCH=x86_64 TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_CPU_AFFINITY=1 PREFIX=/usr/local/haproxy
[root@centos haproxy-1.8.20]$ make install PREFIX=/usr/local/haproxy
install -d "/usr/local/haproxy/sbin"
install haproxy  "/usr/local/haproxy/sbin"
install -d "/usr/local/haproxy/share/man"/man1
install -m 644 doc/haproxy.1 "/usr/local/haproxy/share/man"/man1
install -d "/usr/local/haproxy/doc/haproxy"
for x in configuration management architecture peers-v2.0 cookie-options lua WURFL-device-detection proxy-protocol linux-syn-cookies network-namespaces DeviceAtlas-device-detection 51Degrees-device-detection netscaler-client-ip-insertion-protocol peers close-options SPOE intro; do \
	install -m 644 doc/$x.txt "/usr/local/haproxy/doc/haproxy" ; \
done
[root@centos haproxy-1.8.20]$cp haproxy /usr/sbin/
#创建启动脚本
[root@centos ~]$vim /usr/lib/systemd/system/haproxy.service
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target
[Service]
#支持多配置文件读取，类似于配置文件的include功能
ExecStartPre=/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c -q
ExecStart=/usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID
[Install]
WantedBy=multi-user.target
#创建目录和用户
[root@centos ~]$mkdir /etc/haproxy
[root@centos ~]$useradd -s /sbin/nologin haproxy
[root@centos ~]$mkdir /var/lib/haproxy
[root@centos ~]$chown -R haproxy.haproxy /var/lib/haproxy/
[root@centos ~]$cat /etc/haproxy/haproxy.cfg 
global
	log         127.0.0.1 local2

    chroot      /usr/local/haproxy
    pidfile     /run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
	timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# haproxy.cfg文件中定义了chroot、pidfile、user、group等参数，如果系统没有相应的资源会导致haproxy无法启动，具体参考日志文件/var/log/messages
#启动haproxy
[root@centos ~]$systemctl start haproxy
[root@centos ~]$systemctl enable haproxy
Created symlink from /etc/systemd/system/multi-user.target.wants/haproxy.service to /usr/lib/systemd/system/haproxy.service.
[root@centos ~]$ps -ef | grep haproxy
root       8492      1  0 20:48 ?        00:00:00 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
haproxy    8495   8492  0 20:48 ?        00:00:00 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
root       8606   7171  0 20:53 pts/0    00:00:00 grep --color=auto haproxy
```

#### - 2.2.3：HAProxy组成

- 程序环境：

  主程序：/usr/sbin/haproxy

  配置文件：/etc/haproxy/haproxy.cfg

  Unit file：/usr/lia/systemcd/system/haproxy.service

- 配置段

  - global：全局配置段

    进程及安全配置相关的参数

    性能调整相关参数

    Debug参数

  - proxies：代理配置段

    defaults：为frontend，backend，listen提供默认配置

    frontend：前端，相当于nginx中的server{}

    backend：后端，相当于nginx中的upstream{}

    listen：同时拥有前端和后端配置

##### - 2.2.3.1：global配置

官网参考： https://cbonte.github.io/haproxy-dconv/1.8/configuration.html#3

```bash
chroot #锁定运行目录
deamon #以守护进程运行
#stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin #socket文件
user，group，uid，gid #运行haproxy的用户身份
nbproc #开启haproxy进程数，与cpu保持一致
nbthread #指定每个haproxy进程开启的线程数，默认为每个进程开启一个线程
cpu-map 1 0 #绑定haproxy进程至指定cpu
maxconn #每个haproxy进程最大并发连接数
maxsslconn #每个haproxy进程ssl最大连接数
maxconnrate #每个进程每秒最大连接数
spread-checks #后端server状态check随机提前或延迟百分比时间，建议2-5(20%-50%)之间
pidfile #指定pid文件路径
log 172.0.0.1 local3 info #定义全局的syslog服务器，最多可以定义两个
```

##### - 2.2.3.2：proxies配置

defaults [<name>] #默认配置项，针对以下的frontend、backend和listen生效，可以多个name

frontend <name> #前端servername，类似于nginx的一个虚拟主机server

backend <name> #后端服务器组，等于nginx的upstream

listen <name> #将frontend和backend合并在一起配置

**注：name字段只能使用”-”、”_”、”.”、和”:”，并且严格区分大小写**

##### - 2.2.3.3：defaults配置

```bash
option redispatch #当serverID对应的服务器挂掉后，强制定向到其他健康服务器
option abortonclose #当服务器负载很高的时，自动结束当前队列中处理比较久的连接
option http-keep-alive 60 #开启会话保持
option forwardfor #开启IP透传
mode http #默认工作类型
timeout connect 120s #转发客户端请求到后端server的最长连接时间（TCP之前）
timeout server 600s #转发客户端请求到后端服务器端的超时时长（TCP之后）
timeout client 600s #与客户端的最长空闲时间
timeout http-keep-alive 120s #session会话保持超时时长，范围内会转发到相同的后端服务器
#timeout check 5s #对后端服务器的检测超时时间
```

##### - 2.2.3.4：frontend配置

```bash
bind #指定haproxy的家庭地址，可以使IPV4或IPV6，可以同时监听多个IP和端口，可同时用于listen字段中
bind [<address>]:<port_range> [...] [param*]
mode http/tcp #指定附在协议类型
use_backend backend_name #调用的后端服务器组名称

#示例
frontend web_port
#bind:80,:8080
bind 192.168.30.6:9527,192.168.30.16,9527
use_backend backend_name
```

##### - 2.2.3.5：backend配置

```bash
mode http/tcp #指定负载协议类型
option #配置协议选项
server #定义后端real server
#注意：option后面加httpchk，smtpchk, mysql-check, pgsql-check，ssl-hello-chk方法，可用于实现更多应用层检测功能
```

后端服务器状态监测配置

```bash
check #对指定real进行健康状态检查，默认不开启
addr ip #可指定的健康状态监测ip
port num #指定的健康状态监测端口
inter num #健康状态检查间隔时间，默认2000ms
fall num #后端服务器失效检查次数，默认为3
rise num #后端服务器从下线恢复检查次数，默认为2
weight #默认为1，最大值256,0表示不参与负载均衡
backup #将后端服务器标记为备份状态
diaabled #将后端服务器标记为不可用状态
redirect prefix http://www.magedu.com/ #将请求临时重定向至其它URL，只适用于http模式
maxconn <maxconn> #当前后端server的最大并发连接数
backlog <backlog> #当server的连接数达到上限后的后援队列长度
```

##### - 2.2.3.6：frontend/backend配置案例

```bash
#官网访问入口==================================================
frontend web_port_80
bind 192.168.30.16:80
mode http
use_backend web_port_http_nodes

backend web_port_http_nodes
mode http
option forwardfor
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

##### - 2.2.3.7：listen配置

使用listen替换forntend和backend的配置方式

```bash
#官网业务访问入口===================================================
listen WEB_PORT_80
bind 192.168.30.16:80
mode http
option forwardfor
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

### - 2.3：haproxy调度算法

#### - 2.3.1：静态调度算法

- balance：指明对后端服务器的调度算法，配置在listen或backend
- 静态算法：按照事先定义好的规则轮询公平调度，不关系后端服务器的当前负载，连接数和响应速度等，且无法实时修改权重，只能重启成效



- static-rr：基于权重的轮询制度，不支持权重的运行时调整及后端服务器慢启动，其后端主机数量没有限制
- first：根据服务器在列表中的位置，自上而下进行调度，但是其只会当第一台服务器的连接数达到上限，新请求才会分配给下一台服务器，因此会忽略服务器的权重设置

#### - 2.3.2：动态调度算法

- 动态算法：基于后端服务器状态进行适当调度调整，比如优先调度至当前负载较低的服务器，且权重可以在haproxy运行时动态调整无需重启



- roundrobin：基于权重的轮询动态调度算法，支持权重的动态调整，不同于LVS的rr，支持慢启动即新加的服务器会逐渐增加转发数，每个后端backend最多支持4095个server，此为默认调度算法，server权重设置weight
- leastconn：加权的最少连接数的动态调度，支持权重的动态调度，即当前后端服务器最少的优先调度，比较适合长连接的场景使用，比如mysql等场景

#### - 2.3.3：haproxy调度算法-source

source：源地址hash，基于用户源地址hash并将请求转发到后端服务器，默认为静态即取模方式，但是可以通过hash-type支持的选项更改，后续同一源地址的请求将被转发至同一个后端web服务器，比较适用于session保持/缓存业务场景

- map-based：取模法，基于服务器权重的hash数组取模，该hash是静态的即不支持在线调整权重，不支持慢启动，其对后端服务器调度均衡，缺点是当服务器的总权重发生变化时，即有服务器上线或下线，都会因权重发生变化而导致调度结果整体改变
- consistent：一致性哈希，该hash是动态的，支持在线调整权重，支持慢启动，优点在于当服务器的总权重发生变化时，对调度结果影响是局部的，不会引起大的变动

```bash
listen web_port_80
bind 192.168.30.16:80
mode http
balance source
hash-type consistent
log global #继承default中的日志配置
option forwardfor
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

#### - 2.3.4：haproxy调度算法-uri

- uri：基于对用户请求的uri做hash并将请求转发到后端指定服务器
  - map-based：取模法
  - consistent：一致性哈希

 ```bash
#示例
listen web_port_80
bind 192.168.30.16:80
mode http #不支持tcp，会切换到tcp的roundrobin负载模式
balance uri
hash-type consistent
option forwardfor
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
 ```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559618890555.png)

#### - 2.3.5：haproxy调度算法-url_param

- url_param

  对用户请求中的<params>部分中的参数name做hash运算，并由服务器总权重相除后派发至某调出的服务器；通常用于追踪用户，以确保来自同一个用户的请求失踪发往同一个backend server

  假设url=http://www.magedu.com/foo/bar/index.php?k1=v1&k2=v2

  则：

  host="www.magedu.com"

  url_param="k1=v1&k2=v2"

```bash
#示例
listen web_port_80
bind 192.168.30.16:80
mode http
balance url_param name
hash-type consistent
log global
option forwardfor
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

#### - 2.3.6：haproxy调度算法-hdr

- hdr(<name>)

  针对用户每个http头部（header）请求中的指定信息做hash，此处由<name>指定的http首部将会被取出并做hash运算，然后由服务器总权重相处以后派发至某挑出的服务器没加入无有效的值，则会被轮询调度

  hdr（cookie、user-agent、host）

```bash
#示例
listen web_port_80
bind 192.168.30.16:80
mode http
balance hdr(user-agent) #不同的服务器将被调度到不同的web服务器
hash-type consistent
log global
option forwardfor
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

#### - 2.3.7：haproxy调度算法-rdp-cookie

- rdp-cookie对远程桌面的负载，使用cookie保持会话
- rdp-cookie(<name>)

```bash
listen web_port_80
bind 192.168.30.16:80
mode tcp
balance rdp-cookie
log global
option forwardfor
server rdp0 192.168.30.6:80 check inter 3000 fall 3 rise 5
server rdp1 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

#### - 2.3.8：算法总结

```bash
roundrobin-------->tcp/http 动态
leastconn--------->tcp/http 动态
static-rr--------->tcp/http 静态
first------------->tcp/http 静态
#以下几种取决于hash_type是否consistent
source---------------->tcp/http
Uri------------------->http
url_param------------->http 
hdr------------------->http
rdp-cookie------------>tcp
```

### - 2.4：haproxy的四层与七层

- 四层：
  - 在四层负载设备中，把client发送的报文目标地址（原来的负载均衡的IP地址），根据均衡设备配置的选择web服务器的规则选择对应的web服务器对应的IP地址，这样client就可以直接跟此服务器建立TCP连接并发送数据

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559629994831.png)

- 七层：
  - 七层负载均衡服务器起了一个代理服务器的作用，服务器建立一次TCP连接要三次握手，而client要访问webserver要先与七层负载设备进行三次握手后建立TCP连接，把要访问的报文信息发送给七层负载均衡；然后七层负载均衡再根据设置的均衡规则选择特定的webserver，然后通过三次握手与此台webserver建立TCP连接，然后webserver把需要的数据发送给七层负载均衡设备，负载均衡设备再把数据发送给client；所以，七层负载均衡设备起到了代理服务器的作用

#### - 2.4.1：七层IP透传

- 七层负载：

```bash
#示例1：关闭透传
listen web_port_80
bind 192.168.30.16:80
mode http
#balance rdp-cookie
log global
#option forwardfor #default中的forwardfor也要关闭
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559631315515.png)

```bash
#示例2：开启透传
listen web_port_80
bind 192.168.30.16:80
mode http
#balance rdp-cookie
log global
option forwardfor
server web1 192.168.30.6:80 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 check inter 3000 fall 3 rise 5
```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559631402762.png)

#### - 2.4.2：四层IP透传

- 四层负载：

```bash
#示例1：未开启IP透传
listen web_port_80
bind 192.168.30.16:80
mode tcp
#balance rdp-cookie
log global
#option forwardfor #default中的forwardfor也要关闭
server web1 192.168.30.6:80 send-proxy check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 send-proxy check inter 3000 fall 3 rise 5
```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559632906450.png)

```bash
#示例2：开启IP透传
listen web_port_80
bind 192.168.30.16:80
mode tcp
#balance rdp-cookie
log global
option forwardfor
server web1 192.168.30.6:80 send-proxy check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 send-proxy check inter 3000 fall 3 rise 5
#nginx服务器配置
listen 80 proxy_portocol
```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559632003576.png)

### - 2.5：haproxy自定义配置
#### - 2.5.1：cookie配置

- cookie <value> ：为当前server指定cookie值，实现基于cookie的会话黏性（session会话保持性）

```bash
cookie <name> [ rewrite | insert | prefix ] [ indirect ] [ nocache ] [ postonly ] [preserve ] [ httponly ] [ secure ] [ domain <domain> ]* [ maxidle <idle> ] [maxlife <life> ]
<name>：cookie名称，用于实现持久连接
	rewrite：重写
	insert：插入
	prefix：前缀
	nocache：当client和hapoxy之间有缓存时，不缓存cookie
```

- 基于cookie实现的session保持

```bash
#示例
listen web_port_80
bind 192.168.30.16:80
mode http
cookie SERVER-COOKIE insert indirect nocache
log global
option forwardfor
server web1 192.168.30.6:80 cookie web1 check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80 cookie web2 check inter 3000 fall 3 rise 5
#访问测试
[root@centos ~]$ curl --cookie "SERVER-COOKIE=web1" http://192.168.30.16
this is web1
[root@centos ~]$ curl --cookie "SERVER-COOKIE=web1" http://192.168.30.16
this is web1
[root@centos ~]$ curl --cookie "SERVER-COOKIE=web2" http://192.168.30.16
this is web2
[root@centos ~]$ curl --cookie "SERVER-COOKIE=web2" http://192.168.30.16
this is web2
```
#### - 2.5.2：配置HAProxy状态页

```bash
stats enable #基于默认的参数启用stats page
stats hide-version #隐藏版本
stats refresh <delay> #设定自动刷新时间间隔
stats uri <prefix> #自定义stats page uri，默认值：/haproxy?stats
stats realm <realm> #账户认证时的提示信息，示例：stats realm : HAProxy\ Statistics
stats auth <user>:<passwd> #认证时的账号和密码，可使用多次，默认：no authentication
stats admin {if | unless} <cond> #启用stats page中的管理功能

```

配置示例：

```bash
listen stats
     bind :9009
     stats enable
     stats hide-version
     stats uri /haproxy-status
     stats realm HAProxy\Stats\Page
     stats auth admin:123456
     stats refresh 10s
     stats admin if TRUE
```

访问测试：

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559736324806.png)

#### - 2.5.3：修改报文首部

```bash
#在请求报文尾部添加指定首部
reqadd <string> [{if | unless} <cond>] #支持条件判断
#在响应报文尾部添加指定首部
rspadd <string> [{if | unless} <cond>]
#例： rspadd X-Via:\HAProxy
[root@centos ~]$curl -I http://192.168.30.16
HTTP/1.1 200 OK
Date: Wed, 05 Jun 2019 12:16:53 GMT
Server: Apache/2.4.6 (CentOS)
Last-Modified: Sat, 01 Jun 2019 07:24:06 GMT
ETag: "d-58a3e042f4db0"
Accept-Ranges: bytes
Content-Length: 13
Content-Type: text/html; charset=UTF-8
X-Via:\HAProxy

#从请求报文中删除匹配正则表达式的首部
reqdel <search> [{if | unless} <cond>]
reqidel <search> [{if | unless} <cond>] #不分大小写
#从响应报文中删除匹配正则表达式的首部
rspdel <search> [{if | unless} <cond>]
rspidel <search> [{if | unless} <cond>]
#例：rspidel server.* #从响应报文中删除server信息
[root@centos ~]$curl -I http://192.168.30.16
HTTP/1.1 200 OK
Date: Wed, 05 Jun 2019 12:24:59 GMT
Content-Type: text/html
Content-Length: 13
Last-Modified: Tue, 04 Jun 2019 07:31:47 GMT
ETag: "5cf61e63-d"
Accept-Ranges: bytes
X-Via:\HAProxy

#例：rspidel X-Powered-By:.* #从响应报文删除X-Powered-By信息
```

#### - 2.5.4：HAProxy日志配置

```bash
#HAProxy配置
log 127.0.0.1 local2 info #基于syslog记录日志到指定设备，local{1-7}级别有（err、warning、info、debug）
listen web_port_80
bind 192.168.30.16:80
mode http
log global
option forwardfor
server web1 192.168.30.6:80  check inter 3000 fall 3 rise 5
server web2 192.168.30.26:80  check inter 3000 fall 3 rise 5
#rsyslog配置
$ModLoad imudp
$UDPServerRun 514
local2.*                          /var/log/haproxy.log
[root@centos ~]$systemctl restart haproxy.service 
[root@centos ~]$systemctl restart rsyslog.service 
#测试
[root@centos ~]$tail /var/log/haproxy.log 
Jun  5 20:30:34 localhost haproxy[13197]: 192.168.30.1:7704 [05/Jun/2019:20:30:34.494] web_port_80 web_port_80/web2 0/0/1/0/1 200 272 - - ---- 2/2/0/0/0 0/0 "GET / HTTP/1.1"
...
```

- 自定义记录日志
  - 将特定信息记录在日志中

```bash
capture cookie <name> len <length> #捕获请求和响应报文中的cookie并记录日志
capture request header <name> len <length> #捕获请求请求报文中指定的首部内容和长度并记录日志
capture respone header <name> len <length> #捕获响应报文响应报文中指定的内容和长度首部并记录日志
#示例
capture request header Host len 256
#访问测试
[root@centos ~]$tail /var/log/haproxy.log 
Jun  5 20:43:41 localhost haproxy[13467]: 192.168.30.1:8506 [05/Jun/2019:20:43:41.521] web_port_80 web_port_80/web2 0/0/0/1/1 200 272 - - ---- 2/2/0/0/0 0/0 {192.168.30.16} "GET / HTTP/1.1"
capture request header User-Agent len 512
#访问测试
[root@centos ~]$tail /var/log/haproxy.log
Jun  5 20:45:49 localhost haproxy[13504]: 192.168.30.1:8639 [05/Jun/2019:20:45:49.266] web_port_80 web_port_80/web1 0/0/0/1/1 200 243 - - ---- 2/2/0/0/0 0/0 {192.168.30.16|Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36} "GET / HTTP/1.1"
```

#### - 2.5.5：压缩功能

- compression algo #启用http协议中的压缩机制，常用算法有gzip、deflate
- compression type #要压缩的类型

```bash
#示例1
compression algo gzip
compression type  text/plain text/html text/css text/xml text/javascript application/javascript
```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559739315726.png)

#### - 2.5.6：web服务器状态监测

三种状态监测方式

- 基于四层的传输端口做状态监测
  - server 192.168.30.6 192.168.30.6:80  check inter 3000 fall 3 rise 5
- 基于指定URI做状态监测
- 基于指定URI的request请求头部内容做状态监测

```bash
option httpchk
option httpchk <uri>
option httpchk <method> <uri>
option httpchk <method> <uri> <version>

#示例
option httpchk GET /wp- - includes/js/jquery/jquery.js?ver=1.12.4 HTTP/1.0 # 基于指定 URL
#option httpchk HEAD /wp- - includes/js/jquery/jquery.js?ver=1.12.4 HTTP/1.0\ \r r\ \ nHost:\ \  192.168.7.102 # 通过 request获取的头部信息进行匹配进行健康检测
```

### - 2.6：ACL

#### - 2.6.1：acl简介

- acl：对接收到的报文进行匹配和过滤，基于请求报文头部中的源地址、源端口、目标端口、请求方法、URL、文件后缀等信息内容进行匹配并执行进一步操作

```bash
#用法
acl <aclname> <criterion> [flags]   [operator]  [<value>]\
acl   名称         条件    条件标记位   具体操作符   操作对象类型
```

- ACL名称，可以使用大写字母A-Z、小写字母a-z、冒号：、点.、中横线和下划线，并且严格区分大小写，例如Image_site和image_site就完全是两个acl

```bash
#用法列举
hdr ([<name>[,<occ>]])：完全匹配字符串
hdr_beg ([<name>[,<occ>]])：前缀匹配
hdr_dir（[<name> [，<occ>]]）：路径匹配
hdr_dom（[<name> [，<occ>]]）：域匹配
hdr_end（[<name> [，<occ>]]）：后缀匹配
hdr_len（[<name> [，<occ>]]）：长度匹配
hdr_reg（[<name> [，<occ>]]）：正则表达式匹配
hdr_sub（[<name> [，<occ>]]）：子串匹配
```

- 匹配条件

```bash
<criterion>：匹配条件
dst		   ：目标IP
dst_port   ：目标PORT
src        ：源IP
src_port   ：源PORT
```

- hdr <string>用于测试请求头部首部指定内容

  hdr_dom(host) 请求的host名称，如 www.magedu.com
  hdr_beg(host) 请求的host开头，如 www. img. video. download. ftp.
  hdr_end(host) 请求的host结尾，如 .com .net .cn
  path_beg 请求的URL开头，如/static、/images、/img、/css
  path_end 请求的URL中资源的结尾，如 .gif .png .css .js .jpg .jpeg

- flags

```bash
<flags> 条件标记
-i 不区分大小写
-m 使用指定的pattern匹配方法
-n 不做DNS解析
-u 禁止acl重名，否则多个同名acl匹配或关系
```

- operator

```bash
[operator] 操作符
正数比较：eq、ge、gt、le、lt
字符比较：
- exact match (-m str) :字符串必须完全匹配模式
- substring match (-m sub) :在提取的字符串中查找模式，如果其中任何一个被发现，ACL将匹配
- prefix match (-m beg) :在提取的字符串首部中查找模式，如果其中任何一个被发现，ACL将匹配
- suffix match (-m end) :将模式与提取字符串的尾部进行比较，如果其中任何一个匹配，则ACL进行匹配
- subdir match (-m dir) :查看提取出来的用斜线分隔（“/”）的字符串，如果其中任何一个匹配，则ACL进行匹配
- domain match (-m dom) :查找提取的用点（“.”）分隔字符串，如果其中任何一个匹配，则ACL进行匹配
```

- value

```bash
<value>的类型：
- Boolean #布尔值false，true
- integer or integer range #整数或整数范围，比如用于匹配端口范围，1024～32768
- IP address / network #IP地址或IP范围, 192.168.0.1 ,192.168.0.1/24
- string 
	exact –精确比较
	substring—子串 
	suffix-后缀比较
	prefix-前缀比较
	subdir-路径， /wp-includes/js/jquery/jquery.js
	domain-域名，www.magedu.com
- regular expression #正则表达式
- hex block #16进制
```

#### - 2.6.2：acl定义与调用

- 多个acl作为条件时的逻辑关系
  - 与：隐式（默认）使用
  - 或：使用"or"或"||"表示
  - 否定：使用"!"表示

```bash
#示例：
if valid_src valid_port  #与关系
if invalid_src || invalid_port #或
if !invalid_src			 #非
```

#### - 2.6.3：acl示例-域名匹配

```bash
#示例
listen web_port
    bind 192.168.30.16:80
    mode http
    log global
    acl test_host hdr_dom(host) www.linux.com #匹配域名www.linux.com
    use_backend test_host if test_host #匹配到www.linux.com时使用test_host backend
    default_backend default_web #以上都没有匹配到时使用默认backend

backend test_host
	mode http
	server 192.168.30.6 192.168.30.6:80 check inter 2000 fall 3 rise 5

backend default_web
	mode http
	server 192.168.30.26 192.168.30.26:80 check inter 2000 fall 3 rise 5
```

#### - 2.6.4：acl示例-源地址访问控制

```bash
#示例
listen web_port
    bind 192.168.30.16:80
    mode http
    log global
    acl ip_range src 192.168.30.1 #匹配源地址为192.168.30.1的
    use_backend ip_range if ip_range
    default_backend default_web

backend ip_range
	mode http
	server 192.168.30.6 192.168.30.6:80 check inter 2000 fall 3 rise 5

backend default_web
	mode http
	server 192.168.30.26 192.168.30.26:80 check inter 2000 fall 3 rise 5
```

#### - 2.6.5：acl示例-匹配浏览器

```bash
#示例
listen web_port
    bind 192.168.30.16:80
    mode http
    log global
    acl block_test src 192.168.30.6 192.168.30.26
    acl redirect_test hdr(User-Agent) -m sub "Chrome" #匹配User-Agent中包含Chrome字符的
    use_backend redirect_test if redirect_test
    block if block_test #匹配到block_test就丢弃忽略
    default_backend default_web

backend redirect_test
	mode http
	server 192.168.30.6 192.168.30.6:80 check inter 2000 fall 3 rise 5

backend default_web
	mode http
	server 192.168.30.26 192.168.30.26:80 check inter 2000 fall 3 rise 5
```

#### - 2.6.6：acl示例-访问重定向

要求：公司域名变更，用户访问原来的域名可以无缝跳转至新域名

```bash
#示例
listen web_port
        bind 192.168.30.16:80
        mode http
        log global
        acl www_web_page hdr_dom(host) -i www.linux.com #匹配域名不区分大小写
        redirect prefix http://www.baidu.com if www_web_page #匹配到域名后重定向至www.baidu.com
        default_backend default_web

backend default_web
mode http
server 192.168.30.26 192.168.30.26:80 check inter 2000 fall 3 rise 5
```

#### - 2.6.7：自定义错误页面和跳转

```bash
#示例（写在defaults中）
 errorfile 500 /usr/local/haproxy/html/500.html
 errorfile 502 /usr/local/haproxy/html/502.html
 errorfile 503 http:/192.168.30.6/error_page/503.html #将错误页面定义到其他主机
 #自定义错误跳转示例
 errorloc 503 http://192.168.30.6/error_page/503.html
```

#### - 2.6.8：基于acl+文件后缀实现动静分离

要求：当用户访问时，将php请求与image请求分别调度到两台专门的服务器，实现访问的动静分离

```bash
#haproxy配置示例
listen web_port
    bind 192.168.30.16:80
    mode http
    log global
    acl php_server path_end -i .php
    use_backend php_server if php_server
    acl image_server path_end -i .jpg .png .jpeg .gif
    use_backend image_server if image_server
    default_backend default_web

backend php_server
	mode http
	server 192.168.30.6 192.168.30.6:80 check inter 2000 fall 3 rise 5

backend image_server
	mode http
	server 192.168.30.26 192.168.30.26:80 check inter 2000 fall 3 rise 5

backend default_web
	mode http
	server web2 192.168.30.26:8080 check inter 2000 fall 3 rise 5
#nginx配置示例
 [root@centos ~]$yum install php-fpm -y
 [root@centos ~]$vim /etc/nginx/nginx.conf
 location ~ \.php$ {
      root /usr/share/nginx/html;
      fastcgi_pass 127.0.0.1:9000;
      fastcgi_index index.php;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      include fastcgi_params;
 }
 [root@centos ~]$vim /usr/share/nginx/html/index.php #访问测试页面
<?php
phpinfo();
?> 
```

访问测试：

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559872052301.png)

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/1559872071769.png)

#### - 2.6.9：acl示例-http基于策略的访问控制

实现对满足特定条件的用户访问控制，比如允许与拒绝

```bash
#示例
listen web_port
    bind 192.168.30.16:80
    mode http
    log global
    acl bad_guy src 192.168.30.6
    http-request deny if bad_guy

#访问测试
[root@centos ~]$curl http://192.168.30.16
<html><body><h1>403 Forbidden</h1>
Request forbidden by administrative rules.
</body></html>
```

#### - 2.6.10：系统预定义acl

### - 2.7：四层访问控制

tcp-request connection {accept|reject} [{if | unless} <condition>] 根据第4层条件对传入连接执行操作

```bash
#示例
listen web_port
    bind 192.168.30.16:80
    mode tcp
    log global
    acl invalid_src src 192.168.30.6
    tcp-request connection reject if invalid_src
#访问测试
[root@centos ~]$curl http://192.168.30.16
curl: (56) Recv failure: Connection reset by peer
[root@centos ~]$curl http://192.168.30.16
curl: (56) Recv failure: Connection reset by peer
```

### 2.8：haproxy-https协议

- 配置haproxy支持https协议：

- 支持ssl会话

  - bind *:443 ssl crt /PATH/TO/SOME_PEM_FILE

  crt 后证书文件为PEM格式，且同时包含证书和所有私钥

  cat  demo.crt demo.key > demo.pem

- 把80端口的请求重定向到443

  bind *:80

  redirect scheme http if !{ ssl_fc }

- 向后端传递用户请求的协议和端口frontend或backend）
  http_request set-header X-Forwarded-Port %[dst_port]
  http_request add-header X-Forwared-Proto https if { ssl_fc }

#### - 2.8.1：https-证书制作

```bash
mkdir /usr/local/haproxy/certs
cd /usr/local/haproxy/certs
openssl genrsa -out haproxy.key 2048
openssl req -new -x509 -key haproxy.key -out haproxy.crt -subj "/CN=www.magedu.net"
cat haproxy.key haproxy.crt > haproxy.pem
openssl x509 -in haproxy.pem -noout -text #查看证书
```

#### - 2.8.2：配置实例

```bash
#web server http
• frontend web_server-http
• bind 172.18.200.101:80
• redirect scheme https if !{ ssl_fc }
• mode http
• use_backend web_host

• #web server https
• frontend web_server-https
• bind 172.18.200.101:443 ssl crt /usr/local/haproxy/certs/haproxy.pem
• mode http
• use_backend web_host

 backend default_host
• mode http
• server web1 192.168.7.101:8080 check inter 2000 fall 3 rise 5
• server web2 192.168.7.102:8080 check inter 2000 fall 3 rise 5

• backend web_host
• mode http
• http-request set-header X-Forwarded-Port %[dst_port]
• http-request add-header X-Forwarded-Proto https if { ssl_fc }
• server web1 192.168.7.101:8080 check inter 2000 fall 3 rise 5
• server web2 192.168.7.102:8080 check inter 2000 fall 3 rise 5
```

### - 2.9：haproxy-服务器动态上下线

安装socat命令，socat是和socket通信的命令

```bash
#haproxy配置
[root@centos ~]$vim /etc/haproxy/haproxy.cfg
stats socket /var/lib/haproxy/stats mode 600 level admin #主要是权限设置，不然会显示权限拒绝
#安装
[root@centos ~]$yum install socat -y
[root@centos ~]$echo "help" | socat stdio /var/lib/haproxy/stats #查看帮助
[root@centos ~]$echo "show info" | socat stdio /var/lib/haproxy/stats #查看状态信息
[root@centos ~]$echo "get weight web_port_80/web1" | socat stdio /var/lib/haproxy/stats #查看某个服务的权重
[root@centos ~]$echo "disable server web_port_80/web1" | socat stdio /var/lib/haproxy/stats #关闭web1服务器，注意格式
[root@centos ~]$echo "enable server web_port_80/web1" | socat stdio /var/lib/haproxy/stats #开启web1服务器
```

多个socket文件监控同一服务的多个pid进程，若是有多个子进程，socat命令一次只能关闭一个进程的服务，另一个进程的服务依然存在，所以多进程必须创建多个socket文件来监控同一服务的不同进程

```bash
#示例
[root@centos ~]$echo "disable server php_server/192.168.30.6" | socat stdio /var/lib/haproxy/stats1
[root@centos ~]$echo "disable server php_server/192.168.30.6" | socat stdio /var/lib/haproxy/stats2
#必须两个全部下线才可以，不然在状态栏中还会显示有服务存在
```
