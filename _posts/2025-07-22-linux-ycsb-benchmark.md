---
layout: post
title: Linux YCSB Benchmark
tags: [YCSB]
---

# [YCSB] 使用YCSB对数据库性能测试
## 场景：在Liunx下配置运行YCSB基准（以对接数据库mysql为例，基于jdbc引擎）

## 简介
Yahoo! Cloud Serving Benchmark (YCSB) 是一个Java语言实现的主要用于云端或者服务器端的数据库性能测试工具。

## 前置步骤：下载配置mysql
参考[在Ubuntu 22.04 LTS上安装MySQL](https://blog.csdn.net/weixin_45626288/article/details/133220238)

## 前置步骤2：配置ycsb
使用YCSB对数据库性能测试

官网 [https://github.com/brianfrankcooper/YCSB/tree/master/jdbc](https://github.com/brianfrankcooper/YCSB/tree/master/jdbc)
教程参考[https://piaohua.github.io/post/mysql/20220723-ycsb/](https://piaohua.github.io/post/mysql/20220723-ycsb/)

## 部署YCSB
### 下载ycsb代码
```bash
curl -O --location https://github.com/brianfrankcooper/YCSB/releases/download/0.17.0/ycsb-0.17.0.tar.gz 
tar xfvz ycsb-0.17.0.tar.gz 
cd ycsb-0.17.0
```
(github不稳定，通过Windows直接下载压缩包后传到服务器上替代)

### 选择合适的绑定（Binding）此处以jdbc-binding为例
![250722-image1](\assets\250722-image1.png)
查看当前目录可以看到-binding都是可以选择的，对应不同的数据库场景。
![250722-image2](\assets\250722-image2.png)

![250722-image3](\assets\250722-image3.png)

### 修改ycsb配置

```bash
vi /root/xyy/ycsb-0.17.0/jdbc-binding/conf/db.properties
```

![250722-image4](\assets\250722-image4.png)

```shell
db.driver=com.mysql.jdbc.Driver
db.url=jdbc:mysql://127.0.0.1:3306/ycsb
db.user=root
db.passwd=123456
```

创建test数据表，创建新的dataset
```mysql
SHOW DATABASES;
```
![250722-image5](\assets\250722-image5.png)

```mysql
CREATE DATABASE test;
USE test;
CREATE TABLE usertable (
	YCSB_KEY VARCHAR(255) PRIMARY KEY,
	FIELD0 TEXT, FIELD1 TEXT,
	FIELD2 TEXT, FIELD3 TEXT,
	FIELD4 TEXT, FIELD5 TEXT,
	FIELD6 TEXT, FIELD7 TEXT,
	FIELD8 TEXT, FIELD9 TEXT
);
```

### 选择jdbc驱动，安装下载mysql-connector-java-j-8.0.33.jar
在官网[https://downloads.mysql.com/archives/c-j/](https://downloads.mysql.com/archives/c-j/)下载合适的版本

![250722-image6](\assets\250722-image6.png)

这里以8.0.33版本为例
Windows下载后可以直接复制到服务器上
![250722-image7](\assets\250722-image7.png)

创建临时目录
```shell
mkdir temp-deb && cd temp-deb
```
将下载的包解压
```shell
ar x ~/xyy/mysql-connector-j_8.0.33-1ubuntu20.04_all.deb
tar -xf data.tar.xz
find . -name "*.jar"
```
![250722-image8](\assets\250722-image8.png)

移动jar包到ycsb目录下
```shell
cp ./usr/share/java/mysql-connector-j-8.0.33.jar ~/xyy/ycsb-0.17.0/jdbc-binding/lib/
```

配置好后就可以运行workload

![250722-image9](\assets\250722-image9.png)


## 修改数据库datadir（用于对比测试不同ssd上的性能）
### 1.修改配置文件
需要修改
```shell
vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
里面的datadir

以及apparmor的配置文件

```shell
vim /etc/apparmor.d/usr.sbin.mysqld
```

修改后执行
```shell
service apparmor restart
```
Ps: 企业级盘数据库datadir         = /mnt/test_ssd/mysql_data/

消费级datadir         =/mnt/test_ssd_1602/mysql/

apparmor设置:

企业级
```shell
# Allow data dir access
#  /var/lib/mysql/ r,
#  /var/lib/mysql/** rwk,
  /mnt/test_ssd/mysql_data/ r,
  /mnt/test_ssd/mysql_data/** rwk,
```

手动初始化数据库命令：
```shell
sudo mysqld --initialize --user=mysql --datadir=/mnt/test_ssd/mysql_data
```

消费级
```shell
#  /var/lib/mysql/ r,
#  /var/lib/mysql/** rwk,
#  /mnt/test_ssd/mysql_data/ r,
#  /mnt/test_ssd/mysql_data/** rwk,
/mnt/test_ssd_1602/mysql/ r,
/mnt/test_ssd_1602/mysql/** rwk,
```

手动初始化数据库命令：
```shell
sudo mysqld --initialize --user=mysql --datadir=/mnt/test_ssd_1602/mysql
```

### 2.修改后刷新，重启mysql服务
```shell
sudo systemctl stop mysql
sudo systemctl start mysql
```
详情见“手动测试，重启服务”

### 3.修改workload文件，增加并发以及唯一性
例如：
```shell
vi /root/xyy/ycsb-0.17.0workloads/workloada
```
![250722-image10](\assets\250722-image10.png)

threadcount只并发线程，需要增加insertorder=hased
以及参数-p insertstart=0来避免Duplicate entry
Error 如下：
```shell
Error in processing insert to table: usertablejava.sql.SQLIntegrityConstraintViolationException: Duplicate entry 'user8753205170136912308' for key 'usertable.PRIMARY'
Error inserting, not retrying any more. number of attempts: 1Insertion Retry Limit: 0
```

### 手动测试，重启服务
关闭
```shell
sudo systemctl stop mysql
```
开启
```shell
sudo systemctl start mysql
```

