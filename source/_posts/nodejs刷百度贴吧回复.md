---
title: nodejs刷百度贴吧回复
date: 2016-04-21 23:15:57
tags: nodejs
categories:
- nodeJS
---
nodeJS提供的网络API，使得nodeJS既能作为服务端接受请求，又能作为客户端像其他服务器发起请求，对于一些论坛诸如贴吧之类的，可以通过请求信息（cookie信息)就能够实现脚本自动发帖，这是初次接触nodeJS的尝试，虽然用到的技术较简单，但是还是挺有趣的，比如你想给自己顶贴之类的小需求，可以自己写一个刷回复的脚步来实现哦！ <!--more-->
### 作为前端必备技能的nodejs,非常有趣
#### 利用NodeJs的http模块，很简单地就能做一个刷回复的脚本

这次借用高考吧一学妹的帖子来刷一刷，哈哈，邪恶邪恶
![tie](http://7xsi10.com2.z0.glb.clouddn.com/shua1.png)
nodejs提供的http模块，我们需要用http模块的request方法想服务端发送请求来发送/接收信息
1. require进两个模块
```js
var http = require('http');
var querystring = require('querystring');
```
2. request方法返回一个http.ClientRequest的实例，如果用POST方法向服务端发送数据，数据对象会被写入该对象，request方法接受两个参数，一个必须的options参数，一个可选的callback函数，callback函数可以接受到来自服务端的响应作为参数，options参数可以为字符串或者对象，字符串会被url模块的parse方法序列化为对象
3. 打开学妹的帖子，我们编辑一条回复信息，然后查看网络面板，查看add的ajax请求信息
   ![add](http://7xsi10.com2.z0.glb.clouddn.com/shua2.png)
4. options参数的header属性值为请求头信息，当然得是Json对象
5. 注意Content-Length要设置为postData.length，不然可能请求失败
6. 在最开始的时候，老是出现如下错误:
![error](http://7xsi10.com2.z0.glb.clouddn.com/shua3.png)
最后查阅错误代码，stackoverflow上的解决办法是不加http/https协议，果然，成功
![success](http://7xsi10.com2.z0.glb.clouddn.com/shua4.png)
7. 然后，邪恶的加个定时器，一直刷，看结果
![many](http://7xsi10.com2.z0.glb.clouddn.com/many.png)
![many](http://7xsi10.com2.z0.glb.clouddn.com/shua6.png)
8. 很好玩吧，不过多次刷会出现验证码的，哎，刚学，知识技能不熟悉，只能写个简单的yy一下了，不过这测验也能看出百度的安全并不是那么好，哈哈