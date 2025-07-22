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
### 1.修改配置文件，需要修改

