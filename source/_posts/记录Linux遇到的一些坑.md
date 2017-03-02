---
title: 记录Linux遇到的一些坑
date: 2016-11-14 16:29:30
tags: [CentOS,Linux]
categories: 
- Linux
---
来一篇文章，对之前，最近，乃至后面的Linux折腾中遇到的一些坑进行汇总，一方面是个总结，一方面希望能够给遇到同样问题的小伙伴儿有一点点儿帮助。

这也是我第一篇关于Linux的文章。文章主要汇总一些小问题，有必要会对一些问题开单独文章详细总结。

**新手上路，踩着坑，一步步前行。**

<!--more-->

**环境**

CentOS 7.0

### VM使用NAT模式联网问题

#### 问题描述:

查看启用的网卡发现只有lo,virbr0,并没有发现正常情况下的eth0(或者用于访问网络的网卡),/etc/sysconfig/network-scripts下没有ifcfg-eth0配置文件

#### 解决过程:

起初尝试过收到创建ifcfg-eth0文件，进行相关配置，再

```
service network restart
```
failed,查看报错详情:
![ifcfgerror](http://7xsi10.com1.z0.glb.clouddn.com/ifcfgLBS.png)

查询资料说是mac地址问题，可我已经配置好mac地址。

#### 解决办法:

由于VM虚拟网卡和Linux兼容问题导致。其默认使用的AMD网卡，而我的真机为Intel的网卡，需要手动修改配置，更改网卡类型。

找到CentOS.vmx配置文件，新增如下一条:
```
ethernet0.virtualDev = "e1000"
```

这样，就有用于访问网络的网卡了。并不一定是eth0。

### libXss.so.1问题

#### 问题描述

配置VSCode遇到的问题，正常安装，无法正常启动，报如下错误:

![libXerror](http://7xsi10.com1.z0.glb.clouddn.com/libXS.png)
缺失libXss.so.1

#### 解决方法:

从CentOS官网下载libSScrnSaver rpm文件，然后进行安装

![solvelibX](http://7xsi10.com1.z0.glb.clouddn.com/libXS1.png)

### 双系统为ubuntu扩充硬盘问题

从windows将某磁盘压缩，新建空闲分区后，进入ubuntu时，提示

```
unknown fileSystem
```

由于分区情况发生变化，导致grub找不到

#### 解决方法:
列出所有磁盘: 
```
ls
```
一个一个尝试如: 
```
ls (hd0,10)
```
找到ubuntu启动盘，我启动盘是单独的boot分区，在(hd0,10)下
此时 
```
 set root=hd0,msdos10
 set prefix=(hd0,msdos10)/grub
```
接着
```
insmod normal
normal
```
最后进入系统修复grub
```
sudo update-grub
sudo grub-install /dev/sda
```
注意这里sda后不要加盘号

**未完待续**