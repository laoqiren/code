---
title: Process初始化与模块系统形成
date: 2017-07-24 18:44:54
tags: [process,v8,模块系统]
categories: 
- Nodejs
---

当我们运行`node app.js`的时候都发生了什么？`process`的初始化，模块系统的形成，C/C++与js的结合等等。从源头出发，探索背后的奥秘。
<!--more-->

## 启动及初始化操作

`node_main.cc`是node的入口，根据操作系统做一些初始化工作，最后调用`node::Start()`

在`node.cc`里定义了`Start()`,做了一些初始化platform,V8初始化，libuv event loop创建等工作，然后调用第一个`inline Start()`:
```cpp
const int exit_code =
    Start(uv_default_loop(), argc, argv, exec_argc, exec_argv);
```
在在第一个`inline Start()`里，起一个V8实例，并调用最后一个`inline Start()`:

```cpp
Isolate* const isolate = Isolate::New(params);
...
exit_code = Start(isolate, &isolate_data, argc, argv, exec_argc, exec_argv);
```
接着在最后一个Start()里，初始化context,新建一个env，env用于将`libuv`和`v8`结合在一起:
```cpp
 HandleScope handle_scope(isolate);
  Local<Context> context = Context::New(isolate);
  Context::Scope context_scope(context);
  Environment env(isolate_data, context);
  ...
  env.Start(argc, argv, exec_argc, exec_argv, v8_is_profiling);
```

这里调用了`env.Start();`，`env.Start()`定义在`env.cc`里,该方法里面调用了`SetupProcessObject(this, argc, argv, exec_argc, exec_argv);`，而该方法又定义在`node.cc`里，定义了`process`的一些属性和方法(其中包括了`process.binding()`用于C/C++模块机制，后面详解):

