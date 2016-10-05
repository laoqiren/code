---
title: ruby-sass-problem
date: 2016-04-10 23:25:08
tags: sass
categories:
- css
---
# 关于windows下通过ruby安装sass和compass的问题
### 安装sass和compass
<!--more-->
执行 gem sources -a https://ruby.taobao.org/出现如下错误
![error](http://7xsi10.com2.z0.glb.clouddn.com/QQ%E5%9B%BE%E7%89%8720160410233014.png)
缺少SSL证书
#### 解决办法
1. 下载证书：[官方地址](https://curl.haxx.se/ca/cacert.pem)　[百度云盘](http://pan.baidu.com/s/1pKJSlOf)
2. 将证书保存，比如我的保存到了C:\Ruby22-x64。
3. 配置证书的环境![path](http://7xsi10.com2.z0.glb.clouddn.com/870258-20160405180306187-1063124604.png)
4. 重启，再执行，当当当当，哈哈，成功了![succed](http://7xsi10.com2.z0.glb.clouddn.com/QQ%E5%9B%BE%E7%89%8720160410233111.png)
##### 感谢angular社区前辈 会飞的鱼lala 提供的方法
