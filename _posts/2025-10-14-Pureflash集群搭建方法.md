---
layout: post
title: Pureflash集群搭建方法
tags: [Pureflash, conductor, 分布式存储, Pureflash集群搭建]
---

** *本文档来自@umuzhaohui*

# PureFlash集群构建

## 配置apt源

```
# 集群每台机器的/etc/apt/sources.list配置apt源
cat /etc/apt/sources.list

deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
```

## 修改DNS
```
sudo vi /etc/resolv.conf
# 增加以下
nameserver 8.8.8.8
```

## 安装依赖软件包

```
apt update && apt install cgdb curl gzip jq libaio1 libcurl4   libibverbs1 libicu-dev libjsoncpp25 librdmacm1 readline-common libstdc++6 libtool  libuuid1 tar unzip  util-linux vim wget  net-tools  ninja-build libcurl4-openssl-dev libcppunit-dev uuid-dev libaio-dev nasm autoconf cmake librdmacm-dev pkg-config g++ default-jdk ant meson libssl-dev ncurses-dev libnuma-dev help2man python3-pip libfuse3-dev
apt update && apt upgrade cgdb curl gzip jq libaio1 libcurl4   libibverbs1 libicu-dev libjsoncpp25 librdmacm1 readline-common libstdc++6 libtool  libuuid1 tar unzip  util-linux vim wget  net-tools  ninja-build libcurl4-openssl-dev libcppunit-dev uuid-dev libaio-dev nasm autoconf cmake librdmacm-dev pkg-config g++ default-jdk ant meson libssl-dev ncurses-dev libnuma-dev help2man python3-pip libfuse3-dev

apt install git
pip3 install pyelftools 
apt install putty-tools # 需要plink
```

<!-- ## (虚机里)关闭防火墙
```
# 默认开启防火墙，git clone无法访问口443，需关闭
sudo ufw disable
``` -->

## 下载编译pureflash

```
mkdir -p /home/flyslice/yangxiao/cocalele/
cd /home/flyslice/yangxiao/cocalele/
git clone https://github.com/cocalele/PureFlash.git
cd PureFlash/
mkdir build_deb; cd build_deb && cmake -GNinja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_MAKE_PROGRAM=/usr/bin/ninja .. && ninja
```

## 下载编译 jconductor

```
cd /home/flyslice/yangxiao/cocalele/
git clone https://github.com/cocalele/jconductor.git
cd jconductor
git submodule update --init
ant -f jconductor.xml
```

## 启动zookeeper

```
cd /home/flyslice/

# 安装OpenJDK 8, 每台机器执行
apt update
apt install openjdk-8-jdk
# 验证安装
java -version

# 开放端口, 每台机器执行
sudo ufw allow 2181/tcp
sudo ufw allow 2888/tcp
sudo ufw allow 3888/tcp

# 解压安装包, 每台机器执行
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.7.2/apache-zookeeper-3.7.2-bin.tar.gz
tar -xzvf apache-zookeeper-3.7.2-bin.tar.gz -C /opt
rm -rf apache-zookeeper-3.7.2-bin.tar.gz

# 创建数据目录, 每台机器执行
mkdir -p /var/lib/zookeeper/data

# 配置myid文件, 每台机器执行
# 创建myid文件，其内容为该服务器唯一的数字ID(与zoo.cfg中的server.x对应)
echo "1" > /var/lib/zookeeper/data/myid # 第一台机器执行
echo "2" > /var/lib/zookeeper/data/myid # 第二台机器执行
echo "3" > /var/lib/zookeeper/data/myid # 第三台机器执行

# 配置文件, 每台机器执行
cp /opt/apache-zookeeper-3.7.2-bin/conf/zoo_sample.cfg /opt/apache-zookeeper-3.7.2-bin/conf/zoo.cfg

配置zoo.cfg内容:
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
syncLimit=5
# the directory where the snapshot is stored.
dataDir=/var/lib/zookeeper/data
# the port at which the clients will connect
clientPort=2181
# list of cluster servers
server.1=192.168.61.3:2888:3888
server.2=192.168.61.195:2888:3888
server.3=192.168.61.34:2888:3888

# 启动zookeeper, 每台机器执行
/opt/apache-zookeeper-3.7.2-bin/bin/zkServer.sh start

# 验证集群状态, 结果显示leader或follower, 每台机器执行
/opt/apache-zookeeper-3.7.2-bin/bin/zkServer.sh status

# 客户端连接, 任意一台机器执行
/opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh -server 192.168.61.229:2181
/opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh -server 192.168.61.143:2181
/opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh -server 192.168.61.122:2181

/opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh -server 192.168.61.3:2181

# 停止服务, 可选执行
/opt/apache-zookeeper-3.7.2-bin/bin/zkServer.sh stop
```

