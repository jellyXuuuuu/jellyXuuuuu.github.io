---
layout: post
title: Github 创建，引用现有的子仓库
tags: [github]
---

# [Github] 如何在github创建仓库中引用现有的子仓库

要在 GitHub 仓库中引用现有的子仓库，通常有两种方法：Git 子模块（Git Submodules）和 Git 子树（Git Subtrees）。此处仅进行 子模块 方法介绍。

## 方法一：使用 Git 子模块（推荐）

### 步骤 1：添加子模块
```bash
git submodule add <子仓库URL> <本地路径>
# 示例：将 my-subrepo 仓库添加到 main-repo 的 sub/ 目录下
git submodule add https://github.com/用户名/my-subrepo.git sub/
```

因为部署原因，我这里想把YCSB公共库创建为我的子库，因此可以先fork YCSB公共库然后再进行git submodule add

### 步骤 2：提交更改
```bash
git add .gitmodules sub/
git commit -m "添加子模块 my-subrepo"
git push origin main
```

### 步骤 3：克隆包含子模块的仓库
当其他人克隆主仓库时，需要额外参数来初始化子模块：
```bash
git clone --recurse-submodules <主仓库URL>
# 或者克隆后手动初始化
git clone <主仓库URL>
cd 主仓库目录
git submodule init  # 初始化配置
git submodule update  # 拉取子模块代码
```
