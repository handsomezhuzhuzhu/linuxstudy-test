## 一、环境准备

docker：192.168.30.46

mysql+redis：192.168.30.56

## 二、安装docker

```bash
[root@centos ~]$cd /etc/yum.repos.d/
[root@centos yum.repos.d]$wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
[root@centos yum.repos.d]$ll
total 24
...
-rw-r--r--  1 root root 2640 Jun 29 15:42 docker-ce.repo
[root@centos yum.repos.d]$yum install docker-ce -y #若是提示selinux版本太低则百度解决方法
[root@centos yum.repos.d]$systemctl start docker  #启动服务
[root@centos yum.repos.d]$ll /etc/docker/key.json 
-rw------- 1 root root 244 Jun 30 12:06 /etc/docker/key.json

#配置docker镜像下载加速器
登录https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors获取专属加速地址
[root@centos yum.repos.d]$vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://4n2wu429.mirror.aliyuncs.com"]
}
[root@centos yum.repos.d]$systemctl restart docker

#使用docker下载jumpserver镜像
[root@centos ~]$docker pull jumpserver/jms_all:1.4.8
```

## 三、创建数据库和redis

- 要求：
  - mysql版本需要大于等于5.6
  - mariadb版本需要大于等于5.56
  - 数据库编码要求uft8
- 创建

```bash
#安装数据库（安装完成后需要在docker端测试连接）
[root@centos ~]$yum install mariadb-server -y
[root@centos ~]$systemctl start mariadb
[root@centos ~]$systemctl enable mariadb
[root@centos ~]$mysql
MariaDB [(none)]> create database jumpserver default charset 'utf8';
MariaDB [(none)]> grant all on jumpserver.* to 'jumpserver'@'%' identified by 'centos';

#安装redis
[root@centos ~]$yum install redis -y
[root@centos ~]$vim /etc/redis.conf
61 bind 0.0.0.0
480 requirepass centos
[root@centos ~]$systemctl start redis
[root@centos ~]$systemctl enable redis

#查看端口监听
[root@centos ~]$ss -nlt
State       Recv-Q Send-Q       Local Address:Port                      Peer Address:Port           
LISTEN      0      128                      *:6379                                 *:*             
LISTEN      0      80                      :::3306                                :::*             ...  
```

## 四：最终配置

在docker端

```bash
[root@centos ~]$mkdir -p /opt/mysql
[root@centos ~]$mkdir /opt/jumpserver
[root@centos ~]$if [ "$SECRET_KEY" = "" ]; then SECRET_KEY=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50`; echo "SECRET_KEY=$SECRET_KEY" >> ~/.bashrc; echo $SECRET_KEY; else echo $SECRET_KEY; fi
4lDjFXj1g1RN3K7u3socqqz5c1bAvXn0yulv2o8iXE3yvLCQ6f
[root@centos ~]$if [ "$BOOTSTRAP_TOKEN" = "" ]; then BOOTSTRAP_TOKEN=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16`; echo "BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN" >> ~/.bashrc; echo $BOOTSTRAP_TOKEN; else echo $BOOTSTRAP_TOKEN; fi
HEQXc0G0HSgRGyQP

[root@centos ~]$docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
jumpserver/jms_all   1.4.8               e9274ba449e8        3 months ago        1.31GB
#检查有上述镜像后再进行下面步骤
[root@centos ~]$docker run --name mydocker -d \
    -v /opt/mysql:/var/lib/mysql \
    -v /opt/jumpserver:/opt/jumpserver/data/media \
    -p 80:80 \
    -p 2222:2222 \
    -e SECRET_KEY=4lDjFXj1g1RN3K7u3socqqz5c1bAvXn0yulv2o8iXE3yvLCQ6f \
    -e BOOTSTRAP_TOKEN=HEQXc0G0HSgRGyQP \
    -e DB_HOST=192.168.30.56 \
    -e DB_PORT=3306 \
    -e DB_USER=jumpserver \
    -e DB_PASSWORD=centos \
    -e DB_NAME=jumpserver \
    -e REDIS_HOST=192.168.30.56 \
    -e REDIS_PORT=6379 \
    -e REDIS_PASSWORD=centos \
    jumpserver/jms_all:1.4.8
    #执行完成后生成一串字符
    4b3c4e93e9a6e6ae5b68f7d1c1ce28a6fb32e5cead41b4f1d22d36cf26ebc653
    
    #检查docker的状态
[root@centos ~]$docker ps
CONTAINER ID        IMAGE      COMMAND       CREATED    STATUS        PORTS          NAMES
4b3c4e93e9a6        jumpserver/jms_all:1.4.8   "entrypoint.sh"     2 minutes ago       Up 2 minutes        0.0.0.0:80->80/tcp, 0.0.0.0:2222->2222/tcp   mydocker
[root@centos ~]$docker logs 4b3c4e93e9a6  #查看日志
```

- 检查无误之后登陆web界面

![image](https://github.com/handsomezhuzhuzhu/images/blob/master/Jumpserver/1561870859388.png)

## 五、之后的操作在web界面上进行即可

参考文档：https://jumpserver.readthedocs.io/zh/master/admin_user.html

用户组：可登录客户端的用户的集合

用户：登录到jumpserver客户端的用户

管理用户：一般为root

系统用户：登录到被管理服务器进行操作的用户