![http://7xsi10.com1.z0.glb.clouddn.com/setupProcessObject.png](http://7xsi10.com1.z0.glb.clouddn.com/setupProcessObject.png)

最后一个`inline Start()`还进入了一个while循环处理`libuv`事件:

```cpp
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
```

另外还调用了`LoadEnvironment(&env);`:

```cpp
Local<String> script_name = FIXED_ONE_BYTE_STRING(env->isolate(),
                                                    "bootstrap_node.js");
Local<Value> f_value = ExecuteString(env, MainSource(env), script_name);
```

其中`MainSource(env)`:

```cpp
Local<String> MainSource(Environment* env) {
  return String::NewFromUtf8(
      env->isolate(),
      reinterpret_cast<const char*>(internal_bootstrap_node_native),
      NewStringType::kNormal,
      sizeof(internal_bootstrap_node_native)).ToLocalChecked();
}
```
而这里的`internal_bootstrap_node_native`由`node_natives.h`定义，这个头文件是由`js2c.py`工具生成的，将所有native模块都编译到C++数组里:

![http://7xsi10.com1.z0.glb.clouddn.com/node_natives.h.bootstrap.png](http://7xsi10.com1.z0.glb.clouddn.com/node_natives.h.bootstrap.png)
执行`bootstrap_node.js`的匿名函数并传入`process`对象,`process`对象通过`env->process_object()`获得：

```cpp
...
Local<Function> f = Local<Function>::Cast(f_value);
...
Local<Value> arg = env->process_object();
f->Call(Null(env->isolate()), 1, &arg);
```
在`bootstrap_node.js`初始化了一些process方法和属性，global变量，模块机制等。

## 执行一个js文件

为了说明一个运行一个js文件发生了什么，先说明一下模块系统的初始化

### 模块系统形成

#### process.binding() C/C++内建模块
上面我们说在程序启动后会在`SetupProcessObject()`里为`process`对象绑定一些方法，其中就包括`process.binding`:
```cpp
if (cache->Has(env->context(), module).FromJust()) {
    exports = cache->Get(module)->ToObject(env->isolate());
    args.GetReturnValue().Set(exports);
    return;
  }
...

node_module* mod = get_builtin_module(*module_v);
if (mod != nullptr) {
    exports = Object::New(env->isolate());
    CHECK_EQ(mod->nm_register_func, nullptr);
    CHECK_NE(mod->nm_context_register_func, nullptr);
    Local<Value> unused = Undefined(env->isolate());
    mod->nm_context_register_func(exports, unused,
        env->context(), mod->nm_priv);
    cache->Set(module, exports);
} else if ...
```
其中`get_builtin_module(*module_v);`在`modlist_builtin`链表中获取模块。同样用了缓存机制。那么这些C/C++模块是怎么放到链表上面去的呢？答案是通过`NODE_MODULE_CONTEXT_AWARE_BUILTIN`，比如`zlib`调用了`NODE_MODULE_CONTEXT_AWARE_BUILTIN(zlib, node::InitZlib)`来将该模块加入到上边儿的链表中。

我们在`node.h`看到了这个宏定义:
```cpp
#define NODE_MODULE_CONTEXT_AWARE_BUILTIN(modname, regfunc)           \
  NODE_MODULE_CONTEXT_AWARE_X(modname, regfunc, NULL, NM_F_BUILTIN)
```
而`NODE_MODULE_CONTEXT_AWARE_X`最终会调用`node.cc`里定义的`node_module_register(&_module);`将C/C++模块加入到`modlist_builtin`链表中，供`get_builtin_module()`使用。

```cpp
extern "C" void node_module_register(void* m) {
  struct node_module* mp = reinterpret_cast<struct node_module*>(m);

  if (mp->nm_flags & NM_F_BUILTIN) {
    mp->nm_link = modlist_builtin;
    modlist_builtin = mp;
  } else if (!node_is_initialized) {
    mp->nm_flags = NM_F_LINKED;
    mp->nm_link = modlist_linked;
    modlist_linked = mp;
  } else {
    modpending = mp;
  }
}
```
#### native js模块

在`bootstrap_node.js`里：
```js
NativeModule._source = process.binding('natives');
NativeModule._cache = {};
```

当调用`process.binding('natives');`的时候，`node.cc`:

```cpp
if (!strcmp(*module_v, "natives")) {
    exports = Object::New(env->isolate());
    DefineJavaScript(env, exports);
    cache->Set(module, exports);
}
```
在`src/node_javascript.cc`中关于`DefineJavaScript()`:

```cpp
void DefineJavaScript(Environment* env, Local<Object> target) {
  HandleScope scope(env->isolate());

  for (auto native : natives) {
    if (native.source != internal_bootstrap_node_native) {
      Local<String> name = String::NewFromUtf8(env->isolate(), native.name);
      Local<String> source =
          String::NewFromUtf8(
              env->isolate(), reinterpret_cast<const char*>(native.source),
              NewStringType::kNormal, native.source_len).ToLocalChecked();
      target->Set(name, source);
    }
  }
}
```
而上面的`natives`就是在`node_natives.h`里边儿定义的：

![http://7xsi10.com1.z0.glb.clouddn.com/node_natives.h.natives.png](http://7xsi10.com1.z0.glb.clouddn.com/node_natives.h.natives.png)

对于`require`同样使用了cache机制:
```js
const cached = NativeModule.getCached(id);
    if (cached && (cached.loaded || cached.loading)) {
      return cached.exports;
    }
```
最终调用`compile()`方法:
对源码用`wrapper`进行了包装:
```js
var source = NativeModule.getSource(this.id);
source = NativeModule.wrap(source);
```
然后在vm里执行，并传入一些包装后匿名函数需要的参数:
```js
const fn = runInThisContext(source, {
    filename: this.filename,
    lineOffset: 0,
    displayErrors: true
});
fn(this.exports, NativeModule.require, this, this.filename);
```

这样我们就可以来理解执行一个文件的过程:
### 执行一个js文件（文件模块）
```js
const path = NativeModule.require('path');
process.argv[1] = path.resolve(process.argv[1]);

const Module = NativeModule.require('module');

if (process._syntax_check_only != null) {
    const fs = NativeModule.require('fs');
    const filename = Module._resolveFilename(process.argv[1]);
    var source = fs.readFileSync(filename, 'utf-8');
    checkScriptSyntax(source, filename);
    process.exit(0);
}

preloadModules();
Module.runMain();
```
同步读取执行的js文件,`lib/module.js`中的`runMain()`:
```js
Module.runMain = function() {
  Module._load(process.argv[1], null, true);
  process._tickCallback();
};
```

Module._load:
```js
Module._load = function(request, parent, isMain) {
  if (parent) {
    debug('Module._load REQUEST %s parent: %s', request, parent.id);
  }

  var filename = Module._resolveFilename(request, parent, isMain);

  var cachedModule = Module._cache[filename];
  if (cachedModule) {
    return cachedModule.exports;
  }

  if (NativeModule.nonInternalExists(filename)) {
    debug('load native module %s', request);
    return NativeModule.require(filename);
  } //如果在native模块里找到就调用NativeModule的require机制

  var module = new Module(filename, parent);

  if (isMain) {
    process.mainModule = module;
    module.id = '.';
  }

  Module._cache[filename] = module;

  tryModuleLoad(module, filename);

  return module.exports;
};
```

`Module._resolveFilename()`经过一系列的查找机制（包括后缀扩展，包查找等）后，得到一个合适的`filename`，`tryModuleLoad()`里会调用`Module.load()`:
```js
var extension = path.extname(filename) || '.js';
if (!Module._extensions[extension]) extension = '.js';
Module._extensions[extension](this, filename);
this.loaded = true;
```
对于js文件调用`_compile()`方法:

同样进行了包装（包装方法和内容和NativeModule相同)，并传入自己的参数在vm里执行代码:
```js
content = internalModule.stripShebang(content);
var wrapper = Module.wrap(content);
var compiledWrapper = vm.runInThisContext(wrapper, {
    filename: filename,
    lineOffset: 0,
    displayErrors: true
});
...
result = inspectorWrapper(compiledWrapper, this.exports, this.exports,
require, this, filename, dirname);
```

而传入的`require`就是传入的`Module.prototype.require()`:
```js
Module.prototype.require = function(path) {
  assert(path, 'missing path');
  assert(typeof path === 'string', 'path must be a string');
  return Module._load(path, this, /* isMain */ false);
};
```
可见最终又是走`_load`。这其实是文件模块（第三方和自定义模块）的加载方式，而用node执行一个js文件，实际上用到的也就是这种文件模块的机制，不过多了一系列的启动操作。

可以分析得到，执行一个js文件时，会去初始化`process`，其中包括定义了`process.binding()`方法来定义`C/C++`模块机制，然后会去执行一个native模块即`bootstrap_node.js`，它的代码放在了`node_natives.h`里，从那里读取code array,在C++层面运行即调用了`bootstrap_node.js`的匿名函数并传入`process`对象，在`bootstrap_node.js`里，定义了native js模块机制，即通过`process.binding('natives)`得到`node_natives.h`里的`natives`数组，包含了所有native模块的代码数组。然后对于执行一个js文件，调用原生模块`module`，去执行`Module.runMain()`，而这个操作不过是由`module`定义的文件模块机制罢了。


## 总结

文章从node启动到一个js文件的执行的角度去分析内部原理，详细解释了与`process`对象有关和模块系统的形成。而对于其他细节诸如`libuv event loop`机制还需要深究，会在后面的文章中进行总结。欢迎讨论。