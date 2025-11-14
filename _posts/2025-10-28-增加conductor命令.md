---
layout: post
title: 补充conductor命令
tags: [Pureflash, conductor, 分布式存储]
---

## 目标/任务: 查看com/netbric/s5/conductor/handler/S5RestfulHandler.java，看看还有什么命令，如果pfccli没有实现，思考应该怎么实现

### 补充conductor命令1
任务1: 新增`list_port`

* S5RestfulHandler.java
这是一个服务接口，用于后端可以操作volume/shard/replica等

尝试增加新接口'list_post'
在`CliMian`文件里添加以下代码
```java
static void cmd_list_port(Namespace cmd, Config cfg) throws Exception {
		// ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", "192.168.61.143", ListNodePortReply.class);
		ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", ListNodePortReply.class);
		if(r.retCode == RetCode.OK)
		logger.info("Succeed list_port");
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

修改`src/com/netbric/s5/conductor/SshExec.java`下的ssh路径
```java
// static String ssh_bin = "c:/eclipse/plink.exe";
static String ssh_bin = "ssh"; ///home/flyslice/yangxiao/cocalele/jconductor/res/plink.exe
static Map<String, String> env = new HashMap<String, String>();
...
```

报错如下：
```bash
root@node1:/home/flyslice/yangxiao/cocalele/jconductor# ./pfcli list_port -i 1
cmd_list_port
id:1
[main] ERROR com.netbric.s5.conductor.rpc.SimpleHttpRpc - Failed http GET http://192.168.61.229:49180/s5c/?op=list_port&id=1
java.io.IOException: Failed RPC invoke, code:5, reason:Bad port 'w'

	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeGET(SimpleHttpRpc.java:60)
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeConductor(SimpleHttpRpc.java:90)
	at com.netbric.s5.cli.CliMain.cmd_list_port(CliMain.java:360)
	at com.netbric.s5.cli.CliMain$4.run(CliMain.java:135)
	at com.netbric.s5.cli.CliMain.main(CliMain.java:212)
[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:5, reason:Bad port 'w'

```


- 目前卡住不知道在哪里** - 原因是ssh参数错误
策略：重新下载一个虚拟机模拟环境测试。先放弃这个case，尝试别的case


* 客户端调用路径
```
CliMain.main() → CliMain$4.run() → CliMain.cmd_list_port() → SimpleHttpRpc.invokeConductor() → SimpleHttpRpc.invokeGET()
```

* 251029: 引入`StoreHandler`，可以不用自己创建一个新的相关reply，都已经定义了
#### 更新修改如下
* `CliMain`:

```java
import com.netbric.s5.conductor.handler.StoreHandler.*;

sp=sps.addParser("list_port");
sp.description("List port info");
sp.addArgument("-i").help("Node Id name to check the port").required(true).metavar("node_id");
sp.addArgument("-p").help("port to check the port").required(true).metavar("port_id");
sp.setDefault("__func", new CmdRunner() {
	@Override
	public void run(Namespace cmd, Config cfg) throws Exception {
		cmd_list_port(cmd, cfg);
	}
});

static void cmd_list_port(Namespace cmd, Config cfg) throws Exception {
	System.out.println("cmd_list_port");
	String id = cmd.getString("i");
	String port = cmd.getString("p");
	System.out.println("id:" + id + ", port: " + port);
	ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", ListNodePortReply.class,
					"id", id, "port", port);
	if(r.retCode == RetCode.OK)
	logger.info("Succeed list_port");
	else
		throw new IOException(String.format("Failed to list_port , code:%d, reason:%s", r.retCode, r.reason));
	String [] header = { "IP Address", "Store Id", "Status"};

	// String[][] data = new String[r.ports.size()][];
	// for(int i=0;i<r.ports.size();i++) {
	// 	data[i] = new String[]{ r.ports.get(i).ip_addr, Long.toString(r.ports.get(i).store_id), r.ports.get(i).status };
	// }
	// ASCIITable.getInstance().printTable(header, data);
	String[][] data = new String[r.ports.length][];
	for (int i = 0; i < r.ports.length; i++) {
		data[i] = new String[]{ 
			r.ports[i].ip_addr, 
			Long.toString(r.ports[i].store_id), 
			r.ports[i].status 
		};
	}
	ASCIITable.getInstance().printTable(header, data);
}

