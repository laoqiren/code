---
title: 聊聊Agent/Proxy
date: 2017-07-16 21:59:39
tags: [Proxy,Agent]
categories: 
- Nodejs
---
聊聊Agent/Proxy，以Node为例，看看`ClientRequest`是如何通过`http.Agent`去管理socket池，用Node写http/https proxy,再自定义Agent并在Agent与目标源之间加入Node proxy。

<!--more-->

## http.Agent

首先`addRequest()`方法是入口，`http_client`发请求前会判断是否有agent,如果有，则会调用agent的`addRequest()`方法:
```js
if (this.agent) {
    if (!this.agent.keepAlive && !Number.isFinite(this.agent.maxSockets)) {
      this._last = true;
      this.shouldKeepAlive = false;
    } else {
      this._last = false;
      this.shouldKeepAlive = true;
    }
    this.agent.addRequest(this, options);
}
```
然后我们来看看官方的`http.Agent`做法:
```js
var name = this.getName(options);
if (!this.sockets[name]) {
this.sockets[name] = [];
}

var freeLen = this.freeSockets[name] ? this.freeSockets[name].length : 0;
var sockLen = freeLen + this.sockets[name].length;
```
我们看到`this.sockets[name]`表示了某个host请求正在处理请求的sockets数组，`this.freeSockets[name]`则表示某个host请求处于free状态下的sockets，两者数量之和的上限就是我们传给`new http.Agent()`构造函数options中的`maxSockets`

```js
if (freeLen) {
    var socket = this.freeSockets[name].shift();
    socket._handle.asyncReset();
    socket[async_id_symbol] = socket._handle.getAsyncId();

    if (!this.freeSockets[name].length)
      delete this.freeSockets[name];

    this.reuseSocket(socket, req);
    req.onSocket(socket);
    this.sockets[name].push(socket);
  } else if (sockLen < this.maxSockets) {
    debug('call onSocket', sockLen, freeLen);
    this.createSocket(req, options, handleSocketCreation(req, true));
  } else {
    debug('wait for socket');
    if (!this.requests[name]) {
      this.requests[name] = [];
    }
    this.requests[name].push(req);
  }
```
如果该host的socket池里有空闲的，就从`freeSockets`里取出一个socket,重用并且将这个socket加入到正在处理请求的sockets里，即`sockets[name]`，如果没有空闲的socket，就准备创建新的socket了，但是首先sockets数不能超过`maxSockets`，如果超过了，就暂时没有socket用了，就加入到等待队列吧,即这里的`this.requests[name]`。

接着我们看看socket是怎么被加入到`freeSockets`的:

在`createSocket()`后，会对新的socket绑定一系列listener,其中包括了`onFree`:
```js
function onFree() {
    debug('CLIENT socket onFree');
    agent.emit('free', s, options);
  }
s.on('free', onFree);
```
socket所处理的request关闭后,就相当于free了，然后agent监听了free事件:
```js
if (socket.writable &&self.requests[name] && self.requests[name].length) {
      self.requests[name].shift().onSocket(socket);
      ...
}
```
free以后就去看还有没有处于等待队列的request,如果有就直接取出一个来复用这个socket

如果没有，就尝试将其加入`freeSockets`
```js
if (count > self.maxSockets || freeLen >= self.maxFreeSockets) {
    socket.destroy();
} else {
    // add socket to freeSockets
}
```
当然需要设定agent的`keep-alive`为true

当socket被close时，直接从socket池里删除，如果requests队列里还有等待的，就重新create一个socket。

## proxy

proxy方式有许多，这里仅仅以http/https proxy为例，且用node实现。

### 盲中继

对于http网站，流量是可以通过proxy直接转发的，比如这里我实现的:
```js
let httpAgent = new http.Agent({
        keepAlive: true
    })
function request(cReq, cRes) {
    let u = url.parse(cReq.url);
    let needAgent = false;

    let headers = Object.assign({},cReq.headers,{
        'x-forwarded-for': cReq.connection.remoteAddress
    });
    if(headers['proxy-connection']){
        headers['connection'] = headers['proxy-connection']
        delete headers['proxy-connection'];
        if(headers['connection'] === 'keep-alive'){
            needAgent = true;
        }
    }
    
    let options = {
        hostname : u.hostname, 
        port     : u.port || 80,
        path     : u.path,       
        method   : cReq.method,
        headers,
        agent: needAgent?httpAgent:null
    };
    let pReq = http.request(options, pRes=>{
        cRes.writeHead(pRes.statusCode, pRes.headers);
        pRes.pipe(cRes);
    }).on('error', e=>{
        cRes.end();
    });

    cReq.pipe(pReq);
}
```
这里实现的中继并不是最傻的，加入了`x-forwarded-for`请求头，根据keep-alive设计原则，去除来自agent的请求头中的逐跳首部`proxy-connection`，如果发现agent想要和源网站建立keep-alive连接，这里就加入`http.Agent`来管理proxy与源之间的keep-alive连接。至于proxy和user-agent之间的keep-alive问题，正在思考解决中。

