---
layout: post
title: zookeeper命令
tags: [Pureflash, zookeeper, 分布式存储]
---

## 怎么使用zookeeper

* 连接进入zk服务
```bash
# /opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh -server 127.0.0.1:2181
/opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh
# 默认连接到localhost:2181
```
是一个java脚本，目录在`/opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh`, zk的`log`写入了`/opt/apache-zookeeper-3.7.2-bin/logs/zookeeper-root-server-node1.out`


* 命令合集

*** 详情参加`zookeeperCLI`笔记

```bash
ZooKeeper -server host:port -client-configuration properties-file cmd args
	addWatch [-m mode] path # optional mode is one of [PERSISTENT, PERSISTENT_RECURSIVE] - default is PERSISTENT_RECURSIVE
	addauth scheme auth
	close 
	config [-c] [-w] [-s]
	connect host:port
	create [-s] [-e] [-c] [-t ttl] path [data] [acl]
	delete [-v version] path
	deleteall path [-b batch size]
	delquota [-n|-b|-N|-B] path
	get [-s] [-w] path
	getAcl [-s] path
	getAllChildrenNumber path
	getEphemerals path
	history 
	listquota path
	ls [-s] [-w] [-R] path
	printwatches on|off
	quit 
	reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
	redo cmdno
	removewatches path [-c|-d|-a] [-l]
	set [-s] [-v version] path data
	setAcl [-s] [-v version] [-R] path acl
	setquota -n|-b|-N|-B val path
	stat [-w] path
	sync path
	version 
	whoami
```

`ZooKeeper` 通常存储实时状态数据，如在线 / 离线、心跳），而数据库（如 MariaDB）存储持久化元数据（如总容量、历史分配记录）。两者通过定时同步或事件通知保持一致。

### 命令尝试

#### 连接/close
因为一使用脚本`zkCli.sh`就默认连接本地的2181端口，可以尝试端口`close`,重连。
```zk
close

WATCHER::

WatchedEvent state:Closed type:None path:null
2025-10-21 20:02:45,379 [myid:] - INFO  [main:ZooKeeper@1232] - Session: 0x100040fcf6e06de closed
[zk: 127.0.0.1:2181(CLOSED) 39] 2025-10-21 20:02:45,379 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@568] - EventThread shut down for session: 0x100040fcf6e06de

```

```zk
[zk: 127.0.0.1:2181(CLOSED) 39] connect 127.0.0.1:2181
2025-10-21 20:03:09,387 [myid:] - INFO  [main:ZooKeeper@637] - Initiating client connection, connectString=127.0.0.1:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@6f195bc3
2025-10-21 20:03:09,388 [myid:] - INFO  [main:ClientCnxnSocket@239] - jute.maxbuffer value is 1048575 Bytes
2025-10-21 20:03:09,388 [myid:] - INFO  [main:ClientCnxn@1741] - zookeeper.request.timeout value is 0. feature enabled=false
[zk: 127.0.0.1:2181(CONNECTING) 40] 2025-10-21 20:03:09,391 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1177] - Opening socket connection to server localhost/127.0.0.1:2181.
2025-10-21 20:03:09,391 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1179] - SASL config status: Will not attempt to authenticate using SASL (unknown error)
2025-10-21 20:03:09,392 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1011] - Socket connection established, initiating session, client: /127.0.0.1:45320, server: localhost/127.0.0.1:2181
2025-10-21 20:03:09,399 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1452] - Session establishment complete on server localhost/127.0.0.1:2181, session id = 0x100040fcf6e06e2, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
```

根据`PureFlash/thirdParty/zookeeper/zookeeper-contrib/zookeeper-contrib-huebrowser/zkui/src/zkui/stats.py`里`ZooKeeperStats`的定义，这里默认`init`以`localhost:2181`连接
![251022-image1](\assets\251022-image1.png)

#### 查看当前所用的zkcli版本
```zk
[zk: 127.0.0.1:2181(CONNECTED) 40] version
ZooKeeper CLI version: 3.7.2-c06c7c8a3e95779d4becb1938b378596e3b420d0, built on 2023-10-06 09:51 UTC
```

#### 查看zk里的conductor

![251021-image5](\assets\251021-image5.png)


#### 监听addWatch/removewatches
*addWatch*


增加监听者，用于注册持久化监听的命令，用于解决原生 `ZooKeeper` 监听 “一次性触发” 的问题。
- `PERSISTENT`：持久化监听。对目标节点的数据变化（NodeDataChanged）和节点删除（NodeDeleted）事件持续监听，触发后不会失效（无需重新注册）。但不监听子节点变化。
- `PERSISTENT_RECURSIVE`：持久化递归监听（默认模式）。不仅监听目标节点本身的变化，还会递归监听所有子节点的创建、删除、数据更新事件，且所有监听都是持久化的（触发后不失效）。

- `path`：需要监听的节点路径（必须是已存在的节点，否则会报错）。

```java
zkHelper.watchNode(zkBaseDir + "/stores/"+store_id+"/trays/"+t+"/online", new ZkHelper.NodeChangeCallback() {
	@Override
	void onNodeCreate(String childPath) {
		logger.info("ssd online, {}", childPath);
		S5Database.getInstance().sql("update t_tray set status=? where uuid=?", Status.OK, t).execute();
	}

	@Override
	void onNodeDelete(String childPath) {
		logger.info("ssd offline, {}", childPath);
		S5Database.getInstance().sql("update t_tray set status=? where uuid=?", Status.OFFLINE, t).execute();

	}
});
```
这里的逻辑是检测当zk增加或删除node同步更新数据库，但是会留着记录。

```java
String trayOnZk = zkBaseDir + "/stores/"+store_id+"/trays";

logger.info("watchNewChild on: {} ", trayOnZk);
zkHelper.watchNewChild(trayOnZk, new ZkHelper.NewChildCallback() {
	@Override
	void onNewChild(String childPath) {
		logger.info("New disk found in zk:{}", childPath);
		updateStoreTrays(store_id);
	}
});
```
这里是检测当前节点下有多的tray，就增加用`updateStoreTrays`方法


*removewatches*


这个应该与`addWatch`对应的，删除操作。



### zookeeper代码结构
- `PureFlash/thirdParty/zookeeper/zookeeper-server/src/main/java/org/apache/zookeeper/ZooKeeperMain.java`: 看起来是zoocli的主函数，就是用`/opt/apache-zookeeper-3.7.2-bin/bin/zkCli.sh`启动的client
climain: `PureFlash/thirdParty/zookeeper/zookeeper-server/src/main/java/org/apache/zookeeper/cli/CliCommand.java`