```

* 对应serve端目前改动：
`StoreHandler`:
```java
/* 这里原本已经定义了 ListNodePortReply 
   因此可以直接使用 */
public static class ListNodePortReply extends RestfulReply
    {
        public Port[] ports;
        public ListNodePortReply(String op)
        {
            super(op);
        }
    }
public RestfulReply list_nodeport(Request request, Response response)
{
	String op = request.getParameter("op");
	String hostid = request.getParameter("id"); //改成了id，原本是hostname
	System.out.println("list_nodeport:" + hostid);
	if (StringUtils.isEmpty(hostid))
		return new RestfulReply(op, RetCode.INVALID_ARG, "Invalid argument: id");
	try
	{
		StoreNode node = S5Database.getInstance().where("id=?", hostid).first(StoreNode.class);  //改成了id，原本查找name
		...
	}
}
```

* ssh修改, `SshExec`:

```java
// static String ssh_bin = "c:/eclipse/plink.exe";
static String ssh_bin = "ssh"; ///home/flyslice/yangxiao/cocalele/jconductor/res/plink.exe
...
```

#### 251030 问题更新
发现`Bad port 'w'`问题出处：
直接执行`ssh`可以发现
```bash
root@node1:~# ssh root@192.168.61.229 -pw Flysl1ce
Bad port 'w'
```
因此是参数`-pw`的问题

查看`SshExec`定义`execute`方法可得
```java
CommandLine cl = new CommandLine(ssh_bin);
	cl.addArgument("-T");
	// cl.addArgument("-n"); //use ssh
	cl.addArgument("-pw"); // plink
	// cl.addArgument("123456"); // plink
	cl.addArgument("Flysl1ce");
	cl.addArgument("-l");
	cl.addArgument("root");
// 这里原本定义的是根据plink - Windows上用ssh方法 -需要改成ssh对应参数
```

| 这里找到"Bad port w"的错误原因了，是ssh不支持加了参数-pw导致的因为他这里用的是plink，我前面改成了ssh，但是ssh没办法直接登录，所以在系统上下载一个plink就可以了

![251030-image2](\assets\251030-image2.png)
 
此时遇到新错误

```
root@node1:/home/flyslice/yangxiao/cocalele/jconductor# ./pfcli list_port -i 1
cmd_list_port
id:1
[main] ERROR com.netbric.s5.conductor.rpc.SimpleHttpRpc - Failed http GET http://192.168.61.229:49180/s5c/?op=list_port&id=1
java.io.IOException: Failed RPC invoke, code:5, reason:bash: 行 1: source /etc/profile; echo : 没有那个文件或目录

	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeGET(SimpleHttpRpc.java:60)
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeConductor(SimpleHttpRpc.java:90)
	at com.netbric.s5.cli.CliMain.cmd_list_port(CliMain.java:510)
	at com.netbric.s5.cli.CliMain$4.run(CliMain.java:141)
	at com.netbric.s5.cli.CliMain.main(CliMain.java:272)
