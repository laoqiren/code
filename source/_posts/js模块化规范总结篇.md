---
title: js模块化规范总结篇
date: 2016-07-23 10:21:59
tags: [模块化,CommonJS,AMD,CMD,ES6 Module]
categories: 
- 前端工程
---
之前一篇文章讲过requirejs(amd规范描述)的路径问题，这篇文章对js模块化进行总结，AMD和CMD主要用于客户端的模块化，而CommonJS是Nodejs的规范，由于它们的差异性，也延伸出来一些模块化工具，如broswerify能够让客户端书写Commonjs规范的模块化代码，达到前后端模块公用，webpack支持各种规范的模块化方式，而到后来的ES2015模块化标准，它们的规范及差异就是本文要总结的。

对于webpack的话，最近研究得比较多，我会在后面的另一篇文章中详细总结一下webpack的东西
<!--more-->

### 前言

#### 模块化开发介绍(引用lovenyf's blog)

模块化是指在解决某一个复杂问题或者一系列的杂糅问题时，依照一种分类的思维把问题进行系统性的分解以之处理。模块化是一种处理复杂系统分解为代码结构更合理，可维护性更高的可管理的模块的方式。可以想象一个巨大的系统代码，被整合优化分割成逻辑性很强的模块时，对于软件是一种何等意义的存在。对于软件行业来说：解耦软件系统的复杂性，使得不管多么大的系统，也可以将管理，开发，维护变得“有理可循”。

还有一些对于模块化一些专业的定义为：模块化是软件系统的属性，这个系统被分解为一组高内聚，低耦合的模块。那么在理想状态下我们只需要完成自己部分的核心业务逻辑代码，其他方面的依赖可以通过直接加载被人已经写好模块进行使用即可。

首先，既然是模块化设计，那么作为一个模块化系统所必须的能力：

    1. 定义封装的模块。
    2. 定义新模块对其他模块的依赖。
    3. 可对其他模块的引入支持。

模块化开发的重要性，需要我们制定模块化规范。由于js跨平台的特性，其不同的环境催生出了不同的模块化规范。

#### CommonJS规范

##### 模块化是如此的重要，而js这门奇葩的语言在es2015之前是没有模块的规范的，模块化在其他语言中是那么的普遍，连css也可以通过import实现模块化，伴随着开发复杂度的不断提升，js的模块化需求迫在眉睫。而Nodejs的CommonJS率先制定了js的模块化标准：

三个关键字：require,exports,module
##### 1. 模块定义：
```js
// sayHello.js
function sayHello(){
	console.log('hello');
}
exports.sayHello = sayHello;
```
exports是作为模块文件唯一出口的对象,而module为全局的对象，其上有一个exports属性和全局的exports指向同一个对象，对于直接导出对象的模块，可以这样简写：
```js
module.exports = {
	sayName(){
		console.log('my name is Luo Xia');
	}
	...
}
```
需要注意的是，全局的exports直接指向一个对象是不能改变Module.exports对象的，而Module.exports只要不为空，则最终出口将会忽略在全局exports对象上添加的属性和方法：
```js
//Module.js
function a(){
	console.log('a');
} 
exports.a = a;
module.exports = {
	b(){
		console.log('b');
	}
};

//testM.js
let m = require("./Module");
console.log(typeof m.a); //undefined
console.log(typeof m.b); //function
```
##### 2. 引入模块依赖:
```js
require('./sayHello');
```

require()传入的模块标识需要分情况，核心模块和文件模块:

核心模块存在于node源码中，不用文件定位和编译，而文件模块动态加载，可以是npm安装的依赖包下的模块，也可以是自定义的模块文件，对存在于node_modules下的模块文件，不用写路径，会从子目录到顶层目录依次查找该模块文件，而对于用户自定义的不存在与node_modules下的模块，是需要些路径的。

**加载顺序**：优先从缓存加载（核心模块缓存优先于文件模块缓存）

**文件定位**： 

1. 标识符不含扩展名，按.js,.json,.node顺序尝试自动扩展。
2. 在node_modules内的包下，识别package.json的main属性，定位该文件，而如果有错或者没有package.json文件，则将index.*文件作为该模块的文件。

