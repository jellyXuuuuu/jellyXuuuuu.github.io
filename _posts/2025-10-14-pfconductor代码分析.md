---
layout: post
title: pfconductor代码分析
tags: [Pureflash, conductor, 分布式存储]
---

## 分析代码(1)
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


## 251020
*分析代码(2)*
### main流程整理
`conductor`的初始化流程如下图:
![251020-image1](\assets\251020-image1.png)

```java
ClusterManager.zkBaseDir = "/pureflash/"+clusterName;
ClusterManager.registerAsConductor(managmentIp, zkIp);
ClusterManager.waitToBeMaster(managmentIp);
S5Database.getInstance().init(cfg);
ClusterManager.zkHelper.createZkNodeIfNotExist(ClusterManager.zkBaseDir + "/stores", null);
ClusterManager.watchStores();
ClusterManager.updateStoresFromZk();
ClusterManager.zkHelper.createZkNodeIfNotExist(ClusterManager.zkBaseDir + "/shared_disks", null);
ClusterManager.watchSharedDisks();
ClusterManager.updateSharedDisksFromZk();
```
- 解释以上代码:

| 从zk服务中读取目录路径`zkBaseDir`
| 注册`conductor`
| 注册`leader conductor`
| `S5Database.getInstance().init(cfg)` - `.init(cfg)`：这是对`getInstance()`返回的实例对象调用`init`方法，作用是初始化数据库。
| 注册节点进zk
| zk加`watchStores`
| zk加`updateStoresFromZk`
| (shared) zk加`watchSharedDisks`
| (shared) zk加`updateSharedDisksFromZk`

`getInstance()`方法是返回一个`S5Database instance`, 一个`S5Database`类的实例. (`jconductor/src/com/netbric/s5/orm/S5Database.java`)

### `prepareVolume`是用于`open_volume`, `recoveryVolume`, `moveVolume`的

### jconductor日志启动
启动服务

```
# cd /home/flyslice/yangxiao
# source /home/flyslice/yangxiao/cocalele/PureFlash/build_deb/env.sh
# nohup pfs -c /etc/pureflash/pfs.conf > /home/flyslice/yangxiao/pfstore.log 2>&1 &
```

启动 pfconductor
```
/home/flyslice/yangxiao/cocalele/jconductor/pfc中的CROOT=/root/v2/jconductor
```
修改改为`CROOT=/home/flyslice/yangxiao/cocalele/jconductor`

```
# source /home/flyslice/yangxiao/cocalele/jconductor/env-pfc.sh
# nohup pfc -c /etc/pureflash/pfc.conf > /home/flyslice/yangxiao/pfconductor.log 2>&1 &
```


## 251103

### 分析scrub_volume

`scrubVolume`: 在`src\com\netbric\s5\conductor\Scrubber.java`

这段代码定义了一个名为scrubVolume的静态方法，用于触发对指定卷（Volume）的数据擦洗（Scrub）操作，属于存储系统中保障数据一致性的核心功能。其作用是通过校验卷的所有分片（Shard）及其副本（Replica）的数据完整性，检测并标记不一致的副本，确保数据可靠性。


用法：./pfcli scrub_volume -v volume_name


### 分析deep_scrub_volume

`deepScrubVolume`: 在`src\com\netbric\s5\conductor\Scrubber.java`

#### 对比
这两个方法 `scrubVolume` 和 `deepScrubVolume` 都是用于校验存储卷（Volume）的一致性，但在校验深度、粒度和实现细节上有明显区别，主要差异如下：


**校验粒度不同**
- **`scrubVolume`（普通校验）**：  
  对每个分片（Shard）的副本（Replica）进行**整体校验**。  
  通过调用 `calculate_replica_md5` 接口获取整个副本的 MD5 哈希值，然后与主副本（Primary Replica）的 MD5 对比，判断副本是否一致。

- **`deepScrubVolume`（深度校验）**：  
  对每个分片的副本进行**更细粒度的对象级校验**。  
  先通过 SQL 查询获取每个副本所在存储介质（tray）的 `object_size`，然后按对象索引（`obj_idx`）遍历分片内的所有对象，调用 `calculate_object_md5` 接口获取每个对象的 MD5，再与主副本对应对象的 MD5 对比。  
  （分片总大小 `VolumeIdUtils.SHARD_SIZE` 除以 `object_size` 得到对象总数，逐个校验）。


