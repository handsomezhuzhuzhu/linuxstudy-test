## 一、虚拟化基础

### - 1.1：传统的物理机部署方案

- 服务器选型及采购-->IDC选择及上架-->系统选择及安装-->应用规划及部署-->域名选择及注册-->DNS映射-->外网访问

- 传统数据中心面临的问题

```bash
服务器资源利用低下，cpu、内存等不能共享
资源分配不合理
初始化成本高
自动化能力差
集群环境需要大量的服务器主机
```

### - 1.2：虚拟化

- 虚拟化是为一些组件（例如虚拟应用、服务器、存储和网络）创建基于软件的（或虚拟）表现形式的过程

- 虚拟化的优势：

```bash
降低资金成本和运维成本
最大限度减少或消除停机
提高IT部门的工作效率、效益、敏捷性和响应能力
加快应用和资源的调配速度
提高业务连续性和灾难恢复能力
简化数据中心管理
```

#### - 1.2.2：虚拟机

- 虚拟计算机系统：一种严密隔离且内含操作系统和应用的软件容器

- 虚拟机的主要特性

```bash
#分区：
可在一台物理机上运行多个操作系统
可在虚拟机之间分配系统资源
#隔离
可在硬件级别进行故障和安全隔离
可利用高级资源控制功能保持性能
#封装
可将虚拟机的完成状态保存到文件中
通过复制文件即可移动和复制虚拟机
#独立于硬件，可将任意虚拟机调配或迁移到任意物理服务器上
```

#### - 1.2.3：虚拟化类型

- 服务器虚拟化

```bash
#优势：
提高IT效率
降低运维成本
更快的部署工作负载
提高服务器可用性
提高应用性能
消除服务器数量剧增情况和复杂性
```

- 网络虚拟化
- 桌面虚拟化
- 应用虚拟化
- 存储虚拟化
- 库虚拟化
- 容器虚拟化

#### - 1.2.4：云计算

- 云计算分类：

```bash
公有云：比如aws、阿里云以及azure、金山云、腾讯云等都属于公有云，每个人都可以付费使用，不需要自己关心底层硬件，但是数据安全需要考虑
私有云：在自己公司内部或IDC自建Openstack、VMware等环境
混合云：既要使用公有云，又要使用私有云，即自己的私有云的部分业务和公有云有交接，这部分称为混合云
```

- 云计算分层：

```bash
IaaS：基础设施服务 #自建机房
PaaS：平台服务 #公有云上的redis、RDS等服务
SaaS：软件服务 #企业邮箱、OA系统
```

#### - 1.2.5：虚拟化技术分类

- 模拟器：

  在一个host之上通过虚拟化模拟器软件，模拟出一个硬件或者多个硬件环境，每个环境都是一个独立的虚拟机，CPU、IO、内存等都是模拟出来的，可以在宿主机模拟出不同于当前物理机CPU指令集的虚拟机，比如可以在Windows 模拟出mac OS、unix系统，比较出名的模拟器有：pearpc、QEMU、Bochs

- 全虚拟化/准虚拟化：

  全虚拟化不做CPU和内存模拟，只对CPU和内存做相应的分配等操作，完全虚拟化需要物理硬件的支持，比如需要CPU必须支持并且打开虚拟化功能，因此完全虚拟化是基于硬件辅助的虚拟化技术，vmware workstation、KVM

- 半虚拟化：

  半虚拟化要求guest OS 的内核是知道自己运行在虚拟化环境当中的，因此guestOS的系统架构必须和宿主机的系统架构相同，并且要求对guest OS的内核做相应的修改，因此半虚拟化只支持开源内核的系统，不支持闭源的系统，比较常见的半虚拟化就是早期版本的XEN

## 二、KVM

- KVM是基于硬件的完全虚拟化，需要硬件支持

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/KVM/1.jpg)

```bash
Guest：客户机系统，包括CPU（vCPU）、内存、驱动（Console、网卡、I/O 设备驱动等），被KVM置于一种受限制的CPU模式下运行。
KVM：运行在内核空间，提供CPU和内存的虚级化，以及客户机的I/O拦截，Guest的部分I/O被KVM拦截后，交给QEMU处理。
QEMU：修改过的被KVM虚机使用的QEMU代码，运行在用户空间，提供硬件I/O虚拟化，通过IOCTL/dev/kvm设备和KVM交互，但是，KVM本身不执行任何硬件模拟，需要用户空间程序通过 /dev/kvm 接口设置一个客户机虚拟服务器的地址空间，向它提供模拟I/O，并将它的视频显示映射回宿主的显示屏。目前这个应用程序是QEMU
```
### - 2.1：整体架构