### CONNECT隧道

想想agent与https网站的交流，由于证书机制，在它们中间加一个proxy会怎样，如果用上面那种方式去实现，则会存在一个问题，proxy服务器的证书没有加入agent的信任证书里，那么proxy与agent就建立不了连接。

我们改用CONNECT方式，通过CONNECT方法建立隧道，来实现转发ssl流量:
```js
function connect(cReq, cSock) {
    let u = url.parse('http://' + cReq.url);
    console.log(cReq.url)
    let pSock = net.connect(u.port, u.hostname, ()=>{
        cSock.write('HTTP/1.1 200 Connection Established\r\nProxy-agent: Node.js proxy server\r\n\r\n');
        pSock.pipe(cSock);
    }).on('error', e=>{
        cSock.end();
    });

    cSock.pipe(pSock);
}
http.createServer()
    .on('request', request)
    .on('connect', connect)
    .listen(8888,()=>{
        console.log('the proxy server has been listend at port 8888')
    });
```
抓包看看:
![http://7xsi10.com1.z0.glb.clouddn.com/blog-http-connect.png](http://7xsi10.com1.z0.glb.clouddn.com/blog-http-connect.png)

可以看到，这种方式下，agent与proxy之间的流量是明文的，这就会导致一个问题，我试着将服务放到国外vps作为梯子，不访问黑名单网站还好，一访问就被GWF发现，proxy就挂了。那么我们可以在agent与proxy之间加层ssl啊。

```js
https.createServer({
    key: fs.readFileSync(__dirname + '/server/server.key'),
    cert: fs.readFileSync(__dirname + '/server/server.crt')
})
    .on('request', request)
    .on('connect', connect)
    .listen(8888,()=>{
        console.log('the proxy server has been listend at port 8888')
    });
```

最终测试是没问题的:
![http://7xsi10.com1.z0.glb.clouddn.com/node-https-proxy.png](http://7xsi10.com1.z0.glb.clouddn.com/node-https-proxy.png)

## 加入了proxy的agent

通过自定义agent，并实现`addRequest()`方法，有时候的需求是给Agent加上proxy,这个时候就需要在自定义的agent里进行模拟browser行为与proxy进行交流了。

### http-proxy-agent

这种agent只适用于源网站为http的，这种网站可以直接走盲中继

它的实现方式很简单，直接使用`net`或`tls`建立socket,然后与proxy进行交流:
```js
if (this.secureProxy) {
    socket = tls.connect(proxy);
  } else {
    socket = net.connect(proxy);
  }
```
这个模块它是直接放弃`keep-alive`行为的，即没有发送`'proxy-connection':'keep-alive'`,粗暴处理

### https-proxy-agent

该模块为了转发ssl流量，做法就是上面`proxy`小节提到的CONNECT方式：
```js
var hostname = opts.host + ':' + opts.port;
  var msg = 'CONNECT ' + hostname + ' HTTP/1.1\r\n';

  var headers = Object.assign({}, proxy.headers);
  if (proxy.auth) {
    headers['Proxy-Authorization'] =
      'Basic ' + new Buffer(proxy.auth).toString('base64');
  }

  var host = opts.host;
  if (!isDefaultPort(opts.port, opts.secureEndpoint)) {
    host += ':' + opts.port;
  }
  headers['Host'] = host;

  headers['Connection'] = 'close';
  Object.keys(headers).forEach(function(name) {
    msg += name + ': ' + headers[name] + '\r\n';
  });

  socket.write(msg + '\r\n');
```
相当于就是模拟了browser的行为，向proxy发送CONNECT请求来建立隧道。

### 其他proxy
如socks之类的，篇幅有限，这里就不解释了。