**数据查询范围不同**
- **`scrubVolume`**：  
  查询副本时仅关联 `t_replica`（副本表）和 `t_store`（存储节点表），获取副本的基本信息（ID、存储节点 IP 等）：  
  ```sql
  select r.id replica_id, r.tray_uuid, replica_index, mngt_ip 
  from t_replica r, t_store s 
  where r.store_id=s.id and r.shard_id=? and r.status=?
  ```

- **`deepScrubVolume`**：  
  查询时额外关联 `t_tray`（存储介质表），目的是获取 `object_size`（对象大小），用于后续按对象拆分校验：  
  ```sql
  select r.id replica_id, r.tray_uuid, replica_index, mngt_ip, t.object_size 
  from t_replica r, t_store s, t_tray t 
  where r.store_id=s.id and r.shard_id=? and r.status=? and t.uuid=r.tray_uuid
  ```


**异常处理细节不同**
- **`deepScrubVolume` 多了对象大小一致性校验**：  
  若同一分片中不同副本的 `object_size` 不一致，直接判定校验失败并终止任务：  
  ```java
  if(r.object_size != object_size){
      logger.error("Failed scrub, replicas has different object size, can't continue");
      t.status = BackgroundTaskManager.TaskStatus.FAILED;
      return;
  }
  ```

- **主副本缺失时的处理不同**：  
  - `scrubVolume`：若主副本不存在或 MD5 为空，跳过该分片校验。  
  - `deepScrubVolume`：若主副本不存在，会默认取第一个副本作为临时主副本继续校验（`primary = reps.get(0)`）。


**任务类型标识不同**
- `scrubVolume` 发起的任务类型为 `BackgroundTaskManager.TaskType.SCRUB`。  
- `deepScrubVolume` 发起的任务类型为 `BackgroundTaskManager.TaskType.DEEP_SCRUB`。  
  （用于区分任务类型，可能在任务管理、监控等场景中使用）。


**总结**
- **普通校验（`scrubVolume`）**：轻量、快速，校验副本整体一致性，适合日常快速检查。  
- **深度校验（`deepScrubVolume`）**：更严格、耗时更长，校验到每个对象的一致性，能发现局部数据损坏（如单个对象异常），适合定期深度检查。



### 分析recovery_volume

通过`fromName`方法，`src\com\netbric\s5\orm\Volume.java`:

```java
public static Volume fromName(String tenant_name, String volume_name) {
	return S5Database.getInstance().sql("select v.* from t_volume as v, t_tenant as t where t.id=v.tenant_id and t.name=? and v.name=?",
			tenant_name, volume_name).first(Volume.class);
}
```

数据库结果:
```sql
MariaDB [s5]> select v.* from t_volume as v, t_tenant as t where t.id=v.tenant_id;
+------------+---------+--------------+------+-------+-----------+-----------+-------------+----------+----------+----------+---------+-----------+----------+-------------+---------------------+
| id         | name    | size         | iops | cbs   | bw        | tenant_id | quotaset_id | status   | meta_ver | features | exposed | rep_count | snap_seq | shard_size  | status_time         |
+------------+---------+--------------+------+-------+-----------+-----------+-------------+----------+----------+----------+---------+-----------+----------+-------------+---------------------+
| 3674210304 | test_v6 | 966367641600 | 8192 |     0 | 167772160 |         1 |           0 | OK       |        0 |        0 |       0 |         2 |        1 | 68719476736 | 2025-10-30 19:29:20 |
| 3690987520 | test_v5 | 966367641600 | 8192 | 16384 | 167772160 |         1 |           0 | OK       |        0 |        0 |       0 |         2 |        1 | 68719476736 | 2025-10-31 15:24:20 |
| 3707764736 | test_v4 | 966367641600 | 8192 | 16384 | 167772160 |         1 |           0 | DEGRADED |        4 |        0 |       0 |         2 |        1 | 68719476736 | 2025-10-31 15:05:06 |
| 3724541952 | test_v3 | 966367641600 | 8192 |     0 | 167772160 |         1 |           0 | OK       |        0 |        0 |       0 |         2 |        1 | 68719476736 | 2025-10-30 19:33:59 |
+------------+---------+--------------+------+-------+-----------+-----------+-------------+----------+----------+----------+---------+-----------+----------+-------------+---------------------+

```