- 第一阶段

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/KVM/1560661304906.png)

- 第二阶段

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/KVM/1560690386007.png)

### - 2.2：宿主机环境准备

#### - 2.2.1：CPU开启虚拟化

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/KVM/1560661669590.png)

- 开启宿主机，配置统一格式网卡名

```bash
[root@centos ~]$mv /etc/sysconfig/network-scripts/ifcfg-ens33 /etc/sysconfig/network-scripts/ifcfg-eth0
[root@centos ~]$vim /etc/sysconfig/network-scripts/ifcfg-eth0
NAME=eth0
DEVICE=eth0
[root@centos ~]$vim /etc/default/grub 
GRUB_CMDLINE_LINUX="crashkernel= rhgb net.ifnames=0 biosdevname=0 quiet "
[root@centos ~]$grub2-mkconfig -o /boot/grub2/grub.cfg
[root@centos ~]$reboot
```

#### - 2.2.2：网卡bond、bridge配置

- centos7.2之前版本创建bridge需要安装包bridge-utils

```bash
#eth0配置（其余1、2、3以此为例配置）
[root@centos network-scripts]$vim ifcfg-eth0
BOOTPROTO=static
NAME=eth0
DEVICE=eth0
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#bond0配置（bond1以此为例配置）
[root@centos network-scripts]$vim ifcfg-bond0
OTPROTO=static
NAME=bond0
DEVICE=bond0
ONBOOT=yes
TYPE=Bond
BONDING_MASTER=yes
BONDING_OPTS="mode=1 miimon=100" #指定绑定类型为1及链路状态监测1间隔时间100ms
BRIDGE=br0 #桥接到br0

#brideg0配置
[root@centos network-scripts]$vim ifcfg-br0
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=172.20.30.46
NETMASK=255.255.0.0
GATEWAY=172.20.0.1
DNS1=172.20.0.1

#brideg1配置
[root@centos network-scripts]$vim ifcfg-br1
TYPE=Bridge
BOOTPROTO=static
NAME=br1
DEVICE=br1
ONBOOT=yes
IPADDR=10.20.30.46
NETMASK=255.255.255.0
GATEWAY=10.20.30.1

#查看
[root@centos network-scripts]$ifconfig 
bond0: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet6 fe80::250:56ff:fe3a:fb20  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:3a:fb:20  txqueuelen 1000  (Ethernet)
        RX packets 204090  bytes 256205226 (244.3 MiB)
        RX errors 0  dropped 10  overruns 0  frame 0
        TX packets 756  bytes 196746 (192.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

bond1: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet6 fe80::250:56ff:fe36:6a89  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:36:6a:89  txqueuelen 1000  (Ethernet)
        RX packets 51  bytes 9068 (8.8 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 11  bytes 856 (856.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.30.46  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::250:56ff:fe3a:fb20  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:3a:fb:20  txqueuelen 1000  (Ethernet)
        RX packets 8672  bytes 2257400 (2.1 MiB)
        RX errors 0  dropped 12  overruns 0  frame 0
        TX packets 744  bytes 183844 (179.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

br1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.20.30.46  netmask 255.255.255.0  broadcast 10.20.30.255
        inet6 fe80::250:56ff:fe36:6a89  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:36:6a:89  txqueuelen 1000  (Ethernet)
        RX packets 40  bytes 6339 (6.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7  bytes 490 (490.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:3a:fb:20  txqueuelen 1000  (Ethernet)
        RX packets 10577  bytes 3062920 (2.9 MiB)
        RX errors 0  dropped 7  overruns 0  frame 0
        TX packets 43  bytes 3312 (3.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:3a:fb:20  txqueuelen 1000  (Ethernet)
        RX packets 200649  bytes 255025444 (243.2 MiB)
        RX errors 0  dropped 10  overruns 0  frame 0
        TX packets 756  bytes 196746 (192.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth2: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:36:6a:89  txqueuelen 1000  (Ethernet)
        RX packets 50  bytes 8627 (8.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 26  bytes 2018 (1.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth3: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 00:50:56:36:6a:89  txqueuelen 1000  (Ethernet)
        RX packets 7  bytes 801 (801.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 18  bytes 1448 (1.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 25  bytes 2072 (2.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 25  bytes 2072 (2.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
#访问测试
[root@centos network-scripts]$ping www.baidu.com
PING www.a.shifen.com (61.135.169.121) 56(84) bytes of data.
64 bytes from 61.135.169.121 (61.135.169.121): icmp_seq=1 ttl=56 time=3.03 ms
64 bytes from 61.135.169.121 (61.135.169.121): icmp_seq=2 ttl=56 time=3.06 ms
^C
--- www.a.shifen.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 3.038/3.050/3.062/0.012 ms
```

