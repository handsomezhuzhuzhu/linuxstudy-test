## 一、openvpn简介及部署

VPN：虚拟私人网络，又称虚拟专有网络，用于在不安全的线路上安全的传输数据

OpenVPN 是一个强大的、高度灵活的 VPN 守护进程。它支持 SSL/TLS 安全、Ethernet bridging、经由代理的 TCP 或 UDP 隧道和 NAT。另外，它也支持动态 IP 地址以及DHCP，可伸缩性足以支持数百或数千用户的使用场景，同时可移植至大多数主流操作系统平台上

官网：https://openvpn.net

GitHub地址：https://github.com/OpenVPN/openvpn

### - 1.1：部署架构

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561859139194.png)

### - 1.2：安装openvpn

```bash
[root@centos ~]$yum install openvpn easy-rsa -y
[root@centos ~]$rpm -ql easy-rsa
[root@centos ~]$rpm -ql openvpn

#copy配置文件：
[root@centos ~]$cp /usr/share/doc/openvpn-2.4.7/sample/sample-config-files/server.conf /etc/openvpn/
[root@centos ~]$cp -r /usr/share/easy-rsa/ /etc/openvpn/
[root@centos ~]$cp /usr/share/doc/easy-rsa-3.0.3/vars.example /etc/openvpn/easy-rsa/3.0.3/vars
[root@centos ~]$cd /etc/openvpn/easy-rsa/3.0.3/
[root@centos 3.0.3]$tree #当前目录结构
.
├── easyrsa
├── openssl-1.0.cnf
├── vars
└── x509-types
    ├── ca
    ├── client
    ├── COMMON
    ├── san
    └── server

1 directory, 8 files
```

### - 1.3：创建PKI和CA签发机构

```bash
[root@centos 3.0.3]$pwd
/etc/openvpn/easy-rsa/3.0.3
[root@centos 3.0.3]$./easyrsa init-pki

Note: using Easy-RSA configuration from: ./vars

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/easy-rsa/3.0.3/pki

[root@centos 3.0.3]$ll pki/
total 0
drwx------ 2 root root 6 Jun 28 20:00 private
drwx------ 2 root root 6 Jun 28 20:00 reqs
```

### - 1.4：创建CA机构

```bash
[root@centos 3.0.3]$./easyrsa build-ca nopass

Note: using Easy-RSA configuration from: ./vars
Generating a 2048 bit RSA private key
...........................+++
........+++
writing new private key to '/etc/openvpn/easy-rsa/3.0.3/pki/private/ca.key.xNCTCvRZMQ' #CA的私钥
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]: #此处直接回车即可

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/easy-rsa/3.0.3/pki/ca.crt

#验证CA的私钥
[root@centos 3.0.3]$ll /etc/openvpn/easy-rsa/3.0.3/pki/private/ca.key 
-rw------- 1 root root 1708 Jun 28 20:04 /etc/openvpn/easy-rsa/3.0.3/pki/private/ca.key
```

### - 1.5：创建服务端证书（私钥）

```bash
[root@centos 3.0.3]$./easyrsa gen-req server nopass

Note: using Easy-RSA configuration from: ./vars
Generating a 2048 bit RSA private key
........................................................................................+++
..........................................................................................+++
writing new private key to '/etc/openvpn/easy-rsa/3.0.3/pki/private/server.key.Xb7yq52RXA'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [server]:

Keypair and certificate request completed. Your files are:
req: /etc/openvpn/easy-rsa/3.0.3/pki/reqs/server.req
key: /etc/openvpn/easy-rsa/3.0.3/pki/private/server.key

#验证CA证书
[root@centos 3.0.3]$ll ./pki/private/
total 8
-rw------- 1 root root 1708 Jun 28 20:04 ca.key
-rw------- 1 root root 1704 Jun 28 20:07 server.key
```

### - 1.6：签发服务端证书：

