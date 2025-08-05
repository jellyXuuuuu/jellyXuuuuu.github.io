---
layout: post
title: 复原交叉编译环境
tags: [Liunx, ssh, vscode]
---

## 场景：远端服务器更换系统等原因导致VSCode remote ssh连不上

### 原因分析：
很早之前就连接配置过，但是后来远端服务器改了系统，导致key不匹配。
```
Host key for 192.168.61.249 has changed and you have requested strict checking. Host key verification failed.
```
你连接的服务器（192.168.61.249）的主机密钥已改变，但 SSH 默认启用了 “严格检查”，因此拒绝连接。

同时本机Windows上的VSCode版本可能较低/较高，不一定导致该原因但可以作为参考。


### 解决方法：

#### 删除配置重新连接
删除C:\Users\xuyy\.ssh\known_hosts
![250805-image3](\assets\250805-image3.png)

![250805-image4](\assets\250805-image4.png)
重新连接即可。



