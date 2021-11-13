---
layout: post
title: Html Photo Load
---

HTML的图片加载不出来几大问题: 
[https://blog.csdn.net/HandsomeBoo/article/details/51622598](https://blog.csdn.net/HandsomeBoo/article/details/51622598)

       1、图片调用路径不对；
       2、图片名称不对；
       3、图片本身有问题；
       4、图片调用代码问题。

然后基本上其实是调用路径的问题

结果发现原本用了绝对路径，上传到web上面是找不到的，必须要用相对路径

html img几种路径:
[https://www.w3school.com.cn/html/html_filepaths.asp](https://www.w3school.com.cn/html/html_filepaths.asp)

<table>
  <thead>
    <tr>
      <th>路径</th>
      <th>描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>&lt;img src="picture.jpg"&gt;</td>
      <td>picture.jpg 位于与当前网页相同的文件夹</td>
    </tr>
    <tr>
      <td>&lt;img src="images/picture.jpg"&gt;</td>
      <td>picture.jpg 位于当前文件夹的 images 文件夹中</td>
    </tr>
    <tr>
      <td>&lt;img src="/images/picture.jpg"&gt;</td>
      <td>picture.jpg 当前站点根目录的 images 文件夹中</td>
    </tr>
    <tr>
      <td>&lt;img src="../picture.jpg"&gt;</td>
      <td> picture.jpg 位于当前文件夹的上一级文件夹中</td>
    </tr>
  </tbody>

</table>



于是将调用路径从/images/logo.png -> ../images/logo.png，因为需要返回上层文件夹，当前在_layout/文件夹下。
 
![image-211113](\assets\image-211113.png)
（右侧可以看出文件夹结构，左侧为当前上传到github上面显示的页面）

