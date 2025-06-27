---
layout: post
title: Windows git setup problems
---

## 场景：在windows下配置git并且在vscode中关联能够进行git追踪

### 问题1：git clone https 会报错 'reset by connection'
该问题原因是dns配置问题，同样git clone git下也会遇到，具体现象是运行一半卡住然后一直不动，等待很久后报错
```shell
PS D:\xyy\code\blog> git clone git@github.com:jellyXuuuuu/jellyXuuuuu.github.io.git    
Cloning into 'jellyXuuuuu.github.io'...
remote: Enumerating objects: 229, done.
remote: Counting objects: 100% (229/229), done.
remote: Compressing objects: 100% (186/186), done.
Receiving objects:  54% (124/229), 2.04 MiB | 1.74 MiConnection to github.com closed by remote host.
fetch-pack: unexpected disconnect while reading sideband packet
fatal: early EOF
fatal: fetch-pack: invalid index-pack output
```

#### 解决方法：
1. 修改编辑系统 hosts 文件（需管理员权限），添加 GitHub 的 IP 地址：
```
140.82.114.4    github.com
185.199.108.153  assets-cdn.github.com
185.199.111.153  github.io
```
![250627-image1](\assets\250627-image1.png)

注：所有DNS相关问题解决方法基本相同，Windows下是手动打开修改`C:\Windows\System32\drivers\etc\hosts`，linux下是修改`/etc/hosts`

2. windows cmd下输入
```shell
ipconfig /flushdns
```
![250627-image2](\assets\250627-image2.png)

### 问题2：git clone git 需要配置ssh key

参考：
[github官网增加ssh-key教程](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

1. 检查ssh是否存在
```shell
ls -al ~/.ssh  # 查看是否有id_rsa.pub或id_ed25519.pub等公钥文件
```
如果没有密钥文件（如 id_rsa.pub 或 id_ed25519.pub），需要生成新的 SSH 密钥。

2. 生成新的SSH密钥
```shell
ssh-keygen -t ed25519 -C "your_email@example.com"  # 使用你的GitHub邮箱
```

3. 将SSH密钥添加到SSH代理
```shell
eval "$(ssh-agent -s)"  # 启动SSH代理
ssh-add ~/.ssh/id_ed25519  # 添加私钥（如果使用默认名称）
```

4.  将公钥添加到 GitHub 账户
```shell
cat ~/.ssh/id_ed25519.pub  # 输出公钥内容
```
![250627-image3](\assets\250627-image3.png)

![250627-image4](\assets\250627-image4.png)