[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:5, reason:bash: 行 1: source /etc/profile; echo : 没有那个文件或目录

```

| 发现修改`addArgument`的参数(`CommandLine`接口)可以恢复命令问题

```java
// cl.addArgument(remoteCmd);
cl.addArgument(remoteCmd, false);  // 取消自动分配的双引号，禁止自动添加引号
```

注：ssh需要事先存key，不然会 报错
```bash
The server's host key is not cached. You have no guarantee that the server is the computer you think it is. The server's ssh-ed25519 key fingerprint is: ssh-ed25519 255 SHA256:xMa0mYxFZ3SyZPFyaPLbBR+K9bNHMQhr8dIK5EpnG0o If you trust this host, enter "y" to add the key to PuTTY's cache and carry on connecting. If you want to carry on connecting just once, without adding the key to the cache, enter "n". If you do not trust this host, press Return to abandon the connection. Store key in cache? (y/n, Return cancels connection, i for more info) Connection abandoned.
```

```
plink -pw Flysl1ce -l root 192.168.61.229
plink -pw Flysl1ce -l root 192.168.61.122
plink -pw flyslice -l root 192.168.61.143
```


#### 最终效果(成功返回值)

| 目前只支持在本机上连自己获得这个信息，连别的会卡住

效果如下：

![251031-image1](\assets\251031-image1.png)

目前在229服务器上仅支持`./pfcli list_port -i 1`

日志(含调试打印信息)

![251031-image2](\assets\251031-image2.png)

#### 最终代码

```java

/* CliMain.java */

static void cmd_list_port(Namespace cmd, Config cfg) throws Exception {
	System.out.println("cmd_list_port");
	String id = cmd.getString("i");
	System.out.println("id:" + id);
	ListNodePortReply r = SimpleHttpRpc.invokeConductor(cfg, "list_port", ListNodePortReply.class,
					"id", id);
	if(r.retCode == RetCode.OK)
	logger.info("Succeed list_port");
	else
		throw new IOException(String.format("Failed to list_port , code:%d, reason:%s", r.retCode, r.reason));
	String [] header = { "Name", "IP Address", "Store Id", "Purpose"}; // "Status", 

	// String[][] data = new String[r.ports.size()][];
	// for(int i=0;i<r.ports.size();i++) {
	// 	data[i] = new String[]{ r.ports.get(i).ip_addr, Long.toString(r.ports.get(i).store_id), r.ports.get(i).status };
	// }
	// ASCIITable.getInstance().printTable(header, data);
	String[][] data = new String[r.ports.length][];
	for (int i = 0; i < r.ports.length; i++) {
		data[i] = new String[]{ 
			r.ports[i].name, 
			r.ports[i].ip_addr, 
			Long.toString(r.ports[i].store_id), 
			Long.toString(r.ports[i].purpose), 
			// r.ports[i].status 
		};
	}
	ASCIITable.getInstance().printTable(header, data);
}

```

`src/com/netbric/s5/conductor/handler/StoreHandler.java`更新见: [storevolume-update.txt](\assets\storevolume-update.txt)




### 补充conductor命令2
尝试增加`open_volume`/ `open_aof`
直接用`volume_handler`里面的`open_volume`方法，直接引用。没有输出。

`CliMain.java`: 
```java
sp=sps.addParser("open_aof");
sp.description("Open/prepare volume");
sp.addArgument("-v").help("Aof name to open").required(true).metavar("volume_name");
sp.setDefault("__func", new CmdRunner() {
	@Override
	public void run(Namespace cmd, Config cfg) throws Exception {
		cmd_open_volume(cmd, cfg);
	}
});

sp=sps.addParser("open_volume");
sp.description("Open/prepare volume");
sp.addArgument("-v").help("Volume name to open").required(true).metavar("volume_name");			sp.setDefault("__func", new CmdRunner() {
	@Override
	public void run(Namespace cmd, Config cfg) throws Exception {
		cmd_open_volume(cmd, cfg);
	}
});


private static void cmd_open_volume(Namespace cmd, Config cfg) throws Exception {
	String volumeName = cmd.getString("v");
	OpenVolumeReply r = SimpleHttpRpc.invokeConductor(cfg, "open_volume", OpenVolumeReply.class, "volume_name", volumeName);
	if(r.retCode == RetCode.OK)
		logger.info("Succeed open_volume");
	else
		throw new IOException(String.format("Failed to open_volume , code:%d, reason:%s", r.retCode, r.reason));
	String [] header = { "Id", "Name", "Size", "RepCount", "Status"};

	System.out.printf("cmd_open_volume here. \n");

}

```

修改：将引用的方法由`open_volume`变成`getVolumeInfoForClient`
-- 不对，因为`getVolumeInfoForClient`是`open_volume`下的接口，以上更改即可。

返回日志见[open_volume_test_v2.log](\assets\\open_volume_test_v2.log)

建议修改：增加返回值，参考`list_volume`

更新`cmd_open_volume`代码如下：
```java
private static void cmd_open_volume(Namespace cmd, Config cfg) throws Exception {
	String volumeName = cmd.getString("v");
	OpenVolumeReply r = SimpleHttpRpc.invokeConductor(cfg, "open_volume", OpenVolumeReply.class, "volume_name", volumeName);
	if(r.retCode == RetCode.OK)
		logger.info("Succeed open_volume");
	else
		throw new IOException(String.format("Failed to open_volume , code:%d, reason:%s", r.retCode, r.reason));
	// String [] header = { "Id", "Name", "Size", "RepCount", "Status"};
	String[] shardHeader = {
		"Shard Index",    // 分片索引
		"Status",         // 状态
		"Store IPs",      // 存储节点IP
		"Is Shared Disk", // 是否共享磁盘
		"Disk UUID",      // 磁盘UUID（共享磁盘时有效）
		"Device Name"     // 设备名称（共享磁盘时有效）
	};

	System.out.printf("cmd_open_volume here. \n");
	if (r.shards == null || r.shards.length == 0) {
		System.out.println("No shard data available.");
	} else {
		// 构建表格数据
		String[][] shardData = new String[r.shards.length][];
		for (int i = 0; i < r.shards.length; i++) {
			ShardInfoForClient shard = r.shards[i];
			shardData[i] = new String[]{
				String.valueOf(shard.index),                  // 分片索引
				shard.status,                                 // 状态
				shard.store_ips != null ? shard.store_ips : "", // 存储IP（为空时显示空字符串）
				shard.is_shareddisk == 1 ? "Yes" : "No",      // 是否共享磁盘（转换为Yes/No）
				shard.disk_uuid != null ? shard.disk_uuid : "", // 磁盘UUID
				shard.dev_name != null ? shard.dev_name : ""   // 设备名称
			};
		}
		// 打印分片信息表格
		System.out.println("\n=== Shard Information Table ===");
		ASCIITable.getInstance().printTable(shardHeader, shardData);
	}

	String[] volumeHeader = {
		"Volume ID", "Volume Name", "Size (Bytes)", "Replica Count", "Status", "Meta Version", "Snapshot Seq"
	};
	String[][] volumeData = {
		{
			String.valueOf(r.volume_id),
			r.volume_name,
			String.valueOf(r.volume_size),
			String.valueOf(r.rep_count),
			r.status,
			String.valueOf(r.meta_ver),
			String.valueOf(r.snap_seq)
		}
	};
	System.out.println("\n=== Volume Basic Information ===");
	ASCIITable.getInstance().printTable(volumeHeader, volumeData);
}

```

有输出结果了，返回：
![251029-image1](\assets\251029-image1.png)


### 补充conductor命令3
尝试增加`node_sanity_check`

前端 `CliMain`:
```java
import com.netbric.s5.conductor.handler.StoreHandler.*;

sp=sps.addParser("sanity_check");
sp.description("sanity check");
sp.addArgument("-i").help("Node Id name to check").required(true).metavar("node_id");
sp.setDefault("__func", new CmdRunner() {
	@Override
	public void run(Namespace cmd, Config cfg) throws Exception {
		cmd_sanity_check(cmd, cfg);
	}
});


private static void cmd_sanity_check(Namespace cmd, Config cfg) throws Exception {
	String id = cmd.getString("i");
	SanityCheckReply r = SimpleHttpRpc.invokeConductor(cfg, "node_sanity_check", SanityCheckReply.class, "id", id);
	if(r.retCode == RetCode.OK)
		logger.info("Succeed sanity_check");
	else
		throw new IOException(String.format("Failed to sanity_check , code:%d, reason:%s", r.retCode, r.reason));
}

```
后端serve `StoreHandler`:
```java
public RestfulReply sanity_check(Request request, Response response)
{
	String op = request.getParameter("op");
	String hostname = request.getParameter("id");  // hostname变为id,由前端定义
	if (StringUtils.isEmpty(hostname))
		return new RestfulReply(op, RetCode.INVALID_ARG, "Invalid argument: hostname");
	try
	{
		StoreNode node = S5Database.getInstance().where("id=?", hostname).first(StoreNode.class); //这里查找id，替代name
		if (node == null)
		...
	}
}
```

- 目前还是有报错，原因未知(修改serve端后)


```bash
root@node1:/home/flyslice/yangxiao/cocalele/jconductor# ./pfcli sanity_check -i 1
[main] ERROR com.netbric.s5.conductor.rpc.SimpleHttpRpc - Failed http GET http://192.168.61.229:49180/s5c/?op=node_sanity_check&id=1
java.io.IOException: Failed RPC invoke, code:5, reason:null
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeGET(SimpleHttpRpc.java:60)
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeConductor(SimpleHttpRpc.java:90)
	at com.netbric.s5.cli.CliMain.cmd_sanity_check(CliMain.java:414)
	at com.netbric.s5.cli.CliMain.access$200(CliMain.java:27)
	at com.netbric.s5.cli.CliMain$8.run(CliMain.java:178)
	at com.netbric.s5.cli.CliMain.main(CliMain.java:247)
[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:5, reason:null

```

查看日志输出如下:
```
[2025/10/31 10:43:37.759] [pool-1-thread-11] INFO com.netbric.s5.conductor.handler.S5RestfulHandler - API called: op=/s5c/?op=node_sanity_check&id=1
[2025/10/31 10:43:37.762] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; echo Hello
[2025/10/31 10:43:38.203] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; pidof lt-raio_server
[2025/10/31 10:43:38.658] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; pidof s5afs
[2025/10/31 10:43:39.108] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; pidof bdd
[2025/10/31 10:43:39.551] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; which nbdxadm
[2025/10/31 10:43:39.999] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; lsmod | grep nbdx
[2025/10/31 10:43:40.446] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; lsmod | grep s5bd
[2025/10/31 10:43:40.886] [pool-1-thread-11] INFO com.netbric.s5.conductor.SshExec - cmd after sshexec: source /etc/profile; which s5bd
[2025/10/31 10:43:41.298] [pool-1-thread-11] INFO com.netbric.s5.conductor.handler.S5RestfulHandler - {
  "results": {
    "s5bd": "FAILED: s5bd command not found",
    "s5afs": "FAILED: s5afs not running",
    "bdd": "FAILED: bdd not running",
    "nbdxadm": "FAILED: nbdxadm can not found",
    "ssh": "FAILED:bash: 行 1: echo Hello: 未找到命令\n",
    "nbdx.ko": "FAILED: nbdx.ko not installed",
    "raio_server": "FAILED: raio_server not running",
    "s5bd.ko": "FAILED: s5bd.ko not installed"
  },
  "op": "node_sanity_check_reply",
  "ret_code": 5
}

```
所以接口没问题，但是环境上缺少`s5bd`, `s5afs`, `bdd`, `nbdxadm`, `ssh`, `nbdx.ko`, `raio_server`, `s5bd.ko`.


| 修复`list_port`问题后ssh正常。其他进程都不存在


### 补充conductor命令4
尝试增加`expose_volume` / `unexpose_volume`

命令:
```bash
./pfcli expose_volume -t tenant_default -v test_v1
```

![251030-image1](\assets\251030-image1.png)

目前代码:
```java
/* CliMain.java */
sp=sps.addParser("expose_volume");
sp.description("expose volume");
sp.addArgument("-v").help("Volume name to expose").required(true).metavar("volume_name");
sp.addArgument("-t").help("Tenant name to expose").required(true).metavar("tenant_name");
sp.setDefault("__func", new CmdRunner() {
	@Override
	public void run(Namespace cmd, Config cfg) throws Exception {
		cmd_expose_volume(cmd, cfg);
	}
});

private static void cmd_expose_volume(Namespace cmd, Config cfg) throws Exception {
	String volumeName = cmd.getString("v");
	String tenant_name = cmd.getString("t");
	ExposeVolumeReply r = SimpleHttpRpc.invokeConductor(cfg, "expose_volume", ExposeVolumeReply.class, "volume_name", volumeName, "tenant_name", tenant_name);
	if(r.retCode == RetCode.OK)
		logger.info("Succeed expose_volume");
	else
		throw new IOException(String.format("Failed to expose_volume , code:%d, reason:%s", r.retCode, r.reason));

}



```



返回日志:

```bash
[2025/10/29 17:38:31.859] [pool-1-thread-5] INFO com.netbric.s5.conductor.handler.S5RestfulHandler - API called: op=/s5c/?op=expose_volume&volume_name=test_v1&tenant_name=tenant_default
[2025/10/29 17:38:31.889] [pool-1-thread-5] ERROR com.netbric.s5.conductor.NbdxServer - Execute command on: 192.168.61.229,[ssh, -T, -pw, 123456, -l, root, 192.168.61.229, "source /etc/profile; s5bd map --toe_ip 127.0.0.1 --toe_port 0 --volume_id 3539992576 --volume_size 921600 --dev_name s5r_1_test_v1"], stdout:Bad port 'w'

[2025/10/29 17:38:31.891] [pool-1-thread-5] INFO com.netbric.s5.conductor.handler.S5RestfulHandler - {
  "reason": "s5bd map failed on node:192.168.61.229\nBad port \u0027w\u0027\n",
  "op": "expose_volume_reply",
  "ret_code": 5
}
```


### 补充conductor命令5
尝试增加`update_volume`

```java
/* CliMain.java */
sp=sps.addParser("update_volume");
sp.description("Update volume");
sp.addArgument("-v").help("Volume name to update").required(true).metavar("volume_name");
sp.setDefault("__func", (CmdRunner) (cmd, cfg) ->{
		String volumeName = cmd.getString("v");

		CreateVolumeReply r = SimpleHttpRpc.invokeConductor(cfg, "update_volume", CreateVolumeReply.class, "volume_name", volumeName);
		if(r.retCode == RetCode.OK)
			logger.info("Succeed update volume:{}", volumeName);
		else
			throw new IOException(String.format("Failed to update volume:%s , code:%d, reason:%s", volumeName, r.retCode, r.reason));
	});

```

目前效果: 无返回值，但调用接口成功

命令参考: `./pfcli update_volume -v test_v4`

日志如下(含调试打印信息):

```
[2025/10/31 15:11:07.318] [pool-1-thread-1] INFO com.netbric.s5.conductor.handler.S5RestfulHandler - API called: op=/s5c/?op=update_volume&volume_name=test_v4
Update Volumetest_v4
[2025/10/31 15:11:07.457] [pool-1-thread-1] INFO com.netbric.s5.conductor.handler.S5RestfulHandler - {
  "op": "update_volume_reply",
  "ret_code": 0
}
```



### 补充conductor命令6
增加`list_tenant`





### 补充handle_error

遇到问题
```bash
root@flycd slice-Standard-PC-i440FX-PIIX-1996:/home/flyslice/yangxiao/cocalele/jconductor# ./pfcli handle_error -sc 0xC -i 1
[main] ERROR com.netbric.s5.conductor.rpc.SimpleHttpRpc - Failed http GET http://192.168.61.3:49180/s5c/?op=handle_error&replica_id=1&state_code=0xC
java.io.IOException: Failed RPC invoke, code:1, reason:Invalid argument:rep_id
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeGET(SimpleHttpRpc.java:60)
	at com.netbric.s5.conductor.rpc.SimpleHttpRpc.invokeConductor(SimpleHttpRpc.java:90)
	at com.netbric.s5.cli.CliMain.cmd_handle_error(CliMain.java:371)
	at com.netbric.s5.cli.CliMain.access$000(CliMain.java:28)
	at com.netbric.s5.cli.CliMain$1.run(CliMain.java:100)
	at com.netbric.s5.cli.CliMain.main(CliMain.java:351)
[main] ERROR com.netbric.s5.cli.CliMain - Failed: Failed RPC invoke, code:1, reason:Invalid argument:rep_id

```
修改输入格式 - `CliMain.java`
```java
sp = sps.addParser("handle_error");
sp.description("Handle error case when some replicas are in error state");
sp.addArgument("-i").help("Replica id").required(true).type(Integer.class).metavar("rep_id");
sp.addArgument("-sc").help("State code, 0xC0 - 0xCA, e.g.: 0xC0 means MSG_STATUS_NOT_PRIMARY").required(true).metavar("sc");
sp.setDefault("__func", new CmdRunner() {
	@Override
	public void run(Namespace cmd, Config cfg) throws Exception {
		cmd_handle_error(cmd, cfg);
	}
});

private static void cmd_handle_error(Namespace cmd, Config cfg) throws Exception {
	long repId = cmd.getInt("i");
	System.out.println("repId: " + repId);
	long stateCode = Long.decode(cmd.getString("sc"));
	System.out.println("stateCode: " + stateCode);
	
	RestfulReply r = SimpleHttpRpc.invokeConductor(cfg, "handle_error", RestfulReply.class,
			"rep_id", repId,
			"sc", stateCode);
	if(r.retCode == RetCode.OK)
		logger.info("Succeed handle_error for replica id:{}", repId);
	else
		throw new IOException(String.format("Failed to handle_error for replica id:%d , code:%d, reason:%s", repId, r.retCode, r.reason));
}
```

### add_store/delete_store不开发了，因为环境集群没加，只加元数据没意义。