```bash
#生成服务端crt公钥：
[root@centos 3.0.3]$./easyrsa sign server server

Note: using Easy-RSA configuration from: ./vars


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 3650 days:

subject=
    commonName                = server


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'server'
Certificate is to be certified until Jun 25 12:11:05 2029 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt

#验证生成的服务端公钥：
[root@centos 3.0.3]$ll /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt 
-rw------- 1 root root 4552 Jun 28 20:11 /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt
```

### - 1.7：创建Diffie-Hellman

https://www.cnblogs.com/hyddd/p/7689132.html

密钥交换方法，它是一种安全协议，让双方在完全没有对方任何预先信息的条件下通过不安全信道建立起一个密钥，这个
密钥一般作为“对称加密”的密钥而被双方在后续数据传输中使用。其应用非常广泛，在SSH、VPN、Https...都有应用，勘称现代密码基石

```bash
[root@centos 3.0.3]$./easyrsa gen-dh

Note: using Easy-RSA configuration from: ./vars
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
...
DH parameters of size 2048 created at /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem

#验证生成的密钥文件：
[root@centos 3.0.3]$ll /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem 
-rw------- 1 root root 424 Jun 28 20:18 /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem
```

至此服务端配置完成，下面是配置客户端

### - 1.8：创建客户端证书

```bash
[root@centos 3.0.3]$cp -r /usr/share/easy-rsa/ /etc/openvpn/client/easy-rsa
[root@centos 3.0.3]$cp /usr/share/doc/easy-rsa-3.0.3/vars.example /etc/openvpn/client/easy-rsa/vars

#生成pki目录
[root@centos 3.0.3]$cd /etc/openvpn/client/easy-rsa/3.0.3
[root@centos 3.0.3]$pwd
/etc/openvpn/client/easy-rsa/3.0.3
[root@centos 3.0.3]$./easyrsa init-pki

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/client/easy-rsa/3.0.3/pki

#验证pki目录
[root@centos 3.0.3]$ll ./pki
total 0
drwx------ 2 root root 6 Jun 28 20:28 private
drwx------ 2 root root 6 Jun 28 20:28 reqs
[root@centos 3.0.3]$ll ./pki/private/
total 0
[root@centos 3.0.3]$ll ./pki/reqs/
total 0

#生成客户端证书
[root@centos 3.0.3]$./easyrsa gen-req handsomejack nopass #客户证书为handsomejack，没有设置密码
Generating a 2048 bit RSA private key
.......................................................................+++
.............+++
writing new private key to '/etc/openvpn/client/easy-rsa/3.0.3/pki/private/handsomejack.key.XQ6syCUMFb'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [handsomejack]:

Keypair and certificate request completed. Your files are:
req: /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/handsomejack.req
key: /etc/openvpn/client/easy-rsa/3.0.3/pki/private/handsomejack.key

[root@centos 3.0.3]$tree /etc/openvpn/client/easy-rsa/3.0.3/pki/
/etc/openvpn/client/easy-rsa/3.0.3/pki/
├── private
│   └── handsomejack.key
└── reqs
    └── handsomejack.req

2 directories, 2 files
```

### - 1.9：签发客户端证书

```bash
[root@centos 3.0.3]$cd /etc/openvpn/easy-rsa/3.0.3/
[root@centos 3.0.3]$pwd
/etc/openvpn/easy-rsa/3.0.3

#导入req文件
[root@centos 3.0.3]$./easyrsa import-req /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/handsomejack.req  handsomejack

Note: using Easy-RSA configuration from: ./vars

The request has been successfully imported with a short name of: handsomejack
You may now use this name to perform signing operations on this request.

#签发客户端证书
[root@centos 3.0.3]$./easyrsa sign client handsomejack

Note: using Easy-RSA configuration from: ./vars


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 3650 days:

subject=
    commonName                = handsomejack


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'handsomejack'
Certificate is to be certified until Jun 25 12:38:51 2029 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/easy-rsa/3.0.3/pki/issued/handsomejack.crt

#验证签发后的crt证书
[root@centos 3.0.3]$ll /etc/openvpn/easy-rsa/3.0.3/pki/issued/handsomejack.crt 
-rw------- 1 root root 4446 Jun 28 20:38 /etc/openvpn/easy-rsa/3.0.3/pki/issued/handsomejack.crt
```

