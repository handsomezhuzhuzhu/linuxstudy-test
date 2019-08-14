## 一、Docker基础

### - 1.1：Docker简介

#### - 1.1.1：Docker

Docker采用客户端/服务端架构，使用远程API来管理和创建Docker容器，可以轻松创造一个轻量级、可移植的、自给自足的容器，通过namespace及cgroup来提供容器的资源隔离与安全保障，运行时不需要额外开销，大幅提高了资源利用率

- Docker三大理念
  - build（一次构建）
  - ship（随意运输）
  - run（到处运行）

#### - 1.1.2：Docker组成

主机（Host）：一个物理机或虚拟机，用于运行Docker服务进程和容器

服务端（Server）：Docker守护进程，运行docker容器

客户端（Client）：客户使用docker命令或其他工具调用docker API

仓库（Registry）：保存镜像的仓库，类似于git或svn的版本控制系统，官方仓库： https://hub.docker.com/

镜像（Images）：创建实例使用的模板

容器（Container）：从镜像生成对外提供服务的一个或一组服务

![image](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1562156247153.png)

#### - 1.1.3：Docker对比虚拟机优缺点

- 优点：
  - 快速部署：短时间内可以部署成百上千个应用，更快速交付到线上
  - 高效虚拟化：不需要额外的haypervisor支持，直接基于linux实现应用虚拟化，相比虚拟机大幅提升性能与效率
  - 节省开支：提高服务器利用率，降低IT支出
  - 简化配置：将运行环境打包保存至容器，使用时直接启动即可
  - 快速迁移和扩展：可跨平台运行在物理机、虚拟机、公有云等环境，良好的兼容性
- 缺点：
  - 各应用之间的隔离不如虚拟机彻底

![image](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1562156367244.png)

- 一台宿主机上运行N个容器，可能会引起的问题如何解决？

```bash
#1.如何保证每个容器都有不同的文件系统并且能互不影响
#2.一个docker主进程内的各个容器都是其子进程，如何实现同一个主进程下不同类型的子进程？各个进程间通信能互相访问（内存数据）吗？
#3.每个容器如何解决IP及端口分配的问题?
#4.多个容器的主机名能一样吗？
#5.每个容器都要不要有root用户？怎么解决账户重名问题？
```

#### - 1.1.4：Linux Namespace技术

实现容器运行空间的相互隔离

| 隔离类型                           | 功能                               | 系统调用参数  | 内核版本     |
| ---------------------------------- | ---------------------------------- | ------------- | ------------ |
| MNT （mount）                      | 提供磁盘挂载点和文件系统的隔离能力 | CLONE_NEWNS   | Linux 2.4.19 |
| IPC（Inter-Process Communication） | 提供进程间通信的隔离能力           | CLONE_NEWIPC  | Linux 2.6.19 |
| UTS（UNIX Timesharing System）     | 提供主机名隔离能力                 | CLONE_NEWUTS  | Linux 2.6.19 |
| PID（Process Identification）      | 提供进程隔离能力                   | CLONE_NEWPID  | Linux 2.6.24 |
| Net （network）                    | 提供网络隔离能力                   | CLONE_NEWNET  | Linux 2.6.29 |
| User（user）                       | 提供用户隔离能力                   | CLONE_NEWUSER | Linux 3.8    |

##### - 1.1.4.1：MNT Namespace

每个容器都要有独立的根文件系统有独立的用户空间，以实现在容器里面启动服务并且使用容器的运行环境，即一个宿主机是 ubuntu 的服务器，可以在里面启动一个 centos 运行环境的容器并且在容器里面启动一个 Nginx 服务，此 Nginx运行时使用的运行环境就是 centos 系统目录的运行环境，但是在容器里面是不能访问宿主机的资源，宿主机是使用了 chroot 技术把容器锁定到一个指定的运行目录里面

##### - 1.1.4.2：IPC Namespace

一个容器内的进程间通信，允许一个容器内的不同进程的(内存、缓存等)数据访问，但是不能夸容器访问其他容器的数据

##### - 1.1.4.3：UTS Namespace

包含了运行内核的名称、版本、底层体系结构类型等信息，用于系统标识，其中包含了 hostname 和域名domainname ，它使得一个容器拥有属于自己 hostname 标识，这个主机名标识独立于宿主机系统和其上的其他容器

```bash
root@ubuntu-1804:~# docker exec -it 22220d300010 bash
root@22220d300010:/# uname -a
Linux 22220d300010 4.15.0-29-generic #31-Ubuntu SMP Tue Jul 17 15:39:52 UTC 2018 x86_64 GNU/Linux
root@22220d300010:/# hostname
22220d300010
```

##### - 1.1.4.4：PID Namespace

每个容器内都要有一个父进程来管理其下属的子进程，那么多个容器的进程通过 PID namespace 进程隔离(比如 PID 编号重复、容器内的主进程生成与回收子进程等)

![image](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1562209999643.png)

##### - 1.1.4.5：Net Namespace

每一个容器都类似于虚拟机一样有自己的网卡、监听端口、TCP/IP 协议栈等，Docker 使用 network namespace 启动一个 vethX 接口，这样你的容器将拥有它自己的桥接 ip 地址，通常是 docker0，而 docker0 实质就是 Linux 的虚拟网桥,网
桥是在 OSI 七层模型的数据链路层的网络设备，通过 mac 地址对网络进行划分，并且在不同网络直接传递数据

![image](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1562210268228.png)

```bash
ifconfig  #查看宿主机的网卡信息
brctl show #查看宿主机桥接设备
iptables -t nat -nvL #查看宿主机iptables规则
```

