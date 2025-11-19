---
layout: post
title: 新增feature增加实际alloc_size-pfc
tags: [Pureflash, conductor, 分布式存储]
---

## 任务：分析count/space

根据代码定义如下
```java
/**
 * Get the available space in queue.
 *
 * @param[in]	queue	queue to compute
 *
 * @return available space in queue.
 */
inline int space() { 	return QUEUE_SPACE(queue_depth, head, tail); }

/**
 * get the valid entries count in queue
 *
 * @param[in]	queue	queue to compute
 *
 * @return valid element number in queue.
 */
inline int count() {	return QUEUE_COUNT(queue_depth, head, tail); }

```

| `space`是队列的剩余空间, `free_obj_queue.space()`是存储空闲对象队列的剩余空间，即所使用的空间。因此`free_size`所需的是`count`.


参考以下原有日志打印:
```java
S5LOG_INFO("After second replay, key:%d total obj count:%d free obj count:%d, in triming:%d, obj size:%lld", obj_lmt.size(),
                        free_obj_queue.queue_depth - 1, free_obj_queue.count(), trim_obj_queue.count(), head.objsize);
```

| `count`是表示已有的元素，而`free_obj_queue`里面存的就是剩余空间对列，因此`free_obj_queue.count()`可以获取剩余空间；`space`是队列中还能容纳的元素数量，就是`free_obj_queue`的剩余空间，所以对应已经使用的空间.


```java
void handle_get_obj_count(struct mg_connection *nc, struct http_message * hm) {
	int cnt = 0;
	for(auto disk : app_context.trays) {
		cnt += disk->event_queue->sync_invoke([disk]()->int{
			return disk->free_obj_queue.space();
		});
	}
	mg_send_head(nc, 200, 16, "Content-Type: text/plain");
	mg_printf(nc, "%-16d", cnt);
}
```
这个函数的名字是 “获取对象计数”`（get_obj_count）`，但结合代码中用`space()`累加的逻辑，它实际统计的是：所有磁盘的空闲对象队列 “还能接收多少个空闲对象” 的总容量（即系统对 “空闲对象” 的总管理能力余量）。

| - 系统能接收多少新释放的空闲对象



# 总结

| `handle_get_obj_count` 用 `space()` 是因为它要统计 “系统回收空闲对象的能力余量”，这是一个面向对象回收的指标。`free_size` 需要统计 “当前可分配的空闲空间”，这是一个面向数据写入的指标，因此必须用 `count()`。



注:

```java
int obj_count = (int)((head.tray_capacity - head.meta_size) >> head.objsize_order);
```

这行是初始化流程用到的，所以系统的free_size最开始需要先减去`head.meta_size`.