### - 1.10：复制证书到server目录

```bash
[root@centos 3.0.3]$mkdir /etc/openvpn/certs
[root@centos 3.0.3]$cd /etc/openvpn/certs/
[root@centos certs]$cp /etc/openvpn/easy-rsa/3.0.3/pki/dh.pem .
[root@centos certs]$cp /etc/openvpn/easy-rsa/3.0.3/pki/ca.crt .
[root@centos certs]$cp /etc/openvpn/easy-rsa/3.0.3/pki/issued/server.crt .
[root@centos certs]$cp /etc/openvpn/easy-rsa/3.0.3/pki/private/server.key .

[root@centos certs]$tree .
.
├── ca.crt
├── dh.pem
├── server.crt
└── server.key

0 directories, 4 files

```

### - 1.11：客户端公钥与私钥

```bash
[root@centos certs]$mkdir /etc/openvpn/client/handsomejack                                         
[root@centos certs]$cp /etc/openvpn/easy-rsa/3.0.3/pki/ca.crt /etc/openvpn/client/handsomejack/
ient/handsomejacks]$cp /etc/openvpn/easy-rsa/3.0.3/pki/issued/handsomejack.crt /etc/openvpn/client/handsomejack/
[root@centos certs]$cp /etc/openvpn/client/easy-rsa/3.0.3/pki/private/handsomejack.key /etc/openvpn/client/handsomejack/
[root@centos certs]$tree /etc/openvpn/client/handsomejack/
/etc/openvpn/client/handsomejack/
├── ca.crt
├── handsomejack.crt
└── handsomejack.key

0 directories, 3 files

```

### - 1.12：server端配置文件

```bash
#配置文件解读
[root@centos certs]$vim /etc/openvpn/server.conf
local 172.18.133.222 #本机监听IP
port 1194 #端口
# TCP or UDP server?
proto tcp #协议，指定OpenVPN创建的通信隧道类型
#proto udp
#dev tap：创建一个以太网隧道，以太网使用tap
dev tun：创建一个路由IP隧道，互联网使用tun
一个TUN设备大多时候，被用于基于IP协议的通讯。一个TAP设备允许完整的以太网帧通过Openvpn隧道，因此提
供非ip协议的支持，比如IPX协议和AppleTalk协议
A TUN device is used mostly for VPN tunnels where only IP-traffic is used. A TAP
device allows full Ethernet frames to be passed over the OpenVPN tunnel. hence
providing support for non-ip based protocols such as IPX and AppleTalk.
#dev-node MyTap #TAP-Win32适配器。非windows不需要
#topology subnet #网络拓扑，不需要配置
server 10.8.0.0 255.255.255.0 #客户端连接后分配IP的地址池，服务器默认会占用第一个IP 10.8.0.1
#ifconfig-pool-persist ipp.txt #为客户端分配固定IP，不需要配置
#server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100 #配置网桥模式，不需要
push " route 10.20.30.0 255.255.255.0" #给客户端生成的静态路由表，下一跳为openvpn服务器的10.8.0.1
push " route 172.20.0.0 255.255.0.0"
;client-config-dir ccd #为指定的客户端添加路由，改路由通常是客户端后面的内网网段而不是服务端的，也不需要设置
;route 192.168.40.128 255.255.255.248
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
;learn-address ./script #运行外部脚本，创建不同组的iptables 规则，不配置
;push "redirect-gateway def1 bypass-dhcp" #启用后，客户端所有流量都将通过VPN服务器，因此不需要配置
#;push "dhcp-option DNS 208.67.222.222" #推送DNS服务器，不需要配置
#;push "dhcp-option DNS 208.67.220.220"
client-to-client #运行不同的client直接通信
;duplicate-cn #多个用户共用一个账户，一般用于测试环境，生产环境都是一个用户一个证书
keepalive 10 120 #设置服务端检测的间隔和超时时间，默认为每 10 秒 ping一次，如果 120 秒没有回应则认为对方已经 down
#tls-auth /etc/openvpn/server/ta.key 0 #可使用以下命令来生成：openvpn –genkey –secret
ta.key #服务器和每个客户端都需要拥有该密钥的一个拷贝。第二个参数在服务器端应该为’0’，在客户端应该为’1’
cipher AES-256-CBC #加密算法
;compress lz4-v2 #启用压缩
;push "compress lz4-v2"
;comp-lzo #旧户端兼容的压缩配置，需要客户端配置开启压缩
;max-clients 100 #最大客户端数
;user nobody #运行openvpn服务的用户和组
;group nobody
persist-key #重启VPN服务，你重新读取keys文件，保留使用第一次的keys文件
persist-tun #重启vpn服务，一直保持tun或者tap设备是up的，否则会先down然后再up
status openvpn-status.log #openVPN状态记录文件，每分钟会记录一次
#;log     openvpn.log #日志记录方式和路径，log会在openvpn启动的时候清空日志文件
log-append /var/log/openvpn/openvpn.log #重启openvpn后在之前的日志后面追加新的日志
verb 3 #设置日志级别，0-9，级别越高记录的内容越详细，
mute 20 #相同类别的信息只有前20条会输出到日志文件中
explicit-exit-notify 1 #通知客户端，在服务端重启后可以自动重新连接，仅能用于udp模式，tcp模式不需要配置即可实现断开重连接

#最终配置：
[root@centos openvpn]$grep "^[a-Z]" /etc/openvpn/server.conf
local 172.20.133.222
port 1194
proto tcp
dev tun
ca /etc/openvpn/certs/ca.crt
cert /etc/openvpn/certs/server.crt
key /etc/openvpn/certs/server.key  # This file should be kept secret
dh /etc/openvpn/certs/dh.pem
server 10.8.0.0 255.255.255.0
push "route 10.20.30.0 255.255.255.0"
push "route 172.20.0.0 255.255.0.0"
client-to-client
keepalive 10 120
cipher AES-256-CBC
max-clients 100
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
log-append  /var/log/openvpn/openvpn.log
verb 9
mute 20

[root@centos certs]$mkdir /var/log/openvpn
[root@centos certs]$chown -R nobody.nobody /var/log/openvpn/

```

