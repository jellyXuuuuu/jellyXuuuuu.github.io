---
layout: post
title: Windows git setup problems2
tags: [windows, github]
---

## 场景：在windows下配置git并且在vscode中关联能够进行git追踪2

### 问题1：git push会报错
遇到的这个错误是因为没有权限访问远程仓库。`deploy key` 是指用于特定仓库的 SSH 密钥。这个错误表明你正在使用的部署密钥没有权限对 `jellyXuuuuu/YCSB-benchmark.git` 仓库进行写操作。
```shell
PS D:\xyy\code\YCSB-benchmark> git status
On branch main
Your branch is ahead of 'origin/main' by 1 commit.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean
PS D:\xyy\code\YCSB-benchmark> git push
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

#### 解决方法：
1. 更换为个人 SSH 密钥
```bash
# 查看当前使用的 SSH 密钥
cat ~/.ssh/id_rsa.pub  # 若这不是你的个人公钥，可继续往下操作

# 生成新的 SSH 密钥（如果没有的话）
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# 将新生成的公钥添加到 GitHub 账户
cat ~/.ssh/id_rsa.pub
```
![250627-image1](\assets\250723-image1.png)
![250627-image2](\assets\250723-image2.png)

将新的key输入到guthub网页即可。

![250627-image3](\assets\250723-image3.png)
