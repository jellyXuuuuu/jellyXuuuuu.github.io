---
layout: post
title: Jitisi初步安装运行GUI
---

task: "D:\tools\Google Chrome\chrome.exe" https://map.1tyun.ink:8903/a  这个是一个我做的Windows BAT脚本。目的是用我指定的V73版本的Chrome浏览器打开我的Jitsi视频会议平台的指定房间（a)。 ...

用pywinauto - win下自动化ui工具 - 去自动运行jitisi meet，关闭一些杂项



首先用了网上https://www.kancloud.cn/gnefnuy/pywinauto_doc/1193036 https://zhuanlan.zhihu.com/p/37283722的入门教程，第一步就卡了！



```python
from pywinauto.application import Application
# 对于Windows中自带应用程序，直接执行，对于外部应用应输入完整路径
app = Application(backend="uia").start('notepad.exe') 
```

这个代码按道理pip安装以后应该可以无误运行，但是一直没有反应，并且报错：

![image-20210714040547774](\assets\image-20210714040547774.png)

上网查了原因是pywinauto有bug，python3.7之后的版本都会报错，所以尝试用python3.7运行，但是可能pip安装也不对，所以就用了pipenv这个不同版本协调的类似pip工具 - 

https://www.zhihu.com/question/368411359

![image-20210714040738133](\assets\image-20210714040738133.png)

这里我用的命令是

pipenv -- python 3.7

![image-20210714040859822](\assets\image-20210714040859822.png)

哈哈然后发现电脑上有一般3.7（刚好）不然还要找重新安装3.7

3.7目录：D:/Program Files (x86)/Microsoft Visual Studio/Shared/Python37_64/python.exe

电脑上目前最新是3.8 - 目录：D:/computer/python3/python.exe

然后当前环境安装pywinauto - 即用python3.7避免bug， 再“激活环境” - pipenv shell 

![image-20210714041240769](\assets\image-20210714041240769.png)

然后就环境下直接python运行script就行啦

![image-20210714041406917](\assets\image-20210714041406917.png)



------- 第一步终于结束了。。 运行过了可以成功打开网页

代码就只是把notepad.exe改了

```python
from pywinauto.application import Application
# 对于Windows中自带应用程序，直接执行，对于外部应用应输入完整路径
app = Application(backend="uia").start('"D:\\computer\\tools\\googlechrome73\\Google Chrome\\chrome.exe" --force-renderer-accessibility https://map.1tyun.ink:8903/a"') 
```