---
layout: post
title: 安装ycsb-rocksdb步骤
tags: [YCSB]
---

## 步骤：
- (1)git clone下载官网YCSB 代码
```bash
git clone --depth 1 https://github.com/brianfrankcooper/YCSB.git
```

- (2)安装`maven`
```bash
apt install maven
```

- (3)软连接python3到python
```bash
sudo ln -s /usr/bin/python3 /usr/bin/python
```

- (4)mvn修改换源

```bash
vi /etc/maven/settings.xml
```
在mirror处添加以下内容
```
<mirrors>
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
</mirrors>
```
![251023-image-ycsb](\assets\251023-image-ycsb.png)

	注: 验证配置
	```bash
	mvn help:effective-settings
	```

- (5)安装rocksdb
```bash
cd YCSB
mvn -pl site.ycsb:rocksdb-binding -am clean package
```


### 验证YCSB
```
./bin/ycsb load rocksdb -s -P workloads/workloada -p rocksdb.dir=/mnt/test_ssd/ycsb-rocksdb-data
./bin/ycsb run rocksdb -s -P workloads/workloada -p rocksdb.dir=/mnt/test_ssd/ycsb-rocksdb-data
```

* 其中`rocksdb.dir`用需要测的盘目录替代

* `workloads/workloada`可以换成`workloads/workloadb`等(a-f),注意参数需要修改自定义，默认参数是1000
