---
title: Windows下React Native安卓环境踩坑记
date: 2016-09-06 12:23:46
tags: [React Native,React,安卓开发]
categories: 
- React Native
---
React Native是FaceBook开源的一款用于开发移动app的框架，不用java开发安卓，不用object-c或swift开发IOS，而是用一套JS（ReactJS）。"LEARN ONCE, WRITE ANYWHERE: BUILD MOBILE APPS WITH REACT".

最近折腾折腾用windows来进行react-native开发安卓，毕竟windows，各种坑，搞了好几个小时，功夫不负有心人，最终成功的迎来了Hello,React Native!值得写一篇文章记录记录了。

![真机测试成功](http://7xsi10.com1.z0.glb.clouddn.com/successyeah.png)

有钱了一定要买Mac!!

前几天的NingJS大会上，尤大大宣布Vue和阿里的weex团队合作，目的是让大家可以用vue来开发原生app。虽然还有很长的路要走。。期待国内技术的未来。
<!--more-->

**具体的配置过程react native文档还是写得比较详细的，文章只是记录我遇到的一些问题，通过各种渠道，最终解决完成，感谢社区的各位前辈的热心帮助**

### 虚拟机运行安卓模拟器问题

**问题描述**
在windows上我使用Genymotion安卓模拟器来开发，从virtualBox虚拟机启动，提示报错如下:
![运行模拟器出错](http://7xsi10.com1.z0.glb.clouddn.com/qs01.jpg)

查了查错误，原来是我的系统默认就是破解了主题的，就是uxtheme.dll有关主题的文件出的问题。

**解决办法**
使用主题破解/恢复软件来恢复主题破解，当自己本地没有原厂的主题文件备份时，需要下载原始文件,找到包含了原始文件备份的UniversalThemePatcher软件，[百度云地址:](http://pan.baidu.com/s/1hqpOljY) 提取密码:4p51，解压里面有32位和64位不同的原始文件备份，将里面的三个原始备份.backup文件复制到c:\windows\system32下面,再运行UniversalThemePatcher,就可以恢复了，恢复后的结果：
![恢复主题破解](http://7xsi10.com1.z0.glb.clouddn.com/slove01.png)
这样，安卓模拟器就运行了。

### 模拟器运行应用一片空白，红框报错:
**问题描述**
react-native run-android，apk也是安装了的，packages也是运行中的，没有什么报错，可运行apk就是一片空白加一红框报错。貌似很多人都遇到了这个问题。不过在我的捣鼓和大家的热心帮助下，还是解决了。

![packages红框](http://7xsi10.com1.z0.glb.clouddn.com/qs02.png)

网上看了各种问题解决办法，都尝试过，但是并没有什么卵用。于是我就先试试真机。

**解决办法**
见真机测试部分

### 真机测试

我的安卓机是5.1.1的，打开usb调试，连接电脑，连接电脑的wifi,理论上是没什么问题的

执行adb devices，正常:

![查询设备](http://7xsi10.com1.z0.glb.clouddn.com/device01.png)


**问题:**
执行react-native run-android报错:

![安装apk出错](http://7xsi10.com1.z0.glb.clouddn.com/qs03.png)

提示没法安装apk到手机。。。

**解决办法:**
原来是项目的gradle版本问题，依赖了一个bug比较多的版本，需要改成一个比较稳定的版本如1.2.3。
![gradle](http://7xsi10.com1.z0.glb.clouddn.com/solve02.png)

重新run-android，成功的安装了app，但是打开apk，又一次遇到了上面模拟器的红框错误。

官网教程说的安卓5.0以上是不用配置ip和端口的，但是我最终还是设置了Ip和端口才解决了上面的问题。


吃了个午饭回来

dev settings>debug server host& port for device，然后reload。。。当当当当，神奇的成功了:

![真机测试成功](http://7xsi10.com1.z0.glb.clouddn.com/suc001.png)

我之前设置ip也不行啊。。。不知道是不是它也饿了。。。

然后再回来看看我的模拟器，这下也行了：

![模拟器运行成功](http://7xsi10.com1.z0.glb.clouddn.com/suc002.png)

应该就是上面的gradle版本问题。

**结语**
真是各种问题，各种坑。但终究还是解决了。模拟器找不到错误解决方案，干脆就用真机先试试，正是真机测试找到了解决办法，模拟器的也一并解决了。。。




