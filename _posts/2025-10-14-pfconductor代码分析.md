---
layout: post
title: pfconductor代码分析
tags: [Pureflash, conductor, 分布式存储]
---

## 目标/任务: 查看com/netbric/s5/conductor/handler/S5RestfulHandler.java，看看还有什么命令，如果pfccli没有实现，思考应该怎么实现

### S5RestfulHandler.java
这是一个服务接口，用于后端可以操作volume/shard/replica等

尝试增加新接口'list_post'
在`CliMian`文件里添加以下代码
```java
static void cmd_list_port(Namespace cmd, Config cfg) throws Exception {
		// ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", "192.168.61.143", ListNodePortReply.class);
		ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", ListNodePortReply.class);
		if(r.retCode == RetCode.OK)
		logger.info("Succeed list_store");
		else
			throw new IOException(String.format("Failed to list_port , code:%d, reason:%s", r.retCode, r.reason));
		String [] header = { "IP Address", "Store Id", "Status"};

		String[][] data = new String[r.Ports.size()][];
		for(int i=0;i<r.Ports.size();i++) {
			data[i] = new String[]{ r.Ports.get(i).ip_addr, Long.toString(r.Ports.get(i).store_id), r.Ports.get(i).status };
		}
		ASCIITable.getInstance().printTable(header, data);
	}
```

发现需要增加对于的java类`ListNodePortReply`
/home/flyslice/yangxiao/cocalele/jconductor/src/com/netbric/s5/conductor/rpc/ListNodePortReply.java

```java
package com.netbric.s5.conductor.rpc;

import com.netbric.s5.orm.Port;

import java.util.List;

public class ListNodePortReply extends RestfulReply {

    public List<Port> Ports;
    

    public ListNodePortReply(String op) {
        super(op);
    }

    public ListNodePortReply(String op, int retCode, String reason) {
        super(op, retCode, reason);
    }
    public ListNodePortReply(String op, List<Port> Ports) {
        super(op);
        this.Ports = Ports;
    }
}

```

目前这样的修改，编译可以通过
```bash
ant -f jconductor.xml
```
![251015-image1](\assets\251015-image1.png)
但是运行结果还有问题(10.15待解决):
```bash
root@node2:/home/flyslice/yangxiao/cocalele/jconductor# ./pfcli list_port
[main] ERROR com.netbric.s5.conductor.rpc.SimpleHttpRpc - Failed http GET http://192.168.61.229:49180/s5c/?op=list_port
java.io.IOException: Failed RPC invoke, code:2, reason:Invalid argument: node_name
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeGET(SimpleHttpRpc.java:60)
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeConductor(SimpleHttpRpc.java:90)
	at com.netbric.s5.cli.CliMain.cmd_list_port(CliMain.java:357)
	at com.netbric.s5.cli.CliMain$4.run(CliMain.java:134)
	at com.netbric.s5.cli.CliMain.main(CliMain.java:211)
[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:2, reason:Invalid argument: node_name
```

解决思路: 增加`node_name`参数
```java
ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", ListNodePortReply.class,
						"node_name", "192.168.61.143");
```
然后发现这里原本serve端的代码`jconductor/src/com/netbric/s5/conductor/handler/StoreHandler.java`中是查询`node_name`, 但目前环境上没有配, 所以尝试改成id(也可以改成ip, 但id是唯一标识机器的, 用id更加准确)


![251015-image2](\assets\251015-image2.png)
```sql
MariaDB [s5]> select * from t_store;
+----+------+------+-------+----------------+--------+
| id | name | sn   | model | mngt_ip        | status |
+----+------+------+-------+----------------+--------+
|  1 | NULL | NULL | NULL  | 192.168.61.229 | OK     |
|  2 | NULL | NULL | NULL  | 192.168.61.143 | OK     |
|  3 | NULL | NULL | NULL  | 192.168.61.122 | OK     |
+----+------+------+-------+----------------+--------+
```

- 增加`id`参数
```java
String id = cmd.getString("i");
ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", ListNodePortReply.class,
						"id", id);
```
- 对于cli参数
```bash
./pfcli list_port -i 1
```