**模块编译**:

详细参考《深入浅出Nodejs》

#### AMD

##### 前面的CommonJS是NodeJS的规范，客户端呢？随着客户端开发越来越复杂，如果没有模块化的话，要管理互相的依赖是非常麻烦的，而AMD便是用于客户端的模块规范，可以采用同步和异步地加载方式加载模块文件。

关键字：define,require,

##### 1. 模块定义:全局的define( id?, dependencies?, factory ); 

其中id为模块标识符，dependencies为模块依赖的其他模块，factory为依赖加载完毕后执行的回调函数可以看出，这整个是异步方式加载的。其中id和dependencies都可以省略，factory可以是回调函数，或者js对象。

**a**. 对于独立的模块（即不依赖其他模块），省略dependencies：
```js
define(function(){
	return {
		sayHello(){
			console.log('hello');
		}
	}
});
```
**b**.对于依赖其他模块的模块：
```js
define(['jquery'],function(){
	return {
		...
	}
});
```
**c**. 上述例子，如果直接返回一个对象，factory可以直接是对象的形式：
```js
define({
	sayHello(){
		console.log('hello');
	}
}
```
**d**. AMD的作者本来是想让AMD规范不局限于CommonJS，但在后续版本也能够类似于CommonJS的写法：
```js
define(function(require,exports,module){
	require('...');
	...
});
```
这里require不是全局的，在其内部require是同步加载模块的，除了这种写法，还可以这样：
```js
define(['require'],function(require){
	...
});
```
这就导致有两种情况的require,全局和局部。这也是后面要讲的AMD与CMD的区别之一。

#### CMD