- `BackgroundTaskManager`:

定义了一个`BackgroundTask`类，用来管理后台任务。核心作用是避免耗时操作阻塞主线程或前端请求，同时提供任务的状态监控和管理能力。



### 分析update_volume

update_volume 方法是一个用于更新存储卷（Volume）信息的接口实现，主要功能是接收请求参数，对指定卷的名称、大小、IOPS（每秒输入 / 输出操作数）、带宽（bw）等属性进行更新，并处理相关的数据库事务和异常。


### 分析move_volume

用到了`RebalanceManager`:

```java
S5Database.getInstance().sql("update t_volume set status=IF((select count(*) from t_shard where status!='OK' and volume_id=?) = 0, 'OK', status)" +
				" where id=?", v.id, v.id).execute();
```

`moveVolume` 方法的主要功能是将一个卷（Volume）的副本（Replica）从源存储位置迁移到目标存储位置
步骤:
1. 获取待迁移的卷对象从任务参数 task.arg 中获取当前要迁移的卷（Volume）对象 v。
2. 查询需要迁移的副本通过数据库查询，筛选出属于当前卷（v.id）、且位于源存储节点（fromStoreId）和源 SSD（fromSsdUuid）上的所有副本（Replica），存入 repsToMove 列表。
3. 逐个迁移副本
先通过目标存储节点 ID（targetStoreId）获取目标存储节点对象 n。
遍历 repsToMove 中的每个副本，调用 moveReplica 方法将副本从源位置迁移到目标存储节点 n 和目标 SSD（targetSsdUuid）。
每迁移完一个副本，更新任务进度（task.progress），按已迁移数量占总数量的比例计算百分比。
4. 更新卷的状态 - 迁移完成后，执行数据库更新：
检查该卷下所有分片（t_shard）的状态，如果所有分片状态都是 OK，则将卷的状态设为 OK；否则保持原有状态。
最后将任务进度设为 100%，标识迁移完成。



### 分析check_volume_exists
代码
```java
long v = S5Database.getInstance().queryLongValue("SELECT EXISTS (select v.* from t_volume as v, t_tenant as t where t.id=v.tenant_id and t.name=? and v.name=?)",
					tenant_name, volume_name);
```


```sql
SELECT EXISTS (
  SELECT v.* 
  FROM t_volume AS v, t_tenant AS t 
  WHERE t.id = v.tenant_id 
    AND t.name = 'tenant_default'  -- 字符串加单引号
    AND v.name = 'test_v6'        -- 字符串加单引号
);
```

这条 SQL 语句的作用是判断 “指定租户下是否存在指定名称的卷”，返回结果为 1（存在）或 0（不存在），是一种高效的 existence check（存在性检查）。




### 查询语句, 查询某卷相关的详细信息
查询名为 `test_v1` 的卷（`Volume`）相关的详细信息，包括其分片（`Shard`）、副本（`Replica`）、存储节点（`Store`）等关联资源的状态和属性，主要用于存储系统中排查卷的部署情况、副本分布、健康状态等问题。

```sql
select v.name volume_name, r.volume_id volume_id, d.id shard_id, d.shard_index shard_index, r.replica_id replica_id, r.replica_index replica_index, mngt_ip store_ip, is_primary, s.id store_id, 
r.status rstatus, v.status vstatus, s.status sstatus, d.status dstatus  
from 
v_replica_ext r, t_volume v,t_store s, t_tray t, t_shard d 
where r.store_id=s.id and t.uuid=r.tray_uuid and d.volume_id=r.volume_id and d.volume_id=v.id and d.id=r.shard_id and v.name='test_v1'   
ORDER BY shard_id ASC;

```


