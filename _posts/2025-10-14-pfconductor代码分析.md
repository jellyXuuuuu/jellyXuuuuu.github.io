---
layout: post
title: pfconductor代码分析
tags: [Pureflash, conductor, 分布式存储]
---

## 目标/任务: 查看com/netbric/s5/conductor/handler/S5RestfulHandler.java，看看还有什么命令，如果pfccli没有实现，思考应该怎么实现

### S5RestfulHandler.java
这是一个服务接口，用于后端可以操作volume/shard/replica等



### handler下的StoreHandler.java/TenantHandler.java/VolumeHandler.java
`StoreHandler.java`的注释里写了`backend handler of CLI s5_add_store_node.py`，应该是基于原本的`s5_add_store_node.py`文件改造的，是后端操作对于`storenode`增删检查的
同理，`TenantHandler.java`针对tenants, `VolumeHandler`针对volume