CMD是国内前端前辈玉伯的杰作，只怪前端领域日新月异，变化无常，正当我在写这篇文章时，seajs(CMD描述）已经结束了自己的生涯。

##### 1. 定义模块，类似于AMD的类似于CommonJS的写法，这样使得其和NodeJS兼容性更好。
```js
define(function(require,exports,module){
		...
});
```
	
define()也可以直接传入对象或者字符串:
```js
define('this is a string');
define({...});
```
##### 2. 引入模块：

在define()的factory函数内部，requie作为局部变量使用

同步加载模块:
```js
require('');
```
异步加载：
```js
requie.async(id,callback?);
```
而对于全局加载的话，CMD的实现者Seajs提供了seajs.use()，这样不同于AMD的既有全局require,又有局部require:
```js
seajs.use('./Module.js');//阻塞加载

seajs.use('./Module.js',function(){
	console.log('module is loaded');
});//异步加载
```


#### AMD与CMD区别：(引用自玉伯本人）

1. 对于依赖的模块，AMD 是提前执行，CMD 是延迟执行。不过 RequireJS 从 2.0 开始，也改成可以延迟执行（根据写法不同，处理方式不同）。CMD 推崇 as lazy as possible.


2. CMD 推崇依赖就近，AMD 推崇依赖前置。


3. AMD 的 API 默认是一个当多个用，CMD 的 API 严格区分，推崇职责单一。比如 AMD 里，require 分全局 require 和局部 require，都叫 require。CMD 里，没有全局 require，而是根据模块系统的完备性，提供 seajs.use 来实现模块系统的加载启动。CMD 里，每个 API 都简单纯粹。

4. 参考[https://github.com/seajs/seajs/issues/277](https://github.com/seajs/seajs/issues/277)



#### browserify:让客户端使用CommonJS规范写法，让客户端可以使用强大的npm生态系统：

borowserify运行用户之间书写符号CommonJS规范的模块定义，然后经由browserify打包成可之间在浏览器运行的bundle文件：
```js
Module.js:
let $ = require('jquery');
module.exports = {
	sayName(){
		console.log('my name is LuoXia');
};

// main.js:
let {sayName} = require('./Module.js');
sayName();
```
然后：

		$ browserify main.js > bundle.js

就可以把main.js及其依赖模块打包到bundle.js，浏览器引入就是了。详细使用方法参考browserify官方文档。


#### 拥抱ES2015 Module规范

刀耕火种的年代已久过去，工业化时代已久到了，而es语言规范本身也在不断进步，前面讲述的种种规范都是"野生"的，当模块化话题被带到官方规范，前后端共用一套规范，让开发成本变得更低。


关键字：export, import

##### 定义模块：
```js
// profile.js
var firstName = 'Michael';
var lastName = 'Jackson';
var year = 1958;

export {firstName, lastName, year};
```
这里export后面并不是对象，而是表示出口的一系列值，可以用as关键字进行重命名：
```js
export{firstName as name}//对外提供name
```
注意：
```js
// 报错
export 1;

// 报错
var m = 1;
export m;

// 报错
function f() {}
export f;

// 正确
export function f() {};

// 正确
function f() {}
export {f};
```
##### 引用模块：
```js
import {firstName, lastName, year} from './profile';
```
{}内部的变量名需与profile.js模块中的出口变量名一致，顺序没关系。也可以用as重命名：
```js
import {firstName as name, lastName, year} from './profile';
```
如果在一个模块之中，先输入后输出同一个模块，import语句可以与export语句写在一起。
```js
export { es6 as default } from './someModule';

// 等同于
import { es6 } from './someModule';
export default es6;
```
整体加载：
```js
import * as about from './profile';
console.log(about.firstName);
```
只执行，无入口：
```js
import './someModule';
```
默认输出：export default

有时候我们并不知道模块的API，但提供默认输出，这样就可以直接获取默认输出了。
```js
// export-default.js
export default function () {
  console.log('foo');
}
```
这时通过import不需要{}因为只有一个变量：
```js
import lala from './export-default';
```
这样我们使用一些库，框架的时候会非常方便：
```js
import $ from 'jquery';
import React from 'react';
import ReactDOM from 'react-dom';
```

#### ES6 Module与CommonJS对比：


##### 1. CommonJS的模块是对象，而es6的模块不是对象，是编译时执行的，使得编译时就能够确定依赖关系和输入输出的变量（软一峰前辈原文）


```js
// CommonJS模块
let { stat, exists, readFile } = require('fs');

// 等同于
let _fs = require('fs');
let stat = _fs.stat, exists = _fs.exists, readfile = _fs.readfile;
```
上面代码的实质是整体加载fs模块（即加载fs的所有方法），生成一个对象（_fs），然后再从这个对象上面读取3个方法。这种加载称为“运行时加载”，因为只有运行时才能得到这个对象，导致完全没办法在编译时做“静态优化”。

ES6模块不是对象，而是通过export命令显式指定输出的代码，输入时也采用静态命令的形式。
```js
// ES6模块
import { stat, exists, readFile } from 'fs';
```
上面代码的实质是从fs模块加载3个方法，其他方法不加载。这种加载称为“编译时加载”，即ES6可以在编译时就完成模块加载，效率要比CommonJS模块的加载方式高。当然，这也导致了没法引用ES6模块本身，因为它不是对象。


##### 2. 模块加载实质不同

CommonJS加载的模块是对值的拷贝，而ES6 Module是对值的引用，这样模块定义文件内部变量的变化会实时反映到依赖文件，而CommonJS不同。
```js
// lib.js
export let counter = 3;
export function incCounter() {
  counter++;
}

// main.js
import { counter, incCounter } from './lib';
console.log(counter); // 3
incCounter();
console.log(counter); // 4
//模块定义文件的变量counter的变化能够实时反映
```

##### 3. 模块的循环加载处理方式不同：
		
循环加载：两个脚本互相依赖
```js
// a.js
var b = require('b');

// b.js
var a = require('a');
```
这部分内容在这里不讲了，详情参考《es6入门2》相关内容。


#### 不同的规范有它自己的适用场景，相同的是对模块化开发的追求，作为工程化开发的一部分，模块化开发不仅包括了js的模块化，还包括css,html模板之类的模块化，本文对一系列的标准进行了总结和对比，希望能够传播模块化开发思想，然后让开发变得更高效。

###### webpack作为一款新兴的模块化管理和打包工具，其视任意文件都为模块，并打包成bundle文件，相比于browserify更加强大，后面的文章我会对我在实践当中webpack的使用进行总结，将组件化开发，模块化开发，自动化构建结合，探索高效的开发之道。