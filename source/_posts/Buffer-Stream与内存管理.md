---
title: Buffer/Stream与内存管理
date: 2017-06-29 21:40:28
tags: [buffer,stream]
categories: 
- nodeJS
---

这篇文章将开启我的Node源码解读系列，结合源码和官方文档，探索其背后的原理。首先想谈谈Stream这强大而又容易误解的功能，文章主要从Buffer/Stream与内存分配的关系角度来分析。

<!--more-->

# Buffer

## 单次分配大小限制
一次性分配Buffer的大小限制(src/node_buffer.h)：
```cpp

static const unsigned int kMaxLength =
    sizeof(int32_t) == sizeof(intptr_t) ? 0x3fffffff : 0x7fffffff;
```
`Buffer.from(), Buffer.alloc(),　Buffer.allocUnsafe()`这些用于构造Buffer对象的方法分配内存前都会检查size参数:
```js
function assertSize(size) {
...
  else if (size > binding.kMaxLength) {
    err = new RangeError('"size" argument must not be larger ' +
                         'than ' + binding.kMaxLength);
  }
...
```


## C++直接分配策略
由C++直接分配内存，实际上是由ArrayBuffer分配内存，在此基础上建立Uint8Array的View,Buffer继承该View。

**Buffer.alloc()**
```js
Buffer.alloc = function(size, fill, encoding) {
  assertSize(size);
  if (size > 0 && fill !== undefined) {
    if (typeof encoding !== 'string')
      encoding = undefined;
    return createUnsafeBuffer(size).fill(fill, encoding);
  }
  return new FastBuffer(size);
};

function createUnsafeBuffer(size) {
  return new FastBuffer(createUnsafeArrayBuffer(size));
}

function createUnsafeArrayBuffer(size) {
  zeroFill[0] = 0;//设定C++默认填充策略为不填充
  try {
    return new ArrayBuffer(size);
  } finally {
    zeroFill[0] = 1;　//分配完后，恢复填充策略
  }
}

class FastBuffer extends Uint8Array {
  constructor(arg1, arg2, arg3) {
    super(arg1, arg2, arg3);
  }
}
```
**Buffer.allocUnSafeSlow（）**
```js
Buffer.allocUnsafeSlow = function(size) {
  assertSize(size);
  return createUnsafeBuffer(size);
};
```

**Buffer.from(ArrayBuffer)**
```js
return new FastBuffer(obj, byteOffset, length);
```

可见alloc()没指定填充和from(ArrayBuffer)都不经过createUnsafeBuffer()，就不会有默认填充策略的改变，就直接使用ES　ArrayBuffer的默认行为（即默认字节值为0),保证安全性。

### slab与C++直接分配结合

**Buffer.from(String)**:
```js

  if (length >= (Buffer.poolSize >>> 1))
    return binding.createFromString(string, encoding);

  if (length > (poolSize - poolOffset))
    createPool();
  var b = new FastBuffer(allocPool, poolOffset, length);
  const actual = b.write(string, encoding);
  if (actual !== length) {
    // byteLength() may overestimate. That's a rare case, though.
    b = new FastBuffer(allocPool, poolOffset, actual);
  }
  poolOffset += actual;
  alignPool();
  return b;
  ```

  当字符串长度大于等于poolSize/2时，C++直接分配内存;反之则采用slab分配策略，即从pool里通过slice方式共享pool的一部分内存，剩余的再给其他Buffer用。

  这里的poolSize默认值有设置:
  ```js
  Buffer.poolSize = 8 * 1024;
  ```
当然这只是在Buffer类里而已，后面分析的Stream类就会有不同的poolSize,而不同stream实现又会有差别。

**Buffer.from(Array/Buffer/TypedArray)**和**Buffer.allocUnsafe()**：
```js
    function allocate(size) {
  if (size <= 0) {
    return new FastBuffer();
  }
  if (size < (Buffer.poolSize >>> 1)) {
    if (size > (poolSize - poolOffset))
      createPool();
    var b = new FastBuffer(allocPool, poolOffset, size);
    poolOffset += size;
    alignPool();
    return b;
  } else {
    return createUnsafeBuffer(size);
  }
}
```

来一张图:

![http://7xsi10.com1.z0.glb.clouddn.com/memory.png](http://7xsi10.com1.z0.glb.clouddn.com/memory.png)


# Stream

## highWaterMark

```js
var hwm = options.highWaterMark;
var defaultHwm = this.objectMode ? 16 : 16 * 1024;
this.highWaterMark = (hwm || hwm === 0) ? hwm : defaultHwm;
this.highWaterMark = Math.floor(this.highWaterMark);
```

默认为16个对象(objectMode下)和16KB，而针对于不同的stream实现，又会对其进行重写，如fs模块的readStream:
```js
if (options.highWaterMark === undefined)
    options.highWaterMark = 64 * 1024;
```
默认为64KB

## 过程
整个过程进行了两层抽象，一层是stream层的，具有一个BufferList，源码参照`/lib/internal/streams/BufferList.js`
这一层用于存储将要消费的Buffer队列；而另一层是内部Buffer,通过slab分配内存的方式，从源文件里读出特定大小的数据，然后通过slice()方法，将这部分内存push到BufferList里。

### 实现_read()方法

._read()方法里去调用源资源的read()操作，读出来的数据暂时存在内部buffer里

fs模块的ReadStream实现的_.read()方法里:
```js
ReadStream.prototype._read = function(n) {
  if (typeof this.fd !== 'number') {
    return this.once('open', function() {
      this._read(n);
    });
  }
  if (this.destroyed)
    return;

  if (!pool || pool.length - pool.used < kMinPoolSpace) {
    // discard the old pool.
    allocNewPool(this._readableState.highWaterMark);
  }
  var thisPool = pool;
  var toRead = Math.min(pool.length - pool.used, n);
  var start = pool.used;

  if (this.pos !== undefined)
    toRead = Math.min(this.end - this.pos + 1, toRead);
  if (toRead <= 0)
    return this.push(null);

  var self = this;
  fs.read(this.fd, pool, pool.used, toRead, this.pos, onread);

  if (this.pos !== undefined)
    this.pos += toRead;
  pool.used += toRead;

  function onread(er, bytesRead) {
    if (er) {
      if (self.autoClose) {
        self.destroy();
      }
      self.emit('error', er);
    } else {
      var b = null;
      if (bytesRead > 0) {
        self.bytesRead += bytesRead;
        b = thisPool.slice(start, start + bytesRead);
      }

      self.push(b);
    }
  }
};
```
可以看到，实际上每次从源文件里读取的数据大小toRead为`var toRead = Math.min(pool.length - pool.used, n);`
然后通过slice()将读出来的数据所在这块内存push到Stream的BufferList里(其实一次push过程并不一定只会slice一次，如果一次的slice过来后`state.length < state.highWaterMark`,还会循环继续从内部buffer读，具体源代码可以查看push方法有关的maybeReadMore()方法)

### 从Stream.read()里来看:

```js
Readable.prototype.read = function(n) {
  debug('read', n);
  n = parseInt(n, 10);
  var state = this._readableState;
  var nOrig = n;

  if (n !== 0)
    state.emittedReadable = false;
  if (n === 0 &&
      state.needReadable &&
      (state.length >= state.highWaterMark || state.ended)) {
    debug('read: emitReadable', state.length, state.ended);
    if (state.length === 0 && state.ended)
      endReadable(this);
    else
      emitReadable(this);
    return null;
  }

  n = howMuchToRead(n, state);

  if (n === 0 && state.ended) {
    if (state.length === 0)
      endReadable(this);
    return null;
  }

  var doRead = state.needReadable;
  debug('need readable', doRead);

  if (state.length === 0 || state.length - n < state.highWaterMark) {
    doRead = true;
    debug('length less than watermark', doRead);
  }

  if (state.ended || state.reading) {
    doRead = false;
    debug('reading or ended', doRead);
  } else if (doRead) {
    debug('do read');
    state.reading = true;
    state.sync = true;
    if (state.length === 0)
      state.needReadable = true;
    this._read(state.highWaterMark);
    state.sync = false;
    if (!state.reading)
      n = howMuchToRead(nOrig, state);
  }

  var ret;
  if (n > 0)
    ret = fromList(n, state);
  else
    ret = null;

  if (ret === null) {
    state.needReadable = true;
    n = 0;
  } else {
    state.length -= n;
  }

  if (state.length === 0) {
    if (!state.ended)
      state.needReadable = true;

    if (nOrig !== n && state.ended)
      endReadable(this);
  }

  if (ret !== null)
    this.emit('data', ret);

  return ret;
};
```

起初流处于paused状态，根据官方文档所述，可以通过以下三种方式将mode转换为flowing:

1. 添加data事件监听器
2. resume()
3. src.pipe()

然后以此为脉络，从flowing的启动到最终数据的呈现过程分析:

监听data事件时:

```js
Readable.prototype.on = function(ev, fn) {
  const res = Stream.prototype.on.call(this, ev, fn);
  if (ev === 'data') {
    if (this._readableState.flowing !== false)
      this.resume();
  } ...

  return res;
};

Readable.prototype.resume = function() {
  var state = this._readableState;
  if (!state.flowing) {
    debug('resume');
    state.flowing = true;
    resume(this, state);
  }
  return this;
};

function resume(stream, state) {
  if (!state.resumeScheduled) {
    state.resumeScheduled = true;
    process.nextTick(resume_, stream, state);
  }
}

function resume_(stream, state) {
  if (!state.reading) {
    debug('resume read 0');
    stream.read(0);
  }
  state.resumeScheduled = false;
  state.awaitDrain = 0;
  stream.emit('resume');
  flow(stream);
  if (state.flowing && !state.reading)
    stream.read(0);
}
```
这里有个关键的一点是`read(0)`,我们查看`Stream.read(n)`方法不难知道，当参数n为0时，只会从内部buffer里slice内存到Stream的BufferList里，然后接着有`flow(stream)`的调用，来看看:
```js
function flow(stream) {
  const state = stream._readableState;
  debug('flow', state.flowing);
  while (state.flowing && stream.read() !== null);
}
```
循环不断的调用stream.read()，即参数n为undefined,而在Stream.read()里经过`parseInt(undfined,10)`结果就为NaN了，然后我们从`howMuchToRead`里看到:
```js
if (n !== n) {
    // Only flow one buffer at a time
    if (state.flowing && state.length)
      return state.buffer.head.data.length;
    else
      return state.length;
  }
```
结论就是，Stream添加ondata后，就会每次自动地从BufferList里读取一个Buffer数据，而当`state.length === 0 || state.length - n < state.highWaterMark`满足时，会执行`_.read()`操作，数据通过`fromList`方法得到，最终触发data事件，将Buffer浮现。一直这样直到源数据末尾。

## 从pipe()角度看限流

设想一个场景，我们的readable stream的速度比writable速度快，我们看看Writable Stream的代码:
```js
function writeOrBuffer(stream, state, isBuf, chunk, encoding, cb) {
  if (!isBuf) {
    var newChunk = decodeChunk(state, chunk, encoding);
    if (chunk !== newChunk) {
      isBuf = true;
      encoding = 'buffer';
      chunk = newChunk;
    }
  }
  var len = state.objectMode ? 1 : chunk.length;

  state.length += len;

  var ret = state.length < state.highWaterMark;
  // we must ensure that previous needDrain will not be reset to false.
  if (!ret)
    state.needDrain = true;

  if (state.writing || state.corked) {
    var last = state.lastBufferedRequest;
    state.lastBufferedRequest = {
      chunk,
      encoding,
      isBuf,
      callback: cb,
      next: null
    };
    if (last) {
      last.next = state.lastBufferedRequest;
    } else {
      state.bufferedRequest = state.lastBufferedRequest;
    }
    state.bufferedRequestCount += 1;
  } else {
    doWrite(stream, state, false, len, chunk, encoding, cb);
  }

  return ret;
}
```
可以看到，当`state.length`超过highWaterMark时，`write()`就返回false，此时如果我们不暂停readable stream,那么接下来的Buffer都会暂存到BufferedRequest里，我们也可以在`write()`返回false时，对readable stream暂停，当BuffererdRequest内的数据写入完毕后，会触发`drain`事件,我们在`ondrain`里可以resume readable stream。

正如Readable Stream的pipe()方法:

```js
src.on('data', ondata);
  function ondata(chunk) {
    ...
    var ret = dest.write(chunk);
    if (false === ret && !increasedAwaitDrain) {
      ...
      src.pause();
    }
  }
```

```js
function pipeOnDrain(src) {
  return function() {
    var state = src._readableState;
    debug('pipeOnDrain', state.awaitDrain);
    if (state.awaitDrain)
      state.awaitDrain--;
    if (state.awaitDrain === 0 && EE.listenerCount(src, 'data')) {
      state.flowing = true;
      flow(src);
    }
  };
}
```

## 未完待续