#### - 2.2.3：安装KVM工具包

```bash
[root@centos ~]$yum install qemu-kvm qemu-kvm-tools libvirt virt-manager virt-install -y
[root@centos ~]$systemctl start libvirtd
[root@centos ~]$systemctl enable libvirtd
[root@centos ~]$ifconfig  virbr0
virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:bd:ba:f7  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
[root@centos ~]$grep "192.168.122.1" /etc/libvirt/ -R #通过修改下面文件来修改默认virbr0的ip
/etc/libvirt/qemu/networks/autostart/default.xml:  <ip address='192.168.122.1' netmask='255.255.255.0'>
/etc/libvirt/qemu/networks/default.xml:  <ip address='192.168.122.1' netmask='255.255.255.0'>
[root@centos ~]$ll /etc/libvirt/qemu/networks/autostart/default.xml
lrwxrwxrwx 1 root root 14 Jun 16 14:57 /etc/libvirt/qemu/networks/autostart/default.xml -> ../default.xml #上述两文件其实是一个文件和软链接
```

### - 2.3：创建NAT网络虚拟机

#### - 2.3.1：创建磁盘

```bash
[root@centos ~]$ll /var/lib/libvirt/images/ #确认保存虚拟机磁盘的路径
total 0
[root@centos ~]$qemu-img create -f qcow2 /var/lib/libvirt/images/centos1.qcow2 10G #创建一个格式为qcow2大小为10G的裸磁盘
Formatting '/var/lib/libvirt/images/centos1.qcow2', fmt=qcow2 size=10737418240 encryption=off cluster_size=65536 lazy_refcounts=off 
[root@centos ~]$ll /var/lib/libvirt/images/centos1.qcow2
-rw-r--r-- 1 root root 197120 Jun 16 15:36 /var/lib/libvirt/images/centos1.qcow2
```

#### - 2.3.2：virsh-install命令使用帮助

