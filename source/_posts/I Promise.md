---
title: I Promise
date: 2016-08-28 21:19:07
tags: [异步编程]
categories: 
- JavaScript
---

正如标题那样，Promise对未来某个事件做出承诺，比如`如果成功，那么...；如果失败，那么...`。是一种相对于回调和事件订阅更高级的抽象，本文就从其起源，实现，面临的问题等方面分析Promise。

<!--more-->

Promise被广为所知是来自于Jquery的Promise/Deferred：
# Jquery Promise/Deferred

`$.when()`:如果向 jQuery.when 传入一个延迟对象，那么会返回它的 Promise 对象(延迟方法的一个子集)。可以继续绑定 Promise 对象的其它方法，例如， defered.then 。当延迟对象已经被受理（resolved）或被拒绝(rejected）（通常是由创建延迟对象的最初代码执行的），那么就会调用适当的回调函数。

有多个延迟对象传递给jQuery.when ，该方法返回一个新的“宿主”延迟对象，跟踪所有已通过Deferreds聚集状态。 当所有的延迟对象被受理（resolve）时，该方法才会受理它的 master 延迟对象。当其中有一个延迟对象被拒绝（rejected）时，该方法就会拒绝它的 master 延迟对象。如果 master 延迟对象被受理（resolved），那么会传入所有延迟对象的受理（resolved）值，这些延迟对象指的就是传给 jQuery.when 的参数。
```js
let dtd = $.Deferred();
    function wait(dtd){
        let tasks = function(){
            console.log("执行完毕");
            dtd.resolve("hello","恩恩","你在哪儿");
        }
        setTimeout(tasks,5000);
        return dtd;
    }
    $.when(wait(dtd))
    .done((a1,a2,a3)=>console.log(a1,a2,a3))
    .fail(()=>console.log("完了"))

    dtd.resolve("lfjaljfl","hfaljfla","hhhhh")
```
上述做法有问题，因为`wait`方法暴露出的是Deferred对象，含有可以改变内部状态的方法，可以被外界代码随时用于改变状态，如上面的例子，在外面调用了`resolve`方法，则一开始就会调用`done`方法的回调，后面`wait`内异步操作执行完毕不会再执行`done`的回调了。

换种方式：
```js
function wait1(){
    let dtd = $.Deferred();
    let tasks = function(){
        console.log("执行完毕");
        dtd.resolve("hello","恩恩","你在哪儿1");
    }
    setTimeout(tasks,5000);
    return dtd.promise();
}
function wait2(){
    let dtd = $.Deferred();
    let tasks = function(){
        console.log("执行完毕");
        dtd.resolve("hello","恩恩","你在哪儿2");
    }
    setTimeout(tasks,6000);
    return dtd.promise();
}

$.when(wait1(),wait2())
.then((r1,r2)=>{
    return "r1.concat(r2)"
}).then(rs=>console.log(rs)) 
 //["hello", "恩恩", "你在哪儿1"] ["hello", "恩恩", "你在哪儿2"]
```
deferred.promise() 方法允许一个异步函数阻止那些干涉其内部请求的进度（progress）或状态（status）的其它代码。Promise （承诺）对象仅会暴露那些需要绑定额外的处理或判断状态的延迟方法(then, done, fail, always, pipe, progress, 和 state)时，并不会暴露任何用于改变状态的延迟方法(resolve, reject, notify, resolveWith, rejectWith, 和 notifyWith).

# Promise/A

09年，Promise/Deferred模式被抽象为一个提议草案发布在CommonJS规范中，后来抽象出了一系列模型，包括`Promise/A`,具体标准请看[https://promisesaplus.com/](https://promisesaplus.com/)

![https://mdn.mozillademos.org/files/8633/promises.png](https://mdn.mozillademos.org/files/8633/promises.png)

## Pub/Sub实现

这个就可以看做是事件订阅模式和Promise的小小联系。

```js
const EventEmitter = require("events");
const util = require("util");

let Promise = function(){
    EventEmitter.call(this);
};
util.inherits(Promise,EventEmitter);

Promise.prototype.then = function(fulfilledHandler,errorHandler,progressHandler){
    if(typeof fulfilledHandler === 'function'){
        this.once('success',fulfilledHandler);
    }
    if(typeof errorHandler === 'function'){
        this.once('error',errorHandler);
    }
    if(typeof progressHandler === 'function'){
        this.once('progress',progressHandler);
    }
    return this;
}

let Deferred = function(){
    this.state = 'unfulfilled';
    this.promise = new Promise();
}
Deferred.prototype.resolve = function(obj){
    this.state = 'fulfilled';
    this.promise.emit('success',obj);
};

Deferred.prototype.reject = function(err){
    this.state = 'failed';
    this.promise.emit('error',err);
}

Deferred.prototype.progress = function(data){
    this.promise.emit('progress',data);
}
```

但是，面临一个问题，那就是链式调用问题，多次调用`then()`方法，发现只会针对`resolve()`传入的value多次触发success事件,比如:
```js
function promisify(){
    let dtd = new Deferred();
    setTimeout(()=>dtd.resolve("hello1"),3000);
    return dtd.promise;
}

function promisify2(){
    let dtd = new Deferred();
    setTimeout(()=>dtd.resolve("hello2"),3000);
    return dtd.promise;
}

// 并不能正常完成序列执行的链式调用。
promisify()
    .then(result=>{
        console.log(result)
        return promisify2();
    })
    .then(result=>{
        console.log(result);
    })
// hello1 hello1
```

## 链式Promise

我们要实现像jq的Promise/Deferred那样链式执行，主要处理两种情况：

1. 当`fulfilledHandler`返回非promise时，将结果作为下一个fulfilledHandler的参数；
2. 当`fulfilledHandler`返回新的promise时，等待新的promise完成后执行后续任务;

我在《深入浅出Node.js》书上的实现基础上做了一些改进和完善：
```js
let Deferred = function(){
    this.promise = new Promise();
}

Deferred.prototype.resolve = function(...obj){
    let promise = this.promise;
    let handler;
    while((handler=promise.queue.shift())){
        if(handler && handler.fulfilled) {
            let ret = handler.fulfilled(...obj);
            if(ret && ret.isPromise){
                /*
                如果返回的是新的Promise，将deferred的promise指向新的promise,并转交任务队列，然后终止循环执行任务队列，等到控制新的promise状态的deferred对象调用了resolve才再一次来到循环。
                */
                ret.queue = promise.queue;
                this.promise = ret;
                return;
            }
            obj = [ret]; //如果返回非promise,就继续执行后续任务，并将返回值作为下一个任务的回调参数。
        }
    }
}

Deferred.prototype.reject = function(){}　//省略

let Promise = function(){
    this.queue = [];
    this.isPromise = true;
}
Promise.prototype.then = function(fulfilledHandler,errorHandler,progressHandler){
    let handler = {};
    if(typeof fulfilledHandler === 'function'){
        handler.fulfilled = fulfilledHandler;
    }
    if(typeof errorHandler === 'function'){
        handler.error = errorHandler;
    }
    // 每次调用then,都将回调加入到队列里
    this.queue.push(handler);
    return this;
}

//对Promise/Deferred的封装，符合规范的语法

function promise(resolver) {
    if (typeof resolver !== "function") {
        throw new TypeError("resolver must be a function.");
    }
    var deferred = new Deferred();
    try {
        resolver(deferred.resolve.bind(deferred), 
            deferred.reject.bind(deferred));
    } catch (reason) {
        deferred.reject(reason);
    }
    return deferred.promise;
}
```

可以看到实现思想还是`Promise/Deferred`，`Deferred`来管理状态，每个Promise实例里边维护一个任务队列，通过`then`调用将任务加入到队列里边，`Deferred`调用`resolve`或`reject`改变状态，并挨个执行任务队列里的任务，但是遇到某个任务返回新的Promise时，需要暂停任务执行，将任务队列转交，并等待新的异步任务完成后继续执行任务。

最后对`Promise/Deferred`进行了封装，形成最终与标准语法相近的`promise`构造函数。

上述例子只是一个粗略的实现，还有许多细节问题，比如错误处理，在`fulfilled`函数里抛出错误的处理，`catch`实现，其他重要接口，如`Promise.all(),Promise.race()`等。

## 存在的问题:

1. 对于异步操作API，需要进行封装才能成为Promise对象，可以借助第三方API进行快速封装。

2. 太复杂的异步场景，会使得链式过长。

## 第三方实现

bluebird: [https://github.com/kriskowal/q](https://github.com/kriskowal/q)

Q: [https://github.com/kriskowal/q](https://github.com/kriskowal/q)

when: [https://github.com/cujojs/when](https://github.com/cujojs/when)

## 官方标准实现

参照ES6 `Promise`标准。
