---
title: RabbitMQ
date: 2017-09-07 16:15:26
tags: [RabbitMQ,AMQP,RPC]
categories: 
- Nodejs
---

学习RabbitMQ.以Node client实现为例。

RabbitMQ是一个由erlang开发的AMQP（Advanced Message Queue ）的开源实现。目前RabbitMQ基于AMQP 0-9-1。

![http://img.blog.csdn.net/20160310091724939](http://img.blog.csdn.net/20160310091724939)

<!--more-->

## 概览

Publisher和Consumer都通过Socket与RabbitMQ server连接，在Consumer与RabbitMQ server的socket连接上，建立Channel,用于多次消息传输，减少开支。每一个Consumer都有维护消息的queue(可以多个Consumer共用)，exchange用于调度消息的发布，一边Publisher发布消息到Exchange,另一边，各个queue与exchange建立联系并表达自己的兴趣，Exchange根据不同type的调度方式，将消息推送到适当的queue，对于声明了相同的queue的consumer，可以并行负载均衡的处理消息。总的来说:

* 应用解耦: 两边的程序只需实现相应的接口约定，而不用关心内部细节
* 冗余存储器: 可对消息队列和消息进行持久化，一定程度上减少Server突然挂掉带来的数据丢失
* 可扩展: 对同一queue，可以起多个consumer并行处理，而且提供负载均衡算法
* 消息确认: 提供ack机制，保证消息被正确处理
* 顺序和缓冲: 队列机制保证消息处理顺序，缓冲提供性能优化
* 异步通信: Publisher发布的消息可以不被立即处理，可在后面动态加入consumer进行处理
* 分布式集群: 提供容灾能力和消息吞吐量
* 语言无关: 跨语言通信

作为一种进程间通信的方法，对比Node多进程架构，其IPC通道只是本地的进程间通信，相当于是单机集群。而RabbitMQ通过消息队列的形式完成了IPC（包括本地进程，RPC）。
## 简单的queue消息传输

一个最简单的例子是一个Producer,一个Consumer:

![http://www.rabbitmq.com/img/tutorials/python-one.png](http://www.rabbitmq.com/img/tutorials/python-one.png)

因此需要为Channel断言一个明确名字的queue，通过往queue上加入消息，consumer进行处理:

`client.js (Producer):`
```js
const amqp = require('amqplib');

async function client(msg){
    let q = 'hello';
    try{
        let conn = await amqp.connect('amqp://127.0.0.1');
        let ch = await conn.createChannel();

        await ch.assertQueue(q,{durable:false}); //非持久化的队列断言

        ch.sendToQueue(q,new Buffer(msg));
        console.log("client send msg:",msg);
        
        await ch.close();
        await conn.close();
    } catch(err){
        console.log(err);
    }
}
let msg = process.argv.slice(2).join(' ') || 'hello,world.';

client(msg);
```
`server.js (Consumer)`:
```js
async function server(){
    try{
        let conn = await amqp.connect('amqp://127.0.0.1');
        process.once('SIGINT',()=>conn.close());

        let ch = await conn.createChannel();
        await ch.assertQueue('hello',{durable: false});

        await ch.consume('hello',msg=>{
            let body = msg.content.toString();

            console.log("server recieved:",body);
        },{noAck:true});

        console.log("waiting for massages.");
    } catch(err){
        console.error(err);
    }
}

server();
```

## 通过并行&负载均衡的workers处理消息

对于上面的`server.js`完全可以起多个实例，这样相当于是运行了多个worker，对于queue里的每一个消息都有多个worker可以选择，从而做到多个worker并行处理多个消息，实现负载均衡:

![http://www.rabbitmq.com/img/tutorials/python-two.png](http://www.rabbitmq.com/img/tutorials/python-two.png)

在上例基础上修改后的`server.js`:
```js
async function server(){
    try{
        let conn = await amqp.connect('amqp://127.0.0.1');
        process.once('SIGINT',()=>conn.close());

        let ch = await conn.createChannel();
        await ch.assertQueue('hello',{durable: false});
        ch.prefetch(1); //保证worker一次只处理一个task.

        await ch.consume('hello',msg=>{
            let body = msg.content.toString();

            console.log("server recieved:",body);
            let secs = body.split('.').length - 1;
            setTimeout(()=>{
                console.log('done');
                ch.ack(msg);
            },secs*1000);

        },{noAck:false});

        console.log("waiting for massages.");
    } catch(err){
        console.error(err);
    }
}
```
需要留意的几点:

* `ch.prefetch(1);`,RabbitMQ针对消息分发的默认策略只是简单的第n个消息，发生给第func(n)个workder.这个方法可以保证每个worker一次性只处理一个消息。防止负载不均衡。

* `ack`: 开启消息确认，只有当接收到该消息的ack时，才从队列里删除该消息；若某个worker在处理消息过程中挂掉，即未发送ack，则该消息将会重新被发送给其他的worker,防止消息丢失。

* `durable`&`persistence`:分别标记`queue`(在producer和consumer两边都要说明)和具体消息（`ch.sendToQueue(q, new Buffer(msg), {persistent: true});`）的持久化。RabbitMQ可能会突然挂掉，持久化可以在一定程度上保证server重启后，消息队列不丢失。

## Publish/Subscribe

之前的例子是一个消息交由一个worker处理，而这里我们想实现同一个消息分发给多个consumer，这类似于Pub/Sub模式，想想我们的`EventEmmiter`，为同一个事件（类比这里的消息），订阅多个处理器（类比这里的worker)，消息来临，每个处理器都会被执行。

在这种模式中，每个`consumer`都会有一个匿名（或者说随机生成）的queue用于维护消息队列，而处于`producer`和这些`queue`之间的是`exchange`,这相当于一个调度中心，决定将`producer`产生的消息推给哪些`queue`。

![http://www.rabbitmq.com/img/tutorials/python-three-overall.png](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

`exchange`的调度方式有多种:`direct, topic, headers and fanout`；我们这里实现类似于事件订阅，使用`fanout`（广播）。

之前的直接使用`sendToQueue()`发送消息到指定名称的queue，实际上只是一个语法糖，内部使用了默认的`exchange`来调度。

查看所有`exchange`:`sudo rabbitmqctl list_exchanges`

### 为每个consumer生成新的queue:

```js
let q = await ch.assertQueue(null, {exclusive: true});
```

### bindings

binding指的是`exchange`和`queue`间的关系，将`queue`与指定`exchange`建立关系，相当于将这个`queue`加入到了调度中心。

```js
await ch.assertExchange(ex,'fanout',{durable:false})
let q = await ch.assertQueue(null, {exclusive: true});

await ch.bindQueue(q.queue,ex,'');
```

查看当前所有`binding`:`sudo rabbitmqctl list_bindings`:

![http://7xsi10.com1.z0.glb.clouddn.com/list_bindings.png](http://7xsi10.com1.z0.glb.clouddn.com/list_bindings.png)

完整例子:
```js
//emit_log.js
const amqp = require('amqplib');

async function log(){
    let ex = 'logs';

    try {
        let conn = await amqp.connect('amqp://127.0.0.1');
        let ch = await conn.createChannel();

        await ch.assertExchange(ex,'fanout',{durable:false});

        let msg = process.argv.slice(2).join(' ') || 'info: hello,world';
        ch.publish(ex,'',new Buffer(msg));
        console.log("sent:",msg);
        
        await ch.close();
        await conn.close();

    } catch(err){
        console.error(err);
    }
}

log();

// receive_log.js
async function receive(){
    let ex = 'logs';

    try {
        let conn = await amqp.connect('amqp://127.0.0.1');
        let ch = await conn.createChannel();

        await ch.assertExchange(ex,'fanout',{durable:false})
        let q = await ch.assertQueue(null, {exclusive: true});
        console.log("waiting msg in:",q.queue);

        await ch.bindQueue(q.queue,ex,'');

        await ch.consume(q.queue,msg=>{
            console.log('received:',msg.content.toString())
        },{noAck:true});

    } catch(err){
        console.error(err);
    }
}

receive();
```

运行实例:
![http://7xsi10.com1.z0.glb.clouddn.com/fanout.png](http://7xsi10.com1.z0.glb.clouddn.com/fanout.png)

## Routing

上面的例子不够灵活，对比事件订阅，某个eventemmiter实例可以订阅不同的事件（分类），而上面的例子可以理解为粗暴的将所有的消息直接广播给所有consumer，即consumer要么对exchange上的所有消息感兴趣，要么都不感兴趣。

而要表示对部分消息感兴趣，这里通过`direct` type的exchange调度方式实现。

![http://www.rabbitmq.com/img/tutorials/direct-exchange.png](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)

当然同一`routing`的消息可以被多个queue接收，这就类于`fanout`功能。

直接上完整实例，这个例子中两个consumer分别对`info,warnning`和`error`相关的消息感兴趣，`producer`每次可以发送消息到不同路由:

```js
// emit_direct.js
let ex = 'direct_logs';
let args = process.argv.slice(2);

try {
    let conn = await amqp.connect('amqp://127.0.0.1');
    let ch = await conn.createChannel();

    await ch.assertExchange(ex,'direct',{durable:false});

    let msg = args.slice(1).join(' ') || 'Hello World!';
    let severity = (args.length > 0) ? args[0] : 'info';

    ch.publish(ex,severity,new Buffer(msg));
    console.log(" [x] Sent %s: '%s'", severity, msg);

    await ch.close();
    await conn.close();
} catch(err){
    console.error(err);
}

// recieve_direct.js
let ex = 'direct_logs';
let args = process.argv.slice(2);
    
try {
    let conn = await amqp.connect('amqp://127.0.0.1');
    let ch = await conn.createChannel();

    await ch.assertExchange(ex,'direct',{durable:false})
    let q = await ch.assertQueue(null, {exclusive: true});
    console.log("waiting msg in:",q.queue);

    await args.map(arg=>ch.bindQueue(q.queue,ex,arg));

    await ch.consume(q.queue,msg=>{
        console.log(" [x] %s: '%s'", msg.fields.routingKey, msg.content.toString());
    },{noAck:true});

} catch(err){
    console.error(err);
}
```

运行实例:
![http://7xsi10.com1.z0.glb.clouddn.com/direct.png](http://7xsi10.com1.z0.glb.clouddn.com/direct.png)

## Topics

上面的例子还可以更加灵活，对于一个消息，可以从多方面去描述，然后不同的queue表达对某方面感兴趣，而对于其他方面进行忽略：

![http://www.rabbitmq.com/img/tutorials/python-five.png](http://www.rabbitmq.com/img/tutorials/python-five.png)

如图，Q1表示对颜色为orange的消息感兴趣，Q2对物种是兔子，跑的慢的感兴趣，而不在乎颜色。

`*` 代指一个单词，`#`代指任意个单词。

代码方面，与上例类似，不过是`exchange`类型为`topic`,`routing keys`格式的变化:
```js
ch.assertExchange(ex, 'topic', {durable: false});
ch.publish(ex, 'anonymous.info', new Buffer(msg));
```

## RPC

可以通过RabbitMQ实现RPC

### callback queue

`reply_to`（即为回调指定队列）有两种情况，一种是针对每一个request，都创建（随机生成）一个新的queue:
![http://www.rabbitmq.com/img/tutorials/python-six.png](http://www.rabbitmq.com/img/tutorials/python-six.png)
另一种是每个request使用同一queue（使用固定命名即可）。

第二种更为高效，但是需要解决一个问题，当同时调用多个request的时候，会有多个response,如何使它们匹配。解决方案是加入`Correlation Id`，其对于每一个request都是独一无二的，每个client接收到消息后进行`Correlation Id`判断:

```js
await ch.consume(q.queue,msg=>{
    if (msg.properties.correlationId == corr) {
        //...
    }
},{noAck:false});

ch.sendToQueue('rpc_server',
new Buffer(num.toString()),
{ correlationId: corr, replyTo: q.queue });
```

完整实例：
```js
// rpc_client.js
async function client(){
    let args = process.argv.slice(2);
    let corr = generateUuid();
    let num = parseInt(args[0]);
    try {
        let conn = await amqp.connect('amqp://127.0.0.1');
        let ch = await conn.createChannel();
        let q = await ch.assertQueue('rpc_client',{exclusive: false});

        console.log(' [x] Requesting fib(%d)', num);
        console.log(q.queue)

        await ch.consume(q.queue,msg=>{
            if (msg.properties.correlationId == corr) {
                console.log(' [.] Got %s', msg.content.toString());
                ch.ack(msg)
                setTimeout(()=>{ conn.close(); process.exit(0) }, 2500);
            }
        },{noAck:false});

        ch.sendToQueue('rpc_server',
        new Buffer(num.toString()),
        { correlationId: corr, replyTo: q.queue });

    } catch(err){
        console.error(err);
    }
}

function generateUuid() {
    return Math.random().toString() +
           Math.random().toString() +
           Math.random().toString();
}

// rpc_server.js
async function server(){
    try {
        let conn = await amqp.connect('amqp://127.0.0.1');
        let ch = await conn.createChannel();
        process.once('SIGINT',()=>conn.close());

        let q = await ch.assertQueue('rpc_server');
        ch.prefetch(1);
        console.log(' [x] Awaiting RPC requests');

        await ch.consume(q.queue,msg=>{
            let n = parseInt(msg.content.toString());
            
            console.log(" [.] fib(%d)", n);
    
            let r = fibonacci(n);
    
            ch.sendToQueue(msg.properties.replyTo,
            new Buffer(r.toString()),
            {correlationId: msg.properties.correlationId});
    
            ch.ack(msg);
        },{noAck:false});


    } catch(err) {
        console.error(err);
    }
}

server();

function fibonacci(n) {
    if (n == 0 || n == 1)
        return n;
    else
        return fibonacci(n - 1) + fibonacci(n - 2);
}
```

同意可以启动多个server实例来并行进行计算任务：

![http://7xsi10.com1.z0.glb.clouddn.com/rpc.png](http://7xsi10.com1.z0.glb.clouddn.com/rpc.png)

注意点：

* `exclusive`: 这里不能为true,不然会出现同时启动多个client失败;
* `ack`:　需要开启ack，否则会出现后续的client调用永远处于等待状态。

目前发现如果对所有client共享一个queue，多个request竟然是串行执行的，即需要前一个client关闭后，后一个client才会返回结果。经过排查，`server`端是正常接收到多个request的，理论来说应该是及时向client回复了信息的。问题出现在client这里。而对每个client使用不同的queue是没有这个问题的。

待解决...

## 跨语言通信

*注：实例来自《Node.js实战》*

上面的例子都是在Node进程之间进行的，RabbitMQ本身就用于横向扩展的集群服务之间的消息通信，当然支持不同主机，不同语言服务之间的通信。

一个简单的例子:`server.py`作为consumer:
```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
    host="localhost"
))

channel = connection.channel()

channel.queue_declare(queue="hello")
print "waiting for massage from queue hello"

def callback(ch,method,properties,body):
    print "Received: %r" % (body,)

channel.basic_consume(callback,queue="hello",no_ack=True)

channel.start_consuming()
```
producer还是我们之前第一个例子的`client.js`，运行结果：

![http://7xsi10.com1.z0.glb.clouddn.com/node_py.png](http://7xsi10.com1.z0.glb.clouddn.com/node_py.png)


## 其他

篇幅有限，这篇文章就只做一些基础的介绍，还有诸如与HTTP方案对比，RabbitMQ集群等等，后续文章进行总结。

参考:

* RabbitMQ官网：[http://www.rabbitmq.com/](http://www.rabbitmq.com/)
* AMQP官网: [https://www.amqp.org](https://www.amqp.org)
* amqp.node: [http://www.squaremobius.net/amqp.node/](http://www.squaremobius.net/amqp.node/)