```bash
[root@centos ~]$virt-install -h
usage: virt-install --name NAME --memory MB STORAGE INSTALL [options] #命令语法

Create a new virtual machine from specified install media. #使用指定安装介质新建虚拟机

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
  --connect URI         Connect to hypervisor with libvirt URI #使用libvirt URI连接到hypervisor

General Options: #通用选项
  -n NAME, --name NAME  Name of the guest instance #客户端事件名称
  --memory MEMORY       Configure guest memory allocation. Ex: #配置虚拟机内存分配
                        --memory 1024 (in MiB)
                        --memory 512,maxmemory=1024
                        --memory 512,maxmemory=1024,hotplugmemorymax=2048,hotplugmemoryslots=2
  --vcpus VCPUS         Number of vcpus to configure for your guest. Ex: #为虚拟机配置的vcpus数
                        --vcpus 5
                        --vcpus 5,maxcpus=10,cpuset=1-4,6,8
                        --vcpus sockets=2,cores=4,threads=2,
  --cpu CPU             CPU model and features. Ex: #cpu型号及功能
                        --cpu coreduo,+x2apic
                        --cpu host-passthrough
                        --cpu host
  --metadata METADATA   Configure guest metadata. Ex: #配置虚拟机元数据
                        --metadata name=foo,title="My pretty title",uuid=...
                        --metadata description="My nice long description"

Installation Method Options: #安装方法选项
  --cdrom CDROM         CD-ROM installation media 光驱安装介质
  -l LOCATION, --location LOCATION #安装源
                        Installation source (eg, nfs:host:/path、http://host/path、ftp://host/path)
  --pxe                 Boot from the network using the PXE protocol #使用PXE协议从网络引导
  --import              Build guest around an existing disk image #在磁盘映像中构建虚拟机
  --livecd              Treat the CD-ROM media as a Live CD #将光驱介质视为live CD
  -x EXTRA_ARGS, --extra-args EXTRA_ARGS #附加到使用--location引导的内核的参数
                        Additional arguments to pass to the install kernel
                        booted from --location
  --initrd-inject INITRD_INJECT #使用--location为initrd的root添加给定文件
                        Add given file to root of initrd from --location
  --os-variant DISTRO_VARIANT #在其中安装OS变体的虚拟机
                        The OS variant being installed guests, e.g.
                        'fedora18', 'rhel6', 'winxp', etc.
  --boot BOOT           Configure guest boot settings. Ex: #配置虚拟机引导设置
                        --boot hd,cdrom,menu=on
                        --boot init=/sbin/init (for containers)
  --idmap IDMAP         Enable user namespace for LXC container. Ex: #为LXC容器启用用户名称空间
                        --idmap uid_start=0,uid_target=1000,uid_count=10

Device Options: #设备选项
  --disk DISK           Specify storage with various options. Ex. #使用不同选项指定存储
                        --disk size=10 (new 10GiB image in default location)
                        --disk /my/existing/disk,cache=none
                        --disk device=cdrom,bus=scsi
                        --disk=?
  -w NETWORK, --network NETWORK #配置虚拟机网络接口
                        Configure a guest network interface. Ex:
                        --network bridge=mybr0
                        --network network=my_libvirt_virtual_net
                        --network network=mynet,model=virtio,mac=00:11...
                        --network none
                        --network help
  --graphics GRAPHICS   Configure guest display settings. Ex: #配置虚拟机显示设置
                        --graphics vnc
                        --graphics spice,port=5901,tlsport=5902
                        --graphics none
                        --graphics vnc,password=foobar,port=5910,keymap=ja
  --controller CONTROLLER #配置虚拟机控制程序设备
                        Configure a guest controller device. Ex:
                        --controller type=usb,model=ich9-ehci1
  --input INPUT         Configure a guest input device. Ex: #配置虚拟机输入设备
                        --input tablet
                        --input keyboard,bus=usb
  --serial SERIAL       Configure a guest serial device #配置虚拟机串口设备
  --parallel PARALLEL   Configure a guest parallel device #配置虚拟机并口设备
  --channel CHANNEL     Configure a guest communication channel #配置虚拟机沟通频道
  --console CONSOLE     Configure a text console connection between the guest #配置虚拟机与主机之间的文本控制台连接
                        and host
  --hostdev HOSTDEV     Configure physical USB/PCI/etc host devices to be #将物理USB/PCI/etc主机设备配置为与虚拟机共享
                        shared with the guest
  --filesystem FILESYSTEM #将主机目录传递给虚拟机
                        Pass host directory to the guest. Ex: 
                        --filesystem /my/source/dir,/dir/in/guest
                        --filesystem template_name,/,type=template
  --sound [SOUND]       Configure guest sound device emulation #虚拟机声音设备模拟
  --watchdog WATCHDOG   Configure a guest watchdog device #配置虚拟机看门狗设备
  --video VIDEO         Configure guest video hardware. #配置虚拟机视频硬件
  --smartcard SMARTCARD 
                        Configure a guest smartcard device. Ex: #配置虚拟机智能卡设备
                        --smartcard mode=passthrough
  --redirdev REDIRDEV   Configure a guest redirection device. Ex: #配置虚拟机重定向设备
                        --redirdev usb,type=tcp,server=192.168.1.1:4000
  --memballoon MEMBALLOON
                        Configure a guest memballoon device. Ex: #配置虚拟机memballoon设备
                        --memballoon model=virtio
  --tpm TPM             Configure a guest TPM device. Ex: #配置虚拟机TPM设备
                        --tpm /dev/tpm
  --rng RNG             Configure a guest RNG device. Ex: #配置虚拟机RNG设备
                        --rng /dev/urandom
  --panic PANIC         Configure a guest panic device. Ex: #配置虚拟机panic设备
                        --panic default
  --memdev MEMDEV       Configure a guest memory device. Ex:
                        --memdev dimm,target_size=1024

Guest Configuration Options: #虚拟机配置选项
  --security SECURITY   Set domain security driver configuration. #设定域安全驱动器配置
  --cputune CPUTUNE     Tune CPU parameters for the domain process. 
  --numatune NUMATUNE   Tune NUMA policy for the domain process. #为域进程调整NUMA策略
  --memtune MEMTUNE     Tune memory policy for the domain process. #为域进程调整内存策略
  --blkiotune BLKIOTUNE #为域进程调整blkio策略
                        Tune blkio policy for the domain process.
  --memorybacking MEMORYBACKING #为域进程设置内存后背策略
                        Set memory backing policy for the domain process. Ex:
                        --memorybacking hugepages=on
  --features FEATURES   Set domain <features> XML. Ex: #设置域<features>XML
                        --features acpi=off
                        --features apic=on,eoi=on
  --clock CLOCK         Set domain <clock> XML. Ex:
                        --clock offset=localtime,rtc_tickpolicy=catchup
  --pm PM               Configure VM power management features #配置VM电源管理功能
  --events EVENTS       Configure VM lifecycle management policy #配置VM生命周期管理策略
  --resource RESOURCE   Configure VM resource partitioning (cgroups) #配置VM资源分区
  --sysinfo SYSINFO     Configure SMBIOS System Information. Ex:
                        --sysinfo emulate
                        --sysinfo host
                        --sysinfo bios_vendor=Vendor_Inc.,bios_version=1.2.3-abc,...
                        --sysinfo system_manufacturer=System_Corp.,system_product=Computer,...
                        --sysinfo baseBoard_manufacturer=Baseboard_Corp.,baseBoard_product=Motherboard,...
  --qemu-commandline QEMU_COMMANDLINE
                        Pass arguments directly to the qemu emulator. Ex:
                        --qemu-commandline='-display gtk,gl=on'
                        --qemu-commandline env=DISPLAY=:0.1

Virtualization Platform Options: #虚拟化平台选项
  -v, --hvm             This guest should be a fully virtualized guest #客户端应该是一个全虚拟客户端
  -p, --paravirt        This guest should be a paravirtualized guest #客户端应该是一个半虚拟客户端
  --container           This guest should be a container guest #此虚拟机需要一个容器客户端
  --virt-type HV_TYPE   Hypervisor name to use (kvm, qemu, xen, ...) #要使用的管理程序名称
  --arch ARCH           The CPU architecture to simulate #模拟的CPU架构
  --machine MACHINE     The machine type to emulate #要摸你的机器类型

Miscellaneous Options: #其它选项
  --autostart           Have domain autostart on host boot up. #引导主机时自启动域
  --transient           Create a transient domain.
  --wait WAIT           Minutes to wait for install to complete. #等待安装完成的分钟数
  --noautoconsole       Don't automatically try to connect to the guest #不要地洞尝试连接到客户端控制台
                        console
  --noreboot            Don't boot guest after completing install. #完成安装后不要引导虚拟机
  --print-xml [XMLONLY]
                        Print the generated domain XML rather than create the guest. #输出所生成域 XML，而不是创建虚拟机
  --dry-run             Run through install process, but do not create devices or define the guest. #完成安装步骤，但不要创建设备或者定义虚拟机
  --check CHECK         Enable or disable validation checks. Example: #启用或禁用验证检查
                        --check path_in_use=off
                        --check all=off
  -q, --quiet           Suppress non-error output #禁止无错误输出
  -d, --debug           Print debugging information #输入故障排除信息

Use '--option=?' or '--option help' to see available suboptions
See man page for examples and full option syntax.
```

