---
layout: post
title: Linux YCSB Benchmark
tags: [linux, YCSB]
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

#### systemctl start mysql失败
![250722-image11](\assets\250722-image11.png)

#### mysqld --initialize遇到问题
```shell
(py310) root@flyslice-System-Product-Name:~/xyy/test-mount# mysqld --initialize.
... Failed to set datadir to '/root/xyy/test-mount/mysql/' (OS errno: 13 - Permission denied)
```

原因是没有权限
```shell
sudo chown -R mysql:mysql /root/xyy/test-mount/mysql/
sudo chmod -R 755 /root/xyy/test-mount/mysql/
namei -l /root/xyy/test-mount/mysql/
```
![250722-image12](\assets\250722-image12.png)
建议mount到别的目录比如/mnt下，/root下权限会有问题。

#### 遇到问题OS errno 17 - File exists：

![250722-image13](\assets\250722-image13.png)

原因是因为`apparmor`

参考 [https://www.cnblogs.com/linxiyue/p/8229048.html](https://www.cnblogs.com/linxiyue/p/8229048.html)

![250722-image14](\assets\250722-image14.png)

修改
```shell
vim /etc/apparmor.d/usr.sbin.mysqld
```
然后重启apparmor应用更新
```shell
service apparmor restart
```

记得加上权限
```shell
sudo chown -R mysql:mysql /mnt/test_ssd/mysql_data
sudo chmod -R 755 /mnt/test_ssd/mysql_data
```
检查结果，重新生成数据库需要的是数据，成功了。
```shell
sudo rm -rf /mnt/test_ssd/mysql_data/*
sudo mysqld --initialize --user=mysql --datadir=/mnt/test_ssd/mysql_data

ls -la /mnt/test_ssd/mysql_data
```
![250722-image15](\assets\250722-image15.png)

可以重启mysql了。
```shell
sudo systemctl start mysql
```

#### 查看随机密码
```shell
tail /var/log/mysql/error.log
```
![250722-image16](\assets\250722-image16.png)

```shell
sudo mysql -uroot -p
Bu-Jn-&5IpOv

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';
```

进入mysql检查，成功修改挂载目录。
```shell
sudo mysql -uroot -p
123456
SHOW VARIABLES LIKE 'datadir';
```
![250722-image17](\assets\250722-image17.png)
![250722-image18](\assets\250722-image18.png)
![250722-image19](\assets\250722-image19.png)

### 最终测试参数
Load
```shell
bin/ycsb load jdbc -P workloads/workloada -P jdbc-binding/conf/db.properties   -cp mysql-connector-java-8.0.33.jar   -p recordcount=1000000   -p insertstart=0 -s > enterprise_load.log
```
Run
```shell
bin/ycsb run jdbc -P workloads/workloada -P jdbc-binding/conf/db.properties   -cp mysql-connector-java-8.0.33.jar   -p recordcount=1000000   -p insertstart=0 -s > enterprise_run.log
```

注：
![250722-image20](\assets\250722-image20.png)

运行完一次benchmark后需要清除数据库重新生成文件,以防重复key错误。
```
SELECT COUNT(*) FROM usertable;
```
![250722-image21](\assets\250722-image21.png)

进入mysql，输入
```mysql
TRUNCATE TABLE usertable;
```
或直接输入shell
```shell
export MYSQL_PWD="123456"
mysql -u root -e "USE test; TRUNCATE TABLE usertable;"
```

查看生成的表格数量
```shell
mysql -u root -e "USE test; SELECT COUNT(*) FROM usertable;"
```

## 场景2：在Liunx下配置运行YCSB基准（以对接数据库rocksdb为例，基于rocksdb-binding）
rocksdb多用于存储元数据，作为底层性能测试更为合适。

## 步骤
参考[官网步骤](https://github.com/brianfrankcooper/YCSB/tree/master/rocksdb)


### 重新克隆YCSB库
```bash
git clone https://github.com/brianfrankcooper/YCSB.git
```

克隆不下来网络不稳定用以下代替
```bash
git clone --depth 1 https://github.com/brianfrankcooper/YCSB.git
```

### 通过mvn编译rocksdb-binding
```bash
cd YCSB
mvn -pl site.ycsb:rocksdb-binding -am clean package
```


### 前置条件：maven、python软连接
#### maven安装
需要用系统默认安装maven包
```bash
apt install maven
```

#### mvn修改换源
查找`setting.yml`位置
```bash
find / -name "setting.yml"
```
![250722-image24](\assets\250722-image24.png)

```bash
vi /etc/maven/settings.xml
```
![250722-image25](\assets\250722-image25.png)

```
<mirrors>
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
</mirrors>
```

![250722-image26](\assets\250722-image26.png)

验证配置:
```bash
mvn help:effective-settings
```

![250722-image27](\assets\250722-image27.png)


再次运行mvn即可。


#### 需要软连接python3到python
```bash
sudo ln -s /usr/bin/python3 /usr/bin/python
```
![250722-image22](\assets\250722-image22.png)

![250722-image23](\assets\250722-image23.png)


### 运行benchmark
![250722-image28](\assets\250722-image28.png)
