---
layout: post
title: VSCode remote ssh连不上
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


#### 更新VSCode 版本
怀疑原因是Windows的VSCode版本太久，重启了一下自动更新了vscode版本就好了。

- 更新vscode版本
```
Version: 1.102.0 (user setup)
Commit: cb0c47c0cfaad0757385834bd89d410c78a856c0
Date: 2025-07-09T22:10:34.600Z
Electron: 35.6.0
ElectronBuildId: 11847422
Chromium: 134.0.6998.205
Node.js: 22.15.1
V8: 13.4.114.21-electron.0
OS: Windows_NT x64 10.0.19045
```

- 插件版本
![250805-image5](\assets\250805-image5.png)
```
Remote - SSH  v0.120.0 
Remote Explorer  v0.5.0  
```