## 启动mariadb

```
cd /home/flyslice/

# 安装关软件包, 每台机器执行
apt update
apt install mariadb-server mariadb-client galera-4 -y
apt install rsync -y

# 开放端口, 每台机器执行
ufw allow 3306/tcp
ufw allow 4567/tcp
ufw allow 4568/tcp
ufw allow 4444/tcp

# 配置文件, 每台机器执行
cat /etc/mysql/conf.d/galera.cnf

[mysqld]
binlog_format = ROW
default-storage-engine = InnoDB
innodb_autoinc_lock_mode = 2
bind-address = 0.0.0.0

wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "my_galera_cluster"
wsrep_cluster_address = "gcomm://192.168.61.229,192.168.61.143,192.168.61.122"
wsrep_node_name = "node1"  # 在第二个节点改为 "node2"，第三个改为 "node3"
wsrep_node_address = "192.168.61.229" # 在第二个节点改为 "node2_ip"，第三个改为 "node3_ip"
wsrep_sst_method = rsync 
wsrep_sst_auth = "sst_user:your_secure_password"


```bash
[mysqld]
binlog_format = ROW
default-storage-engine = InnoDB
innodb_autoinc_lock_mode = 2
bind-address = 0.0.0.0

wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "my_galera_cluster"
wsrep_cluster_address = "gcomm://192.168.61.3,192.168.61.195,192.168.61.34"
wsrep_node_name = "node1"
wsrep_node_address = "192.168.61.3"
wsrep_sst_method = rsync
wsrep_sst_auth = "sst_user:your_secure_password"
```


# 确保MariaDB已停止, 第一个节点执行
systemctl stop mariadb

# 初始化新集群, 第一个节点执行
galera_new_cluster 

#如果配置了wsrep_sst_auth, 创建SST用户, 在第一个节点或任一节点初始化后，登录MySQL创建相应用户：
mysql -u root -p
Enter password: ------输入your_secure_password
..............................
MariaDB [(none)]> CREATE USER 'sst_user'@'%' IDENTIFIED BY 'your_secure_password';
Query OK, 0 rows affected (0.002 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON *.* TO 'sst_user'@'%';
Query OK, 0 rows affected (0.001 sec)
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.001 sec)
MariaDB [(none)]> EXIT;
Bye

#第一个节点的MariaDB服务应该已经运行。可以通过以下命令检查其状态：
systemctl status mariadb

#第一个节点上验证集群规模，能看到只有当前节点加入
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';" 
Enter password: -----输入your_secure_password
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 1     |
+--------------------+-------+

#启动MariaDB服务, 其他节点执行
systemctl start mariadb

#MariaDB服务状态, 其他节点执行
systemctl status mariadb

#在任一节点上验证集群规模, 能看到所有节点都已加入
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';" 
Enter password: -----输入your_secure_password
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+

#在任一节点上，你可以运行以下命令来检查集群的健康状况：
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"        # 查看集群大小，确认所有节点都已加入
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_ready';"               # 确保复制状态为 ON
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment';" # 查看节点状态，了解其在集群中的角色
mysql -u root -p -e "SHOW GLOBAL STATUS LIKE 'wsrep%';"             # 查看更多 WSREP 状态变量


#授权pureflash用户
mysql -e "GRANT ALL PRIVILEGES ON *.* TO 'pureflash'@'%' IDENTIFIED BY '123456'"

#导入表格
mysql -e "source /home/flyslice/yangxiao/cocalele/jconductor/res/init_s5metadb.sql"
```