发现修改serve `jconductor/src/com/netbric/s5/conductor/handler/StoreHandler.java`文件内容没有生效，怀疑是因为这个是pfc服务，应该重启pfconductor服务才能生效 - 尝试重启pfconductor serve端仍然报错
| 报错如下:
```bash
root@node2:/home/flyslice/yangxiao/cocalele/jconductor# ./pfcli list_port
[main] ERROR com.netbric.s5.conductor.rpc.SimpleHttpRpc - Failed http GET http://192.168.61.229:49180/s5c/?op=list_port&id=1
java.io.IOException: Failed RPC invoke, code:2, reason:Invalid argument: node_name
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeGET(SimpleHttpRpc.java:60)
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeConductor(SimpleHttpRpc.java:90)
	at com.netbric.s5.cli.CliMain.cmd_list_port(CliMain.java:357)
	at com.netbric.s5.cli.CliMain$4.run(CliMain.java:134)
	at com.netbric.s5.cli.CliMain.main(CliMain.java:211)
[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:2, reason:Invalid argument: node_name
```
* 启动 pfconductor

```bash
source /home/flyslice/yangxiao/cocalele/jconductor/env-pfc.sh
nohup pfc -c /etc/pureflash/pfc.conf &
```

| 目前问题: 这个pfcli报错一直是这个`[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:2, reason:Invalid argument: node_name`，但是我修改了代码里面唯一有包含字段"Invalid argument: node_name"的地方重新编译报错一直不变，找不到他是从哪里出来的;其实报错也可以看到执行的http命令已修改为传递id(因为climain里面修改了)，但看起来后端serve对于的地方还是没有修改。

*问题原因：没有在`leader conductor`服务器上修改*

更新serve端步骤:
1. 更新后端代码并重新编译
2. kill掉三台服务器上的conductor serve进程
3. 重新启动serve，现在229上执行，因为默认只有一个主leader，最先执行服务的是leader，若kill掉leader上的进程默认会跳到别的服务器
* 想要一直保持229作为主节点需要三台同时以上操作

例如如下操作:
```bash
root@node2:/home/flyslice/yangxiao/cocalele/jconductor# ps -ef | grep pfc
root     1168411 1128129  0 15:44 pts/0    00:00:01 java -classpath /home/flyslice/yangxiao/cocalele/jconductor/out/production/jconductor:/home/flyslice/yangxiao/cocalele/jconductor/lib/* -Dorg.slf4j.simpleLogger.showDateTime=true -Dorg.slf4j.simpleLogger.dateTimeFormat=[yyyy/MM/dd H:mm:ss.SSS] -XX:+HeapDumpOnOutOfMemoryError com.netbric.s5.conductor.Main -c /etc/pureflash/pfc.conf
root     1168483 1128129  0 15:59 pts/0    00:00:00 grep --color=auto pfc
kill 1168411

# 启动pfconductor:三个节点上分别执行
source /home/flyslice/yangxiao/cocalele/jconductor/env-pfc.sh
nohup pfc -c /etc/pureflash/pfc.conf &
```

- 更新到229后生效，出现新问题

(2025.10.16)

```bash
flyslice@node1:~/yangxiao/cocalele/jconductor$ ./pfcli list_port -i 1
cmd_list_port
[main] ERROR com.netbric.s5.conductor.rpc.SimpleHttpRpc - Failed http GET http://192.168.61.229:49180/s5c/?op=list_port&id=1
java.io.IOException: Failed RPC invoke, code:4, reason:Cannot run program "c:/eclipse/plink.exe" (in directory "."): error=2, 没有那个文件或目录
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeGET(SimpleHttpRpc.java:60)
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeConductor(SimpleHttpRpc.java:90)
	at com.netbric.s5.cli.CliMain.cmd_list_port(CliMain.java:359)
	at com.netbric.s5.cli.CliMain$4.run(CliMain.java:135)
	at com.netbric.s5.cli.CliMain.main(CliMain.java:212)
[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:4, reason:Cannot run program "c:/eclipse/plink.exe" (in directory "."): error=2, 没有那个文件或目录

```



### handler下的StoreHandler.java/TenantHandler.java/VolumeHandler.java
`StoreHandler.java`的注释里写了`backend handler of CLI s5_add_store_node.py`，应该是基于原本的`s5_add_store_node.py`文件改造的，是后端操作对于`storenode`增删检查的
同理，`TenantHandler.java`针对tenants, `VolumeHandler`针对volume

### Recovery流程的处理
Recovery的处理是：
```bash
./pfcli recovery_volume -v test_v1
```
针对`test_v1 volume`查询状态不是`OK`的shard
遍历shard中的replica，对于状态不是`OK`的slave replica
给这个slave replica所在机器的pfstore发送recovery_replica，让其从primary拷贝数据恢复数据该slave replica
