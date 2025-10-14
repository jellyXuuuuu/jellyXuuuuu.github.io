---
layout: post
title: 分布式存储Pureflash(conductor)问题分析
tags: [Pureflash, conductor, 分布式存储]
---


## 任务(2025.10.13)
### 分析为什么数据库里面alloc_size出现大于total_size情况、free_size为负数；分析代码alloc size是怎么来的-shard_size的计算流程分析


根据代码函数`select_suitable_store_tray`内对于`v_store_free_size`的查询，可见conductor部分仅用这部分数据库访问select筛选剩余可用空间，用来分配存储节点。
```java
if(hostId == -1){
  list = S5Database.getInstance()
      .sql("select * from v_store_free_size as s "
          + "where s.status='OK' "
          + " order by free_size desc limit ? ", replica_count ).transaction(trans)
      .results(HashMap.class);

} else {
  if (replica_count > 1 ) {
    list = S5Database.getInstance()
        .sql("select * from v_store_free_size as s "
            + "where s.status='OK' and store_id!=?"
            + " order by free_size desc limit ? ", hostId, replica_count - 1).transaction(trans)
        .results(HashMap.class);
  }
  List<HashMap> list2 = S5Database.getInstance()
      .sql("select * from v_store_free_size as s "
          + "where s.status='OK' "
          + " and store_id=? ", hostId ).transaction(trans)
      .results(HashMap.class);
  if(list2 == null || list2.size() == 0){
    throw new InvalidParamException("Can't find specified store ID: " + hostId);
  }
  list.add(0, list2.get(0));
}
```

- 根据文件`jconductor/res/init_s5metadb.sql`里面的创建`v_store_alloc_size`表的语句
```sql
create view v_store_alloc_size as  select store_id, sum(t_volume.shard_size) as alloc_size from t_volume, t_replica where t_volume.id=t_replica.volume_id group by t_replica.store_id;
/* v_store_alloc_size即 */
select store_id, sum(t_volume.shard_size) as alloc_size from t_volume, t_replica where t_volume.id=t_replica.volume_id group by t_replica.store_id;
```
v_store_alloc_size是从t_volume、t_replica两个表，是volume里面的shard_size的总和


** 注: 这里docker的代码和conductor代码不同，docker用的是volume的size总和
```sql
create view v_store_alloc_size as  select store_id, sum(size) as alloc_size from t_volume, t_replica where t_volume.id=t_replica.volume_id group by t_replica.store_id;
```

- 创建`v_tray_alloc_size`语句是
```sql
select  t_replica.store_id as store_id, tray_uuid, sum(t_volume.shard_size) as alloc_size from t_volume, t_replica where t_volume.id = t_replica.volume_id group by t_replica.tray_uuid
, t_replica.store_id;
```
可以看到这里其实`tray_alloc_size`是`shard_size`的总和

根据代码`/home/flyslice/yangxiao/cocalele/PureFlash/pfs/src/pf_flash_store.cpp`中的以下语句，可以看到这里在`recovery_replica`store存储空间时pfs采取的是'前`N-1`个分片均使用标准大小(64G), 最后一个分片可能因总容量不是标准大小的整数倍, 而取`剩余空间`作为其大小，确保所有分片的总容量之和等于存储卷的总容量'.

```cpp
int64_t shard_size = std::min<int64_t>(SHARD_SIZE, vol->size - rep_id.shard_index()*SHARD_SIZE);
```

