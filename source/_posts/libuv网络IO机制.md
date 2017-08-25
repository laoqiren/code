---
title: libuv网络I/O机制
date: 2017-07-27 14:18:17
tags: [异步,event loop,libuv]
categories: 
- Nodejs
---

两条故事线去探索http.Server从`listen()`到`connection`事件的触发其背后的原理，主要是为了理解libuv在网络I/O方面的异步实现。一条故事线看TCP handle的I/O观察者是怎样加入到event loop的观察者队列，另一条故事线看隐藏于背后的event loop在liunx下如何利用系统的`epoll`机制注册并收集events从而调用观察者回调。

<!--more-->

源码解读以linux系统为方向。我们知道libuv对不同平台的内部网络I/O异步机制进行了抽象，如Windows的IOCP,FreeBSD下的kqueue,linux下的epoll等。

## 从js层面来到C++层面


`http.Server`类继承`net.Server`类,以`Server.listen()`为例，其实调用的是`net.Server.listen()`,实际调用的是`setupListenHandle()`,包括`createServerHandle()`到C++层面创建handle和调用C++层面定义的`listen()`方法:
```js
rval = createServerHandle(address, port, addressType, fd);
this._handle = rval;
this[async_id_symbol] = getNewAsyncId(this._handle);
this._handle.onconnection = onconnection;
this._handle.owner = this;
var err = this._handle.listen(backlog || 511);
nextTick(this[async_id_symbol], emitListeningNT, this); //触发listening事件
```
在`emitListeningNT`里触发`listening`事件,其中`handle`是C/C++内建模块`tcp_wrap.cc`里定义的,这里还定义了`onconnection`，这个会在C/C++层面合适的时候调用，而`onconnection()`里会触发`connection`事件。

说到handle创建，`onconnection`的调用,那就要开始我们的第一条故事线:向event loop里加入观察者。

## 故事线一：向event loop观察者队列里加入观察者

### 观察者的产生
首先`TCPWrap`类的constructor:
```cpp
TCPWrap::TCPWrap(Environment* env, Local<Object> object)
    : ConnectionWrap(env,
                     object,
                     AsyncWrap::PROVIDER_TCPWRAP) {
  int r = uv_tcp_init(env->event_loop(), &handle_);
  CHECK_EQ(r, 0);
  UpdateWriteQueueSize();
}
```
我们看到调用了`uv_tcp_init()`用于设置`handle_`，比如将`env.event_loop()`赋值给`handle_->loop`。

`TCPWrap`继承`ConnectionWrap`：
```cpp
class TCPWrap : public ConnectionWrap<TCPWrap, uv_tcp_t>{
    //
}
```
然后这里这里的`handle_`就是在`ConnectionWrap`里定义的:
```cpp
namespace node {

template <typename WrapType, typename UVType>
class ConnectionWrap : public StreamWrap {
  ...
  UVType handle_;
};
```
可以看到这里的`handle_`类型根据模板参数`UVType`来,这样`&wrap->handle_`就是一个`uv_tcp_t`观察者类型的指针了。

关于`uv_tcp_t`:
```cpp
struct uv_tcp_s {
  UV_HANDLE_FIELDS
  UV_STREAM_FIELDS
  UV_TCP_PRIVATE_FIELDS
};
```
`UV_HANDLE_FIELDS`定义如下:

![http://7xsi10.com1.z0.glb.clouddn.com/uv_handle_fileds.png](http://7xsi10.com1.z0.glb.clouddn.com/uv_handle_fileds.png)

在`UV_STREAM_FIELDS`的`UV_STREAM_PRIVATE_FIELDS`定义如下:
![http://7xsi10.com1.z0.glb.clouddn.com/UV_STREAM_PRIVATE_FIELDS.png](http://7xsi10.com1.z0.glb.clouddn.com/UV_STREAM_PRIVATE_FIELDS.png)

其中`io_watcher`就是我们所说的I/O观察者了，`connection_cb`将是最终C++进入js层面需要调用的，后面会详解。

### TCPWrap::Listen

这便是js层面调用的`TCP.listen`
```cpp
void TCPWrap::Listen(const FunctionCallbackInfo<Value>& args) {
  TCPWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap,
                          args.Holder(),
                          args.GetReturnValue().Set(UV_EBADF));
  int backlog = args[0]->Int32Value();
  int err = uv_listen(reinterpret_cast<uv_stream_t*>(&wrap->handle_),
                      backlog,
                      OnConnection);
  args.GetReturnValue().Set(err);
}
```
我们看到这里它调用了`uv_listen`，而参数有强制转换为`uv_stream_t`的`uv_tcp_t`指针，

在`stream.c`里会根据`handle`具体类型，调用不同的函数:
```cpp
int uv_listen(uv_stream_t* stream, int backlog, uv_connection_cb cb) {
  int err;
  switch (stream->type) {
  case UV_TCP:
    err = uv_tcp_listen((uv_tcp_t*)stream, backlog, cb);
    break;
  case UV_NAMED_PIPE:
    err = uv_pipe_listen((uv_pipe_t*)stream, backlog, cb);
    break;

  default:
    err = -EINVAL;
  }
  if (err == 0)
    uv__handle_start(stream); //会执行activehandles++
  return err;
}
```
这里调用｀uv_tcp_listen`：
```cpp
if (listen(tcp->io_watcher.fd, backlog))
    return -errno;

tcp->connection_cb = cb;
tcp->flags |= UV_HANDLE_BOUND;

/* Start listening for connections. */
tcp->io_watcher.cb = uv__server_io;
uv__io_start(tcp->loop, &tcp->io_watcher, POLLIN);

return 0;
```
`uv__io_t io_watcher.cb`被设置为`uv__server_io`,该函数会由event loop调用，后面会详解。而`tcp->connection_cb = cb`被设置为cb,即`onConnection`。

`io_watcher`为`uv__io_t`类型:
```cpp
struct uv__io_s {
  uv__io_cb cb;
  void* pending_queue[2];
  void* watcher_queue[2];
  unsigned int pevents;
  unsigned int events;
  int fd;
  UV_IO_PRIVATE_PLATFORM_FIELDS
};
```

再进入`uv__io_start()`:
```cpp
void uv__io_start(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  w->pevents |= events;
  maybe_resize(loop, w->fd + 1);

  if (QUEUE_EMPTY(&w->watcher_queue))
    QUEUE_INSERT_TAIL(&loop->watcher_queue, &w->watcher_queue);

  if (loop->watchers[w->fd] == NULL) {
    loop->watchers[w->fd] = w;
    loop->nfds++;
  }
}
```
可以看到这里，将events赋值给`w->pevents`，就是说加入到pending events里去，就是它对events事件感兴趣。这个会在后面`epoll_wait()`用到。

`uv_tcp_t`类型的`_handle`的`io_wathcer`就加入到了`_handle->loop`（实际上就是event loop)的`watcher_queue`里。

至此，添加I/O观察者的任务就完成了，该函数执行完毕，`TCP.listen()`的调用就介绍，就会返回到js层面,去触发`listening`事件，可见，这个过程是同步执行的，那么`listening`事件真正的意义就是标志着I/O观察者成功加入到事件循环了。

而要谈真正意义上的异步，还得从另一条故事线出发，那便是隐藏于背后的`event loop`。

## 故事线二：event loop

还记得我上一篇文章吗？在`node.cc`的最后一个`inline Start()`里:
```cpp
{
    SealHandleScope seal(isolate);
    bool more;
    do {
      v8_platform.PumpMessageLoop(isolate);
      more = uv_run(env.event_loop(), UV_RUN_ONCE);


      if (more == false) {
        v8_platform.PumpMessageLoop(isolate);
        EmitBeforeExit(&env);
        more = uv_loop_alive(env.event_loop());
        if (uv_run(env.event_loop(), UV_RUN_NOWAIT) != 0)
          more = true;
      }
    } while (more == true);
  }
```
`uv_run()`开启了event loop。

然后我们进入到`uv_run()`：
```cpp
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

//这里就是那个被称作event loop的while loop
  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```
这里不断地循环，不断地判断event loop是否保持active，如果有handle存在，那么它将一直在我们背后运行。

而这里与我们有关的是`uv__io_poll()`，而poll这个词正暴露了它的内部实现机制`epoll`

首先进入了一个循环:
```cpp
while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
    e.events = w->pevents;
    e.data = w->fd;

    if (w->events == 0)
      op = UV__EPOLL_CTL_ADD;
    else
      op = UV__EPOLL_CTL_MOD;
    if (uv__epoll_ctl(loop->backend_fd, op, w->fd, &e)) {
      if (errno != EEXIST)
        abort();

      assert(op == UV__EPOLL_CTL_ADD);
        abort();
    }

    w->events = w->pevents;
  }
```
每次从观察者队列里取出队头的观察者队列，然后根据它才取出`w`(实际的`io_wathcer`)；并取出观察者感兴趣的pending events和与之绑定的`fd`。

然后调用`epoll`机制三大方法之一的`uv__epoll_ctl`去注册io观察者感兴趣的pending状态的events。

接着又会进去一个无限循环`for(;;)`,首先调用`uv__epoll_wait`等待上面注册的事件发生:
```cpp
nfds = uv__epoll_wait(loop->backend_fd,
                            events,
                            ARRAY_SIZE(events),
                            timeout);
```
`epoll`机制会在该`fd`产生观察者感兴趣的事件发生后返回，收集到事件会放到`events`数组里。

接着在这个循环里依次取出`events`(由epoll产生):
```cpp
for (i = 0; i < nfds; i++) {
    pe = events + i;
    fd = pe->data;
    w = loop->watchers[fd];
    ...
    w->cb(loop, w, pe->events);
}
```
针对每个事件调用`w->cb(loop, w, pe->events);`而这个`cb`我们之前在｀uv_tcp_listen()`里进行了设置:
```cpp
tcp->io_watcher.cb = uv__server_io;
```
然后由于`timeout`在event loop里被设置为0，即表示立即返回，当下一次event loop来临时，根据epoll机制：

---
LT模式: 当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。

这样就不用担心两次事件循环之间的时间有事件来不及处理了。

这样这个与I/O有关的epoll操作结束，直到下一个event loop来临，又会去`取观察者->注册感兴趣的事件->找epoll询问产生的事件->执行回调`。

## 回调进入js层面

顺藤摸瓜来到`uv_server_io`:
```cpp
uv__io_start(stream->loop, &stream->io_watcher, POLLIN);
while (uv__stream_fd(stream) != -1) {
    assert(stream->accepted_fd == -1);
    err = uv__accept(uv__stream_fd(stream));
    if (err < 0) {
      if (err == -EAGAIN || err == -EWOULDBLOCK)
        return;

      if (err == -ECONNABORTED)
        continue;  /* Ignore. Nothing we can do about that. */

      if (err == -EMFILE || err == -ENFILE) {
        err = uv__emfile_trick(loop, uv__stream_fd(stream));
        if (err == -EAGAIN || err == -EWOULDBLOCK)
          break;
      }

      stream->connection_cb(stream, err);
      continue;
    }
```
可以看到在调用回调函数之前，再次调用了`uv__io_start()`，这样就可以继续监听其他连接的到来了。

最后我们来看看`stream->connection_cb(stream,err)`的调用:
```cpp
WrapType* wrap_data = static_cast<WrapType*>(handle->data);
wrap_data->MakeCallback(env->onconnection_string(), arraysize(argv), argv);
```
至此，通过在js层面通过`Server.handle.onconnection`设置的`onconnection`即这里的`env->onconnection_string()`就会被调用了。而在`onconnection`里：
```js
var socket = new Socket({
    handle: clientHandle,
    allowHalfOpen: self.allowHalfOpen,
    pauseOnCreate: self.pauseOnConnect
  });
socket.readable = socket.writable = true;


self._connections++;
socket.server = self;
socket._server = self;
...
self.emit('connection', socket);
```
这样我们在最表层见到的`connection`事件，才会触发，再经过http的一层封装，才会触发`request`事件。

nice job!

## 总结
可以看到，调用`listen`是一个同步过程，即会调用C++层面的`Listen`,而这个`Listen`的作用就是将io观察者加入到`loop->wathcer_queue`里，完成后才会返回。

然后在V8执行js代码的背后，通过执行`uv_run()`开始`event loop`,背后的io异步，实际上是利用的`epoll`机制，里面是一个无限循环，`epoll_wait()`不断地监听`watcher_queue`里每个观察者期待的事件是否发生，如果发生，就生成`events`数组，`events`数组可能来自不同的`fd`，针对每一个`fd`分别调用它们所属观察者的回调函数。接着，进入下一个for循环。

可以看到暴露给应用层的异步网络I/O，内部实现还是同步的，因为epoll这种机制虽然是非阻塞的I/O多路复用，但是需要不断地去轮询事件的产生或者休眠，相当于还是阻塞了process。不过这些都是发生在V8外的event loop内部的，v8的线程没有被阻塞。而event loop的处理方式是要求`epoll_wait()`立即返回，通过循环的方式去调用它，这就有点类似于`read`方式了。