##### - 1.1.4.6：User Namespace

User Namespace 允许在各个宿主机的各个容器空间内创建相同的用户名以及相同的用户 UID 和 GID，只是会把用户的作用范围限制在每个容器内，即 A 容器和 B 容器可以有相同的用户名称和 ID 的账户，但是此用户的有效范围仅是当前容器内，不能访问另外一个容器内的文件系统，即相互隔离、互补影响、永不相见

- 每个容器内都有超级管理员root及其他普通用户，且与其他容器相同

#### - 1.1.5：Linxu control groups

在一个容器，如果不对其做任何资源限制，则宿主机会允许其占用无限大的内存空间，有时候会因为代码 bug 程序会一直申请内存，直到把宿主机内存占完，为了避免此类的问题出现，宿主机有必要对容器进行资源分配限制，比如CPU、内存等，Linux Cgroups 的全称是 Linux Control Groups，它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。此外，还能够对进程进行优先级设置，以及将进程挂起和恢复等操作

##### - 1.1.5.1：验证系统cgroups

![image](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1562210797385.png)

##### - 1.1.5.2：cgroup具体实现

![image](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1562210983049.png)

```bash
blkio：块设备 IO 限制。
cpu：使用调度程序为 cgroup 任务提供 cpu 的访问。
cpuacct：产生 cgroup 任务的 cpu 资源报告。
cpuset：如果是多核心的 cpu，这个子系统会为 cgroup 任务分配单独的 cpu 和内存
devices：允许或拒绝 cgroup 任务对设备的访问。
freezer：暂停和恢复 cgroup 任务。
memory：设置每个 cgroup 的内存限制以及产生内存资源报告。
net_cls：标记每个网络包以供 cgroup 方便使用。
ns：命名空间子系统。
perf_event：增加了对每 group 的监测跟踪的能力，可以监测属于某个特定的 group 的所有线程以及运行在特定 CPU 上的线程
```

#### - 1.1.6：容器管理工具

Docker 启动一个容器也需要一个外部模板docker镜像，docker 的镜像可以保存在一个公共的地方共享使用，只要把镜像下载下来就可以使用，最主要的是可以在镜像基础之上做自定义配置并且可以再把其提交为一个镜像，一个镜像可以被启动为多个容器

Docker 的镜像是分层的，镜像底层为库文件且只读层即不能写入也不能删除数据，从镜像加载启动为一个容器后会生成一个可写层，其写入的数据会复制到容器目录，但是容器内的数据在删除容器后也会被随之删除

![image](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1562214753310.png)

##### - 1.1.6.1：pouch

阿里云开源容器技术：https://www.infoq.cn/article/alibaba-pouch

git地址：https://github.com/alibaba/pouch

#### - 1.1.7：docker核心技术

##### - 1.1.7.1：容器规范

- runtime spec
- image format spec

##### - 1.1.7.1：容器runtime

runtime是真正运行容器的地方，因此为了运行不同的容器runtime需要和操作系统内核紧密合作相互支持，以便为容器提供相应的运行环境

- Lxc：早期使用
- runc：Docker默认的runtime，可以兼容lxc
- rkt：CoreOS开发的容器runtime

##### - 1.1.7.2：容器管理工具

管理工具连接runtime与用户，对用户提供图形或命令方式操作，然后管理工具将用户操作传递给runtime执行

- Runc的管理工具是docker engine，包括后台daemon和cli两部分，经常提到的 Docker 就是指的 docker engine

##### - 1.1.7.3：容器定义工具

容器定义工具允许用户定义容器的属性和内容，一遍容器能够被保存、共享和重建

- Docker image：docker容易的模板，runtime依据docker image创建容器
- Dockerfile：包含N个命令的文本文件，通过dockerfile创建出docker image
- ACI：与docker image类似，时CoreOS开发的rkt容器的镜像格式

##### - 1.1.7.4：Registry-镜像仓库

统一保存镜像

- Image registry：docker官方提供的私有仓库部署工具
- Docker hub：docker官方的公共仓库
- Harbor：vmware提供的自带web界面自带认证功能的镜像仓库

##### - 1.1.7.5：编排工具

- 容器编排引擎：实现统一管理，动态伸缩、故障自愈、批量执行等功能
- 容器编排通常包括容器管理、调度、集群定义和服务发现等功能
- Docker swarm：docker开发的容器编排引擎
- kubernetes：google开发，支持docker与CoreOS
- Mesos+Marathon：通用的集群组员调度平台，mesos（资源分配）与marathon（容器编排平抬）一起提供容器编排引擎功能

#### - 1.1.8：docker的依赖技术

- 容器网络：

  docker 自带的网络 docker network 仅支持管理单机上的容器网络，当多主机运行的时候需要使用第三方开源网络，例如 calico、flannel 等

- 服务发现：

  容器的动态扩容特性决定了容器 IP 也会随之变化，因此需要有一种机制开源自动识别并将用户请求动态转发到新创建的容器上，kubernetes 自带服务发现功能，需要结合 kube-dns 服务解析内部域名

- 容器监控：

  可以通过原生命令 docker ps/top/stats 查看容器运行状态，另外也可以使heapster/ Prometheus 等第三方监控工具监控容器的运行状态

- 数据管理;

  容器的动态迁移会导致其在不同的 Host 之间迁移，因此如何保证与容器相关的数据也能随之迁移或随时访问，可以使用逻辑卷/存储挂载等方式解决

- 日志收集：

  docker 原生的日志查看工具 docker logs，但是容器内部的日志需要通过 ELK 等专门的日志收集分析和展示工具进行处理