```sql
MariaDB [s5]> select  t_replica.store_id as store_id, tray_uuid, sum(t_volume.shard_size) as alloc_size from t_volume, t_replica where t_volume.id = t_replica.volume_id group by t_replica.tray_uuid
, t_replica.store_id;
+----------+--------------------------------------+---------------+
| store_id | tray_uuid                            | alloc_size    |
+----------+--------------------------------------+---------------+
|        1 | 7054366f-c8f1-404f-925e-09bb04288df1 | 3092376453120 |
|        2 | 88c73032-9b6b-493c-b826-2fccb4e245dd | 2680059592704 |
|        2 | ae6582a5-fd11-446c-a8de-22eb7a8a6540 | 1168231104512 |
|        3 | b4f726d1-fffb-444a-89f9-814314acc680 | 2336462209024 |
|        1 | d46569d5-a82a-47a3-b34d-608c8afb5e06 | 3092376453120 |
+----------+--------------------------------------+---------------+
5 rows in set (0.001 sec)

MariaDB [s5]> select * from v_tray_alloc_size;
+----------+--------------------------------------+---------------+
| store_id | tray_uuid                            | alloc_size    |
+----------+--------------------------------------+---------------+
|        1 | 7054366f-c8f1-404f-925e-09bb04288df1 | 3092376453120 |
|        2 | 88c73032-9b6b-493c-b826-2fccb4e245dd | 2680059592704 |
|        2 | ae6582a5-fd11-446c-a8de-22eb7a8a6540 | 1168231104512 |
|        3 | b4f726d1-fffb-444a-89f9-814314acc680 | 2336462209024 |
|        1 | d46569d5-a82a-47a3-b34d-608c8afb5e06 | 3092376453120 |
+----------+--------------------------------------+---------------+
```

查看当前环境的`v_tray_total_size`结果如下:
```sql
MariaDB [s5]> select * from v_tray_total_size where status = 'OK';
+----------+--------------------------------------+---------------+--------+
| store_id | tray_uuid                            | total_size    | status |
+----------+--------------------------------------+---------------+--------+
|        1 | 7054366f-c8f1-404f-925e-09bb04288df1 | 8001563222016 | OK     |
|        2 | 88c73032-9b6b-493c-b826-2fccb4e245dd | 2048408248320 | OK     |
|        2 | ae6582a5-fd11-446c-a8de-22eb7a8a6540 |  500107862016 | OK     |
|        3 | b4f726d1-fffb-444a-89f9-814314acc680 | 1000204886016 | OK     |
|        1 | cd7d26e6-9d99-4ea9-b31d-bc57b2a6c43c | 2048408248320 | OK     |
|        1 | d46569d5-a82a-47a3-b34d-608c8afb5e06 | 8001563222016 | OK     |
+----------+--------------------------------------+---------------+--------+
```
可见上表的第1235条是已用的盘(作为存储节点),可以看到目前问题就是总的大小比已分配的还小

- 创建`v_tray_total_size`语句是
```sql
select store_id, uuid as tray_uuid, raw_capacity as total_size, status from t_tray;
```

通过上述sql语句可以看出这里是把raw_capacity直接作为总的size大小的

根据以下代码，`jconductor/src/com/netbric/s5/conductor/handler/StoreHandler.java`下，在函数`add_storenode`中，定义一个新的tray时，设置他的`raw_capacity`为`8T`
```java
Tray t = new Tray();
		for (int i = 0; i < 20; ++i)
		{

			t.device = "Tray-" + i;
			t.status = Status.OK;
			t.raw_capacity = 8L << 40;
			t.store_id = n.id;
			S5Database.getInstance().insert(t);
		}
```

在`/home/flyslice/yangxiao/cocalele/jconductor/src/com/netbric/s5/cluster/ClusterManager.java`下的`updateStoreTrays`函数中，这里更新tray的`raw_capacity`用以下语句
```java
tr.raw_capacity = Long.parseLong(new String(zk.getData(zkBaseDir + "/stores/"+store_id+"/trays/"+t+"/capacity", false, null)));
```
根据上下文这里应该是根据实际的硬盘的容量来定义的
tr.raw_capacity的具体值完全取决于ZooKeeper对应节点中存储的字符串数值，也就是leader conductor节点上的值。(?)

根据`ps -ef | grep zookeeper`可以看到环境启动zookeeper的log在`/opt/apache-zookeeper-3.7.2-bin/bin/../logs`下的`zookeeper-root-server-node1.out`文件，配置文件在`/opt/apache-zookeeper-3.7.2-bin/bin/../conf/zoo.cfg`

