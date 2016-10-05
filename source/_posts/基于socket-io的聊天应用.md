---
title: 基于socket.io的聊天应用
date: 2016-06-01 22:07:31
tags: [socket.io,websocket]
categories: 
- nodeJS
---
WebSocket是HTML5的一种新通信协议，它实现了浏览器与服务器之间的双向通讯。而Socket.IO是一个完全由JavaScript实现、基于Node.js、支持WebSocket的协议用于实时通信、跨平台的开源框架，它包括了客户端的JavaScript和服务器端的Node.js。

本文记录利用socket.io开发聊天室的实践
![user1](http://7xsi10.com1.z0.glb.clouddn.com/user1.png)
<!--more-->
##### Socket.IO除了支持WebSocket通讯协议外，还支持许多种轮询（Polling）机制以及其它实时通信方式，并封装成了通用的接口，并且在服务端实现了这些实时机制的相应代码。Socket.IO实现的Polling通信机制包括Adobe Flash Socket、AJAX长轮询、AJAX multipart streaming、持久Iframe、JSONP轮询等。Socket.IO能够根据浏览器对通讯机制的支持情况自动地选择最佳的方式来实现网络实时应用。

#### a. 需要依赖的模块

1. 客户端jquery
2. express
3. express的静态文件服务
4. http模块
5. socket.io

##### Nodejs服务端server.js:
```js
var app = require('express')();
var http = require('http').Server(app);
var io = require('socket.io')(http);
var staticServer = require('express-static');
```
##### 客户端引入socket.io包的soket.io.js
```html
<script type="text/javascript" src="./socket.io.js"></script>
```
#### b. 客户端 index.html

##### 界面以清新简约为主，此处省略样式代码，最终界面效果如下：
![UI](http://7xsi10.com1.z0.glb.clouddn.com/chatUI.png)

##### 来看几个客户端socket.io的API:
1. io(String url,Obj opts) 暴露于window下，可传入建立连接的url字符串和用于设置连接的对象，如果不设置url,则默认建立到位客户端提供服务的服务器根目录
2. emmit()事件机制，触发某个事件

##### index.html:

1. 建立连接:
```js
var socket = io();
```
2. 触发服务端用户加入事件join:
```js
socket.emit('join',name);
```		

3. 消息发送，触发服务端消息发送事件text:
```js
socket.emit('text',$('#msg').val());
```
4. 监听由服务器当其他用户加入时触发的announcement事件,通知新用户加入:
```js
socket.on('announcement',function(msg){
            $('.otherEnter ul').append('<li>'+msg+'</li>');
        });
```
5. 监听由服务端当其他用户有消息发送时触发的text事件，将新消息加入显示:
```js
socket.on('text',function(content){
            $('.chatList ul').append('<li>'+ content +'</li>');
        });
```
6. 自己发的消息要在触发服务器text事件之前显示,因为socket.io广播是默认不给本用户触发指定事件的:
```js
$('.chatList ul').append('<li><span>me</span>:'+$('#msg').val()+'</li>')
            socket.emit('text',$('#msg').val());
```
#### c. 服务端 server.js

1. 建立静态文件服务器，用到express-static中间件:
```js	
app.use(staticServer('./'));
app.get('/',function(req,res){
    res.render('index');
});
http.listen(8068,function(){
    console.log("Sever has been listened at port 8068");
});
```
![server](http://7xsi10.com1.z0.glb.clouddn.com/chatServer.png)

2. 将http服务绑定到socket.io服务:
```js
var io = require('socket.io')(http);
```
3. 监听connection事件，当有新连接时触发:
```js	
io.on("connection",function(socket){
	console.log("some on connected");
}
```
![connect](http://7xsi10.com1.z0.glb.clouddn.com/chatconnect.png)
4. 新用户连接:join事件,var total = 0;用于统计在线人数,每次新用户加入，通知其他所有用户，并完成total的更新，这就要用到socket.io的广播了，通过.broadcast来进行广播:
```js
total++;
console.log(name+"joined");
socket.broadcast.emit('announcement',name+' joined in');
socket.broadcast.emit('totalChange',total);  //广播用户总数改变事件
```
5. 当某个链接断开时，需要通知其他用户，total更新,socket.io的disconnect事件，当链接断开时触发:
```js	
socket.on('disconnection',function(){
        total--;
        console.log('hi,leave');
        socket.broadcast.emit('announcement',name+' leved away');
        socket.broadcast.emit('totalChange',total);
    });
```	
6. 广播聊天内容:
```js
socket.on('text',function(msg){
    socket.broadcast.emit('text','<span>'+name+'</span>'+':'+msg);
});
```

#### 最终测试效果如下：

![user1](http://7xsi10.com1.z0.glb.clouddn.com/user1.png)
![user2](http://7xsi10.com1.z0.glb.clouddn.com/user2.png)
![user3](http://7xsi10.com1.z0.glb.clouddn.com/user3.png)

正如截图中的旁白君所说，应用还有bug和需要改善的地方，所以这篇文章会继续更新！
有兴趣的，可以到我的github查看应用全部源码:
[我的github](https://github.com/laoqiren/socket.io-demo)