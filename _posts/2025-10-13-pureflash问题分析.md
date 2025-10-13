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


### 分析各个table/view表格的作用

![](\assets\.png)

