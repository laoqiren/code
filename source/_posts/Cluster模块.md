---
title: Cluster模块
date: 2016-11-27 23:15:15
tags: [负载均衡,集群,多进程]
categories: 
- nodeJS
---

Node的一大优点就是异步IO,适合I/O密集型的高并发业务，但是它是单线程的，当面对CPU密集型业务的时候，性能就会出现瓶颈。但是幸运的是，子进程能够解决这个问题，与HTML5的Web Worker类似，通过主进程管理一系列的子进程来利用多核CPU以实现集群的功能。

关于集群是一个比较大的话题。Node原生Child_process就可以实现集群，但是要处理好一系列问题，如自动重启，自杀信号，负载均衡等问题，需要我们做大量的工作。从Node V0.8开始，其内置一个核心模块Cluster,帮助我们更加方便的实现单机集群。

<!--more-->

### Example:
```js
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  // Fork workers.
  for (var i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`worker ${worker.process.pid} died`);
  });
} else {
  // Workers can share any TCP connection
  // In this case it is an HTTP server
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');
  }).listen(8000);
}
```
这样就会根据计算机cpu情况，创建多个子进程来处理服务，对于TCP服务，实现多个子进程共享端口。

or一种分离worker逻辑的方式:

master.js:
```js
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

cluster.setupMaster({
        exec:"worker.js"
});
```
worker.js:
```js
http.createServer((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');
  }).listen(8000);
```

### IPC进程间通信

master和worker之间通过IPC通道通信，在Node中IPC通道由libuv提供，在Windows下由命名管道(named pipe)实现，*nix系统采用Unix Domain Socket实现。而经过libuv抽象后，就是十分简单的message事件和send()方法。

---
父进程在创建子进程之前，先创建IPC通道并对其进行监听，再创建出子进程，并通过环境变量(NODE_CHANNEL_FD)告诉子进程这个IPC通道的文件描述符。子进程在启动过程中，根据文件描述符去连接这个IPC通道。

IPC通道被抽象为流对象。

### 句柄传递

当我们直接通过创建多个worker来监听同一端口时,会报错:
```
Error: listen EADDRINUSE
```
由于一个工作进程已经监听某个端口，其余进程再次监听将会抛出错误，那要如何实现监听同一端口呢？

#### 句柄

句柄是一种可以用来标识资源的引用，内部包含了指向对象的文件描述符。

#### 句柄发送与还原

Child_process在发送消息到IPC之前，会将消息进行封装，一个句柄文件描述符，一个message对象,最终子进程收到消息后做如下操作:

```js
function(message,handle,emit){
	var server = new net.Server();
	server.listen(handle,()=>{
		emit(server);
	});
}
```
可见，Node进程之间传递的并非是真正地传递socket对象，而是进行消息传递。

#### SO_REUSEADDR端口重用

```js
setsockopt(tcp->io_wather.fd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on))
```
通过send()发送的句柄还原出来的服务，文件描述符相同，就可以监听在同一端口了。

### Cluster实现原理

Cluster实际上就是child_process和net模块的组合应用。

文章开头的例子，Cluster会启动TCP服务器，然后fork子进程时会将TCP服务的socket文件描述符发送给worker,然后通过句柄还原，拿到该文件描述符，然后启动http服务，并监听在这个文件描述符上面。

### 部分常用Cluster事件及API

#### master相关

**fork**
当一个新worker被fork时，可以监听该事件来设置timeOut,用以测试网络连接是否正常，如果网络连接超时就报错或者杀掉子进程。


```js
var timeouts = [];
function errorMsg() {
  console.error('Something must be wrong with the connection ...');
}

cluster.on('fork', (worker) => {
  timeouts[worker.id] = setTimeout(errorMsg, 2000);
});
cluster.on('listening', (worker, address) => {
  clearTimeout(timeouts[worker.id]);
});
cluster.on('exit', (worker, code, signal) => {
  clearTimeout(timeouts[worker.id]);
  errorMsg();
});
```

**listening**

worker调用listen()即共享Socket后，发送一条listening消息给master,master触发

**disconnect**

当master和worker端口IPC通道后触发，这个事件和exit事件触发之间是有一定的时间间隔的，这样可以让worker平稳安全的关闭，在进程被杀掉之前，可以close一些服务，如长连接这种断开需要一定时间的服务。close完后再exit

当worker在超过一定时间后还没有挂掉，我们可以强制性地将其over掉:

cluster.js:
```js
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

cluster.setupMaster({
        exec:"worker.js"
});
console.log("master start");
for(var i=0;i<numCPUs;i++){
        cluster.fork();
}
cluster.on('listening',function(worker,address){
        setTimeout(()=>worker.send('kill yourself'),2000);
        console.log(`listening:worker ${worker.process.pid},address:${address.address}:${address.port}`);
});
cluster.on('exit',(worker,code,signal)=>{
        console.log(`worker ${worker.process.pid}died\n`);
});
cluster.on('message',(worker,message,handle)=>{
        console.log(`message:${message}`);
});
function eachWorker(cb){
        for(var id in cluster.workers){
                cb(cluster.workers[id]);
        }
}
var timeout = setTimeout(()=>{
        eachWorker((worker)=>{
                worker.kill('SIGHUP');
                console.log('killed by master');
        });
},5000);
eachWorker((worker)=>{
	worker.on('exit',(code,signal)=>{
            clearTimeout(timeout);
            if(signal){
                    console.log(`worker ${worker.process.pid} ws killed by signal ${signal}`);
            }else if(code!==0){
                    console.log(`worker${worker.process.pid} ws killed by code ${code}`);
            }
    });
});
```
worker.js:
```js
const http = require('http');
const cluster = require('cluster');
const worker = cluster.worker;
http.createServer((req,res)=>{
 res.writeHead(200);
 res.end("hello world");
}).listen(8099);
process.on('message',(message,handle)=>{
        console.log(`${process.pid} accept:${message}`);
        if(message === 'kill yourself'){
                setTimeout(()=>process.exit(1),50000);
        }

});
```

运行结果:
![cluster](http://7xsi10.com1.z0.glb.clouddn.com/cluster.png)

**message**

这个不用多说，当从worker发来消息时触发,但是在node V6.0之前回调函数是没有worker对象传入的。可以进行参数判断:

```js
cluster.on('message', function(worker, message, handle) {
  if (arguments.length === 2) {
    handle = message;
    message = worker;
    worker = undefined;
  }
  // ...
});
```

**cluster.isMaster和cluster.isWorker**

用于判断当前进程是master还是worker,当进程是子进程时其环境变量会存在NODE_UNIQUE_ID，通过这个判断:
```js
cluster.isWorker = ('NONE_UNIQUE_ID' in process.env);
```

**cluster.setupMaster()**
还记得文章开头fork子进程的方式吗，就是用的这个方法，通过设置exec，来指定运行指定的文件来生成子进程，这样可以实现主进程和子进程的逻辑分离。

**cluster.workers**
获取由cluster生成的所有子进程，通过id访问，代码示例见上述disconnect部分。

**cluster.schedulingPolicy**
关于负载均衡，后面会单独讲
#### worker相关

**worker.kill()**
杀掉子进程，可以通过状态或者信号，一般自杀用状态码，正常为0,由master来杀掉的话用信号，如上面的代码那要可以进行一个判断:

```js
worker.on('exit',(code,signal)=>{
        clearTimeout(timeout);
        if(signal){
                console.log(`worker ${worker.process.pid} ws killed by signal ${signal}`);
        }else if(code!==0){
                console.log(`worker${worker.process.pid} ws killed by code ${code}`);
        }
});
```

**worker.exitedAfterDisconnect**
可以用来判断worker是否是自杀的:
```js
cluster.on('exit', (worker, code, signal) => {
  if (worker.exitedAfterDisconnect === true) {
    console.log('Oh, it was just voluntary – no need to worry');
  }
});
```
**需要注意的是，这些API和事件都是通过cluster模块来调用的，跟在worker中自行调用一些方法，如process.kill(),process.on('message',cb);不是一回事。但是它们都差不多。

**其余常用的还有许多API,这里只是提了一些需要注意点的东西，详细的API移步node官网API文档,由于cluster的东西都是基于process和process_child的，可以结合它们一起理解**

### 负载均衡问题

当用原生Child_process实现集群时，我们还需要处理负载均衡问题。Node通过CPU繁忙度来给worker分配任务。但是有可能某个worker的CPU空闲，但是I/O却比较繁忙，但是这个时候依然会给这个worker分配较多请求。这就出现负载不均衡情况。

Node v0.11中提供一种Round-Robin的策略（调度）来进行分配任务。主进程接受连接，在N个工作进程中，每次选择第**i=(i+1)mod N**个进程来发送连接。Node默认选择Round-Robin方式

```js
cluster.schedulingPolicy = cluster.SCHED_RR;//启用RR
cluster.schedulingPolicy = cluster.SCHED_NONE;//不启用RR
```
或者在环境变量中设置NODE_CLUSTER_POLICY:
```
export NODE_CLUSTER_POLICY=rr;
export NODE_CLUSTER_POLICY=none;
```

### 啦啦啦，我是PM2

pm2 是一个进程管理工具，可以做到不间断重启、负载均衡、集群管理等，比forever更强大。利用pm2可以做到 no code but just config 实现应用的cluster。 

前面的一篇关于前后端分离实践的文章，我写了一个论坛，而我正是使用PM2来部署服务的，鉴于买的阿里云服务器学生版，CPU只有一个。所以没用到它的cluster服务。

**Coding is my life**