### - 1.13：客户端配置文件

```bash
[root@centos certs]$cd /etc/openvpn/client/handsomejack/
[root@centos handsomejack]$ grep -Ev "^(#|$|;)" /usr/share/doc/openvpn-2.4.7/sample/sample-config-files/client.conf >/etc/openvpn/client/handsomejack/client.ovpn #windows客户端后缀为.ovpn

#客户端最终配置：
[root@centos handsomejack]$vim /etc/openvpn/client/handsomejack/client.ovpn
client #声明自己是客户端
dev tun #接口类型，必须和服务器一致
proto tcp #使用的协议，必须和服务端保持一致
remote 172.20.133.222 1194 #server端的ip和端口，可以写域名但是需要在本地配置解析
resolv-retry infinite #如果是写的server端的域名，那么就始终解析，如果域名发生变化，会重新连接到新的域名对应的IP
nobind #本机不绑定监听端口，客户端是随机打开端口连接到服务端的1194
persist-key
persist-tun
ca ca.crt
cert handsomejack.crt
key handsomejack.key
remote-cert-tls server #指定采用服务器校验方式
#tls-auth ta.key 1
cipher AES-256-CBC
verb 3

#验证当前目录
[root@centos handsomejack]$tree /etc/openvpn/client/handsomejack/
/etc/openvpn/client/handsomejack/
├── ca.crt
├── client.ovpn
├── handsomejack.crt
└── handsomejack.key

0 directories, 4 files

```

### - 1.14：启动openvpn服务