```bash
flyslice@node1:~/yangxiao$ cat /opt/apache-zookeeper-3.7.2-bin/bin/../conf/zoo.cfg
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
server.1=192.168.61.229:2888:3888
server.2=192.168.61.143:2888:3888
server.3=192.168.61.122:2888:3888
```
不过ai说这个配置是zookeeper自身的运行配置，容量数据tray相关存储应该在zookeeper集群的节点路径中




### 分析各个table/view表格的作用

[数据库tables UML](\assets\数据库uml.png)
[view UML](\assets\viewuml.png)

* t_tray: 存储盘的信息，比如配置文件里面可以看到相关`tray.0`,`tray.1`等的配置，就是对应环境上的nvme路径
```bash
flyslice@node1:~/yangxiao$ cat /etc/pureflash/pfs.conf
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
dev=/dev/nvme1n1
[tray.1]
dev=/dev/nvme3n1
[tray.2]
dev=/dev/nvme4n1
[port.0]
ip=192.168.61.229
[rep_port.0]
ip=192.168.61.229
[tcp_server]
poller_count=8
[replicator]
conn_type=tcp
count=4
```

* t_port: 存储ip的信息，各个存储节点ip具体地址、状态以及store_id

* t_replica: 存储副本信息

* t_quotaset: 管理租户资源, car_id

* t_tenant: 租户信息

* t_shard: 存储切片信息

* t_volume: 存储卷信息

* t_snapshot: 存储快照信息

* t_shared_disk: 共享表(单独特性)

* seq_gen: 序列表，是`MariaDB`数据库中用于管理序列`(Sequence)`的系统表

| 从表结构来看，各字段的含义和用途如下:
| next_not_cached_value: 下一个未被缓存的序列值，用于记录序列缓存之外的下一个可用值。
| minimum_value: 序列的最小值，定义了序列可生成的最小数值。
| maximum_value: 序列的最大值，定义了序列可生成的最大数值。
| start_value: 序列的起始值，序列从该值开始生成。
| increment: 步长，每次生成下一个序列值时增加的数值（通常为 1）。
| cache_size: 缓存大小，指定预先缓存的序列值数量，用于提高序列生成性能。
| cycle_option: 是否循环（1 表示循环，0 表示不循环），当序列达到最大值时，若开启循环则从最小值重新开始。
| cycle_count: 循环次数，记录序列从最大值循环到最小值的次数。


** 总之，这个序列用于为表中的列(主键PRI KEY)提供自动生成的唯一标识符

* v_id: 所有租户(tenant)/卷(volume)/配额quataset内的id集合(union)
```sql
select id from t_tenant union all select id from t_volume union all select id from t_quotaset;
```

* v_replica_ext: 
关联5张表: t_volume/t_shard/t_replica/t_tenant/t_port通过volume_id/shard_id/replica_id/tenant_id/store_id关联整合成一个表
(view不实际存储数据，是从tables拉过来的数据整合成的新的表格)

* v_store_alloc_size/v_store_free_size/v_store_total_size:
v_store_free_size定义
```sql
select store_id, sum(t_volume.shard_size) as alloc_size from t_volume, t_replica where t_volume.id=t_replica.volume_id group by t_replica.store_id;
```
![251013-image2](\assets\251013-image2.png)

** 这里`alloc_size`的问题见前一个问题分析

在VolumeHandler.java里面判断剩余空间，用于给每个shard的每个副本分配节点的条件判断

* v_tray_alloc_size/v_tray_free_size/v_tray_total_size:
其中tray相关的空间用于选择存储节点策略，store相关的空间则是在判断总剩余容量时使用*
```sql
select  t_replica.store_id as store_id, tray_uuid, sum(t_volume.shard_size) as alloc_size from t_volume, t_replica where t_volume.id = t_replica.volume_id group by t_replica.tray_uuid
, t_replica.store_id;
```
![251013-image1](\assets\251013-image1.png)