#### - 2.3.3：上传镜像并安装虚拟机

- 安装第一台虚拟机

```bash
[root@centos ~]$cd /usr/local/src/
[root@centos src]$wget http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso
virt-install --virt-type kvm \
--name centos7 \
--ram 1024 \
--vcpus 2 \
--cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1810.iso \
--disk path=/var/lib/libvirt/images/centos1.qcow2 \
--network network=default \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole
[root@centos src]$virt-manager #此时会弹出xshell界面，安装安装系统的步骤安装即可（xshell必须使用企业版）

#查看所有虚拟机
[root@centos src]$virsh list --all 
 Id    Name                           State
----------------------------------------------------
 2     centos7                        running
[root@centos src]$yum install acpid -y #安装电源管理包
[root@centos src]$virsh start centos7 #开启虚拟机
```

- 安装第二台虚拟机

```bash
#本着在linux中一切皆文件的原则，第二台就不用手动安装了，而是复制第一台的文件
[root@centos images]$cp centos1.qcow2 centos2.qcow2 
[root@centos images]$ls
centos1.qcow2  centos2.qcow2
[root@centos images]$virt-install --virt-type kvm \
> --name centos777 \ #修改主机名
> --ram 1024 \
> --vcpus 2 \
> --cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1810.iso \
> --disk path=/var/lib/libvirt/images/centos2.qcow2 \ #修改磁盘名
> --network network=default \
> --graphics vnc,listen=0.0.0.0 \
> --noautoconsole
Starting install...
Domain installation still in progress. You can reconnect to 
the console to complete the installation process.
[root@centos images]$virt-manager
#需要注意的是进入安装界面后选择强制关机，然后再重新启动即可免安装，不然又需要手动安装
```

#### - 2.3.4：登录到虚拟机

- 用vm1、vm2、vm3、vm4实现keepalived+haproxy架构

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/KVM/1560690386007.png)

接下来的步骤之前已经实现过，就不再赘述

- 虚拟机的配置选项界面

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/KVM/1560690756568.png)

（需要注意的是新装虚拟机需要关闭firewalld、selinux等）