```bash
[root@centos handsomejack]$getenforce 
Disabled
[root@centos handsomejack]$systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@centos handsomejack]$yum install iptables-services iptables -y
[root@centos handsomejack]$systemctl enable iptables.service
Created symlink from /etc/systemd/system/basic.target.wants/iptables.service to /usr/lib/systemd/system/iptables.service.
[root@centos handsomejack]$systemctl start iptables.service

#清空已有规则
[root@centos ~]$iptables -F
[root@centos ~]$iptables -X
[root@centos ~]$iptables -Z
[root@centos ~]$iptables -t nat -F
[root@centos ~]$iptables -t nat -X
[root@centos ~]$iptables -t nat -Z

#开启路由转发功能
[root@centos ~]$sysctl -p
net.ipv4.ip_forward = 1

#创建iptables规则
[root@centos ~]$iptables -t nat -A POSTROUTING -s 10.8.0.0/16 -j MASQUERADE
[root@centos ~]$iptables -A INPUT -p TCP --dport 1194 -j ACCEPT
[root@centos ~]$iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
[root@centos ~]$service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]

#验证防火墙规则
[root@centos ~]$iptables -nvL
Chain INPUT (policy ACCEPT 286 packets, 22738 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:1194
  122  8664 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 96 packets, 6440 bytes)
 pkts bytes target     prot opt in     out     source               destination
 
 [root@centos ~]$iptables -t nat -nvL
Chain PREROUTING (policy ACCEPT 25 packets, 2644 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 22 packets, 1747 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      *       10.8.0.0/16          0.0.0.0/0  
    
#创建日志目录并授权
[root@centos ~]$ll /var/log/openvpn/ -d
drwxr-xr-x 2 nobody nobody 6 Jun 28 21:18 /var/log/openvpn/

#启动openvpn服务
[root@centos ~]$systemctl start openvpn@server
[root@centos ~]$systemctl enable openvpn@server

#验证日志
[root@centos openvpn]$tail /var/log/openvpn/openvpn.log 
Fri Jun 28 22:18:36 2019 us=581582 MULTI TCP: multi_tcp_action a=TA_TUN_READ p=0
Fri Jun 28 22:18:36 2019 us=581589 MULTI TCP: multi_tcp_dispatch a=TA_TUN_READ mi=0x00000000
Fri Jun 28 22:18:36 2019 us=581598  read from TUN/TAP returned 48
Fri Jun 28 22:18:36 2019 us=581605 MULTI TCP: multi_tcp_post TA_TUN_READ -> TA_UNDEF
Fri Jun 28 22:18:36 2019 us=581611 SCHEDULE: schedule_find_least NULL
Fri Jun 28 22:18:40 2019 us=589838 EP_WAIT[0] rwflags=0x0001 ev=0x00000001 arg=0x00000002
Fri Jun 28 22:18:40 2019 us=589891 MULTI: REAP range 16 -> 32
Fri Jun 28 22:18:40 2019 us=589899 MULTI TCP: multi_tcp_action a=TA_TUN_READ p=0
Fri Jun 28 22:18:40 2019 us=589904 MULTI TCP: multi_tcp_dispatch a=TA_TUN_READ mi=0x00000000
Fri Jun 28 22:18:40 2019 us=589913 NOTE: --mute triggered...

```

- 验证tun网卡设备

