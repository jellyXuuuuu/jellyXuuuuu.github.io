---
layout: post
title: 分布式存储Pureflash(conductor)问题分析
tags: [Pureflash, conductor, 分布式存储]
---


## 任务(2025.10.13)
### 分析为什么数据库里面alloc_size出现大于total_size情况、free_size为负数；分析代码alloc size是怎么来的-shard_size的计算流程分析
见`post`: *Pureflash空间计算问题分析*



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

#### 查询`test_v1`卷下所有分片相关信息
```sql
select v.name volume_name, r.volume_id volume_id, d.id shard_id, d.shard_index shard_index, r.replica_id replica_id, r.replica_index replica_index, mngt_ip store_ip, is_primary, s.id store_id, 
r.status rstatus, v.status vstatus, s.status sstatus, d.status dstatus  
from 
v_replica_ext r, t_volume v,t_store s, t_tray t, t_shard d 
where r.store_id=s.id and t.uuid=r.tray_uuid and d.volume_id=r.volume_id and d.volume_id=v.id and d.id=r.shard_id and v.name='test_v1'   
ORDER BY shard_id ASC;
```
这条 SQL 主要用于查询名为 test_v1 的卷下所有分片的相关信息，包括每个分片的各个副本分布在哪些存储节点上，以及它们的状态等详细信息，适合用于查看特定卷的分片和副本部署情况。