## 安装**Keepalived**配置mariadb集群的VIP

```
# 安装keepalived软件包
apt-get install keepalived

# 主节点配置文件, 备份节点的配置与主节点类似, 需要修改state为BACKUP, 并且priority值要低于主节点。
cat /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL  # 标识，可自定义
}

vrrp_instance VI_1 {
    state MASTER         
    interface eno2       
    virtual_router_id 51
    priority 100         

    advert_int 1         
    authentication {     
        auth_type PASS
        auth_pass 1111   
    }

    virtual_ipaddress {
        192.168.61.111/24 
    }
}

# 主备服务器上启动Keepalived服务并设置开机自启
systemctl start keepalived
systemctl enable keepalived

# 在主节点上使用ip addr show [interface]命令查看配置的VIP是否已经绑定到指定的网络接口上
ip addr show eno2
<!-- ens18 -->
```

## 创建配置文件

```
mkdir /etc/pureflash/
touch pf.conf pfc.conf pfs.conf # 配置文件内容见附录
```

## 启动 pfserver

### 清理硬盘

pfserver启动之前，要清理磁盘的头部的10G空间

#### 卸载挂载点

```
lsblk  # 在输出中查看 "MOUNTPOINTS" 列，确认是否有分区被挂载
df -h  # 另一种查看已挂载文件系统的方法

# 如果发现例如 `/dev/nvme1n1` 被挂载到了 `/mnt/data`，
umount /dev/nvme1n1 # 通过硬盘卸载
umount /mnt/data    # 通过挂载点卸载

# 卸载时遇到“target is busy”错误，表示有进程正在使用该挂载点。
lsof +f -- /dev/sdb1 # 查找并终止相关进程
umount /dev/nvme1n1  # 使用懒卸载(强制卸载，但可能不安全)
lsblk                # 确认卸载成功
```

#### 删除分区

```
# 使用parted删除分区
parted /dev/sdb # 进入 (parted) 交互式命令行
(parted) print  # 查看分区
(parted) rm 1   # 假设有一个分区1, 删除分区, 重复print和 rm <分区号> 命令，直到所有分区都被删除。
(parted) quit   # 退出 parted


# 使用 fdisk(用于MBR分区表)或gdisk(用于GPT分区表)来删除分区
fdisk /dev/nvme1n1 # 在 fdisk 提示符下，输入 d 来删除分区，然后输入 w 将更改写入并退出
```

#### 内容清理

```
# 使用 shred
shred -v -n 3 -z /dev/nvme1n1 # 对整个磁盘进行3次随机写入覆盖，最后用零覆盖一次并删除分区表：


# 使用dd
dd if=/dev/zero of=/dev/nvme1n1 status=progress bs=1M
```

### 启动服务

```
source /home/flyslice/yangxiao/cocalele/PureFlash/build_deb/env.sh
nohup pfs -c /etc/pureflash/pfs.conf &
```

## 启动 pfconductor
修改pfc
```
JCROOT=/home/flyslice/yangxiao/cocalele/jconductor
```

```
source /home/flyslice/yangxiao/cocalele/jconductor/env-pfc.sh
nohup pfc -c /etc/pureflash/pfc.conf > /home/flyslice/yangxiao/pfconductor.log 2>&1 &

<!-- nohup pfc -c /etc/pureflash/pfc.conf & -->
```

启动顺序：
db、zk、conductor、pfstore

## 附录

### pf.conf

```
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.229:2181,192.168.61.143:2181,192.168.61.122:2181
[client]
conn_type=tcp
```

### pfc.conf