```bash
[root@centos openvpn]$ifconfig tun0
tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.8.0.1  netmask 255.255.255.255  destination 10.8.0.2
        inet6 fe80::3107:a40a:9e56:e30e  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 100  (UNSPEC)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3  bytes 144 (144.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

## 二、windows PC安装openvpn客户端

官方下载：https://openvpn.net/community-downloads/

非官方下载：https://sourceforge.net/projects/securepoint/files/

下载安装即可

### - 2.1：windows客户端测试连接

保存证书到openvpn客户端安装目录：F:\OpenVPN\config

```bash
[root@centos ~]$cd /etc/openvpn/client/handsomejack/
[root@centos handsomejack]$ll
total 20
-rw------- 1 root root 1172 Jun 28 20:49 ca.crt
-rw-r--r-- 1 root root  325 Jun 28 22:16 client.ovpn
-rw------- 1 root root 4446 Jun 28 20:50 handsomejack.crt
-rw------- 1 root root 1704 Jun 28 20:54 handsomejack.key
[root@centos handsomejack]$tar czvf handsomejack.tar.gz ./*
./ca.crt
./client.ovpn
./handsomejack.crt
./handsomejack.key
[root@centos handsomejack]$sz handsomejack.tar.gz #将压缩包传至windows端

```

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561806566537.png)

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561807968734.png)

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561808079664.png)

### - 2.2：windows客户端验证通信

### - 2.3：验证windows路由表

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561808332823.png)

### - 2.4：验证客户端与内网服务器通信

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561808481348.png)

### - 2.5：xshell验证ssh连接内网服务器及状态

```bash
[root@centos ~]$ssh root@10.20.30.3
The authenticity of host '10.20.30.3 (10.20.30.3)' can't be established.
ECDSA key fingerprint is SHA256:hwmwefpGsP1+Or9zGxfs+JFeBycZrc2b4uYBH/3joQg.
ECDSA key fingerprint is MD5:33:68:fc:8f:0f:d7:45:54:91:fc:ef:37:cc:36:a9:31.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.20.30.3' (ECDSA) to the list of known hosts.
root@10.20.30.3's password: 
Last login: Sat Jun 29 18:54:42 2019
[root@centos ~]$ss -ntla
State       Recv-Q Send-Q       Local Address:Port                      Peer Address:Port              
LISTEN      0      128                      *:22                                   *:*                  
LISTEN      0      100              127.0.0.1:25                                   *:*                  
ESTAB       0      0               10.20.30.3:22                          10.20.30.6:51890              
LISTEN      0      128                     :::22                                  :::*                  
LISTEN      0      100                    ::1:25                                  :::* 

```

## 三、高级功能

主要是员工入职、离职涉及到的创建账户与吊销账户证书

### - 3.1：密钥设置连接密码

新建一个名为javis的账户，并且设置证书密码，提高证书安全性

```bash
[root@centos ~]$cd /etc/openvpn/client/easy-rsa/3.0.3/
[root@centos 3.0.3]$pwd
/etc/openvpn/client/easy-rsa/3.0.3
[root@centos 3.0.3]$./easyrsa gen-req javis
Generating a 2048 bit RSA private key
..........................................................................+++
.........+++
writing new private key to '/etc/openvpn/client/easy-rsa/3.0.3/pki/private/javis.key.nTK9kTqYEz'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [javis]:

Keypair and certificate request completed. Your files are:
req: /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/javis.req
key: /etc/openvpn/client/easy-rsa/3.0.3/pki/private/javis.key

[root@centos 3.0.3]$cd /etc/openvpn/easy-rsa/3.0.3/
[root@centos 3.0.3]$pwd
/etc/openvpn/easy-rsa/3.0.3
[root@entos 3.0.3]$./easyrsa import-req /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/javis.req javis

Note: using Easy-RSA configuration from: ./vars

The request has been successfully imported with a short name of: javis
You may now use this name to perform signing operations on this request.

[root@centos 3.0.3]$./easyrsa sign client javis

Note: using Easy-RSA configuration from: ./vars


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 3650 days:

subject=
    commonName                = javis


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'javis'
Certificate is to be certified until Jun 26 11:57:34 2029 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/easy-rsa/3.0.3/pki/issued/javis.crt

[root@centos 3.0.3]$mkdir /etc/openvpn/client/javis
[root@centos 3.0.3]$cd /etc/openvpn/client/javis
[root@centos javis]$cp /etc/openvpn/easy-rsa/3.0.3/pki/ca.crt .
[root@centos javis]$cp /etc/openvpn/easy-rsa/3.0.3/pki/issued/javis.crt .
[root@centos javis]$cp /etc/openvpn/client/easy-rsa/3.0.3/pki/private/javis.key .
[root@centos javis]$cp /etc/openvpn/client/handsomejack/client.ovpn . #基于之前的账户生成并修改新客户配置文件

```

- 连接测试

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561810121220.png)

### - 3.2：账户证书管理

主要是证书的创建和吊销，对应的员工的入职和离职

#### - 3.2.1：证书自动过期

过期时间以服务器时间为准，开始检查证书的有效期是否在服务器时间为准的有效期内

```bash
[root@centos ~]$cd /etc/openvpn/easy-rsa/3.0.3/
[root@centos 3.0.3]$pwd
/etc/openvpn/easy-rsa/3.0.3
[root@centos 3.0.3]$vim vars
117 #set_var EASYRSA_CERT_EXPIRE    3650 #默认3650天
118 set_var EASYRSA_CERT_EXPIRE     60 #更改为60天有效期

```

#### - 3.2.2：证书手动注销

```bash
[root@centos 3.0.3]$cat /etc/openvpn/easy-rsa/3/pki/index.txt #状态V有效，R为吊销
V	290625121105Z		DF18FF1E3E8249FB0E2D4E80330C4E6E	unknown	/CN=server
V	290625123851Z		CA7AECBFEAB5CC57D6BF0E71CAC03E4A	unknown	/CN=handsomejack
V	290626115734Z		AFA803A237F8E89E258DA3934F23896D	unknown	/CN=javis

[root@centos 3.0.3]$cd /etc/openvpn/easy-rsa/3.0.3/
[root@centos 3.0.3]$./easyrsa revoke javis  #吊销名为javis的用户

Note: using Easy-RSA configuration from: ./vars


Please confirm you wish to revoke the certificate with the following subject:

subject= 
    commonName                = javis


Type the word 'yes' to continue, or any other input to abort.
  Continue with revocation: yes
Using configuration from ./openssl-1.0.cnf
Revoking Certificate AFA803A237F8E89E258DA3934F23896D.
Data Base Updated

IMPORTANT!!!

Revocation was successful. You must run gen-crl and upload a CRL to your
infrastructure in order to prevent the revoked cert from being accepted.

#证书吊销文件
[root@centos 3.0.3]$./easyrsa gen-crl

Note: using Easy-RSA configuration from: ./vars
Using configuration from ./openssl-1.0.cnf

An updated CRL has been created.
CRL file: /etc/openvpn/easy-rsa/3.0.3/pki/crl.pem

#编辑配置文件调用吊销证书的文件
[root@centos 3.0.3]$vim /etc/openvpn/server.conf
dh /etc/openvpn/certs/dh.pem
crl-verify /etc/openvpn/easy-rsa/3.0.3/pki/crl.pem
[root@centos 3.0.3]$systemctl restart openvpn@server

#验证吊销证书的用户状态
[root@centos 3.0.3]$cat /etc/openvpn/easy-rsa/3/pki/index.txt
V	290625121105Z		DF18FF1E3E8249FB0E2D4E80330C4E6E	unknown	/CN=server
V	290625123851Z		CA7AECBFEAB5CC57D6BF0E71CAC03E4A	unknown	/CN=handsomejack
R	290626115734Z	190629122342Z	AFA803A237F8E89E258DA3934F23896D	unknown	/CN=javis

```

- 验证吊销证书的用户连接

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/OpenVPN/1561811580422.png)

#### - 3.2.3：账户重名证书签发

假如公司已有员工叫zhangxiaoming已经离职且证书已被吊销，但是又新来一个员工还叫zhangxiaoming，那么一般的区分办法是在用户名后面加数字，如zhangxiaoming1、zhangxiaoming2等，假如还想使用zhangxiaoming这个账户名签发证书的话，那么需要删除服务器之前zhangxiaoming的账户，并删除签发记录和证书，否则新用户的证书无法导入，具体如下

```bash
#删除已离职被撤销的账户证书
[root@centos ~]$cd /etc/openvpn/client/easy-rsa/3.0.3/
[root@centos 3.0.3]$rm -f pki/private/javis.key 
[root@centos 3.0.3]$rm -f pki/reqs/javis.req 
[root@centos 3.0.3]$rm -rf /etc/openvpn/client/javis/*
[root@centos 3.0.3]$rm -rf /etc/openvpn/easy-rsa/3.0.3/pki/reqs/javis.req 
[root@centos 3.0.3]$rm -rf /etc/openvpn/easy-rsa/3.0.3/pki/issued/javis.crt 
[root@centos 3.0.3]$vim /etc/openvpn/easy-rsa/3/pki/index.txt #删除带R的吊销记录

#重新生成新的账户并签发crt证书
[root@centos ~]$cd /etc/openvpn/client/easy-rsa/3.0.3/
[root@centos 3.0.3]$pwd
/etc/openvpn/client/easy-rsa/3.0.3
[root@centos 3.0.3]$./easyrsa gen-req javis
...

#CA导入证书并签发
[root@centos 3.0.3]$cd /etc/openvpn/easy-rsa/3.0.3/
[root@centos 3.0.3]$pwd
/etc/openvpn/easy-rsa/3.0.3
[root@entos 3.0.3]$./easyrsa import-req /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/javis.req javis
...

#打包证书给客户端
[root@centos 3.0.3]$cd /etc/openvpn/client/javis/
[root@centos javis]$cp /etc/openvpn/easy-rsa/3.0.3/pki/issued/javis.crt .
[root@centos javis]$cp /etc/openvpn/client/easy-rsa/3.0.3/pki/private/javis.key .
[root@centos javis]$cp ../handsomejack/client.ovpn . #修改客户端文件中的证书名称
[root@centos javis]$cp ../handsomejack/ca.crt . #ca的公钥
[root@centos javis]$tree
.
├── ca.crt
├── client.ovpn
├── javis.crt
└── javis.key

0 directories, 4 files

```

#### - 3.2.4：server端最终配置文件

```bash
[root@centos javis]$egrep -v "^(#|;)" /etc/openvpn/server.conf | grep -v "^$"
local 172.20.133.222
port 1194
proto tcp
dev tun
ca /etc/openvpn/certs/ca.crt
cert /etc/openvpn/certs/server.crt
key /etc/openvpn/certs/server.key  # This file should be kept secret
dh /etc/openvpn/certs/dh.pem
crl-verify /etc/openvpn/easy-rsa/3.0.3/pki/crl.pem
server 10.8.0.0 255.255.255.0
push "route 10.20.30.0 255.255.255.0"
push "route 172.20.0.0 255.255.0.0"
client-to-client
keepalive 10 120
cipher AES-256-CBC
max-clients 100
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
log-append  /var/log/openvpn/openvpn.log
verb 9
mute 20

```

#### - 3.2.5：client端最终配置

```bash
[root@centos javis]$egrep -v "^(#|;)" /etc/openvpn/client/javis/client.ovpn | grep -v "^$"
client
dev tun
proto tcp
remote 172.20.133.222 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert javis.crt
key javis.key
remote-cert-tls server
cipher AES-256-CBC
verb 3

```

### - 3.3：自动创建用户脚本

```bash
[root@centos ~]$cat openvpn_create_user.sh
#!/bin/bash
read -p "请输入用户名: " USER
cd /etc/openvpn/client/easy-rsa/3.0.3/
echo "开始生成客户端证书"
expect <<EOF
spawn ./easyrsa gen-req ${USER}
expect "Enter" {send "${USER}\n"}
expect "Verifying" {send "${USER}\n"}
expect "Common" {send "\n"}
expect eof
EOF
echo "正在签发客户端证书"
cd /etc/openvpn/easy-rsa/3.0.3/
./easyrsa import-req /etc/openvpn/client/easy-rsa/3.0.3/pki/reqs/${USER}.req ${USER}
expect <<EOF
spawn ./easyrsa sign client ${USER}
expect "Confirm" {send "yes\n"}
expect eof
EOF
echo "正在生成打包文件"
mkdir /etc/openvpn/client/${USER}
cd /etc/openvpn/client/${USER}
cp /etc/openvpn/easy-rsa/3.0.3/pki/ca.crt .
cp /etc/openvpn/easy-rsa/3.0.3/pki/issued/${USER}.crt .
cp /etc/openvpn/client/easy-rsa/3.0.3/pki/private/${USER}.key .
cp /etc/openvpn/client/handsomejack/client.ovpn .
sed -ri 's/^cert(.*)/#cert \1/' client.ovpn
sed -ri 's/^key(.*)/#key \1/' client.ovpn
echo "cert $USER.crt" >> client.ovpn
echo "key $USER.key" >> client.ovpn
tar zcvf ${USER}.tar.gz ./*

```