```
# 192.168.61.229节点
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.229:2181,192.168.61.143:2181,192.168.61.122:2181
[conductor]
mngt_ip=192.168.61.229
[db]
ip=127.0.0.1
user=pureflash
pass=123456
db_name=s5

# 192.168.61.143节点
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.229:2181,192.168.61.143:2181,192.168.61.122:2181
[conductor]
mngt_ip=192.168.61.143
[db]
ip=127.0.0.1
user=pureflash
pass=123456
db_name=s5

# 192.168.61.122节点
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.229:2181,192.168.61.143:2181,192.168.61.122:2181
[conductor]
mngt_ip=192.168.61.122
[db]
ip=127.0.0.1 
user=pureflash
pass=123456
db_name=s5
```

### pfs.conf

```
# 192.168.61.229节点
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.229:2181,192.168.61.143:2181,192.168.61.122:2181
[afs]
mngt_ip=192.168.61.229 
id=1                   
meta_size=10737418240
[engine]
name=aio
[tray.0]
dev=/dev/nvme2n1       
[tray.1]
dev=/dev/nvme4n2
[port.0]
ip=192.168.61.229
[rep_port.0]
ip=192.168.61.229
[tcp_server]
poller_count=8
[replicator]
conn_type=tcp
count=4

# 192.168.61.143节点
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.229:2181,192.168.61.143:2181,192.168.61.122:2181
[afs]
mngt_ip=192.168.61.143
id=2                   
meta_size=10737418240
[engine]
name=aio
[tray.0]
dev=/dev/nvme0n1
[port.0]
ip=192.168.61.143
[rep_port.0]
ip=192.168.61.143
[tcp_server]
poller_count=8
[replicator]
conn_type=tcp
count=4

# 192.168.61.122节点
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.229:2181,192.168.61.143:2181,192.168.61.122:2181
[afs]
mngt_ip=192.168.61.122
id=3                  
meta_size=10737418240
[engine]
name=aio
[tray.0]
dev=/dev/nvme1n1
[port.0]
ip=192.168.61.122
[rep_port.0]
ip=192.168.61.122
[tcp_server]
poller_count=8
[replicator]
conn_type=tcp
count=4
```


注:
虚拟机中搭建必须也要分配至少一个盘，可以通过虚拟机管理器分配，然后用`lsblk`查看

```
root@flyslice-Standard-PC-i440FX-PIIX-1996:/home/flyslice/yangxiao/cocalele/jconductor# lsblk
NAME   MAJ:MIN RM    SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0      4K  1 loop /snap/bare/5
loop1    7:1    0     55M  1 loop /snap/core18/1880
loop2    7:2    0   55.5M  1 loop /snap/core18/2959
loop3    7:3    0  255.6M  1 loop /snap/gnome-3-34-1804/36
loop4    7:4    0  218.4M  1 loop /snap/gnome-3-34-1804/93
loop5    7:5    0   91.7M  1 loop /snap/gtk-common-themes/1535
loop6    7:6    0   62.1M  1 loop /snap/gtk-common-themes/1506
loop7    7:7    0   49.8M  1 loop /snap/snap-store/467
loop8    7:8    0   29.9M  1 loop /snap/snapd/8542
sda      8:0    0      1T  0 disk 
├─sda1   8:1    0    512M  0 part /boot/efi
├─sda2   8:2    0      1K  0 part 
└─sda5   8:5    0 1023.5G  0 part /
sdb      8:16   0     32G  0 disk 
sr0     11:0    1    2.6G  0 rom
```

这里都用sdb作为盘，写入pfs配置文件，并重新运行pfs。

```
<!-- vi /etc/pureflash/pfs.conf -->
[cluster]
name=cluster1
[zookeeper]
ip=192.168.61.3:2181,192.168.61.195:2181,192.168.61.34:2181
[afs]
mngt_ip=192.168.61.3
id=1
meta_size=10737418240
[engine]
name=aio
#name=spdk
[tray.0]
dev=/dev/sdb
#dev=/dev/nvme0n1     # path of physical flash device
#dev=trtype:PCIE traddr:0000.03.00.0
[tray.1]
#dev=/dev/nvme1n1
#dev=trtype:PCIE traddr:0000.04.00.0
[port.0]
ip=192.168.61.3
[rep_port.0]
ip=192.168.61.3
[tcp_server]
poller_count=8
[replicator]
conn_type=tcp
count=4
```

