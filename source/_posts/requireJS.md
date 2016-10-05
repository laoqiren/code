---
title: requireJS
date: 2016-05-07 14:04:59
tags: [模块化,requirejs]
categories: 
- 前端工程
---
#### 序言
模块化开发在如今纷繁复杂的产品开发中显得非常重要，而由于js自身的缺陷，在es6之前，js并没有自己的模块规范，而nodeJS通过CommonJS的模块规范实现了自己的模块机制，但是这仅仅是在node端，而在客户端，有自己的实现方式，诸如AMD,CMD之类，本文是我初次接触AMD规范时遇到的问题总结，而这段序言也是在我后面重新加上的。
<!--more-->
##### 在于客户端，requirejs是AMD规范最好的实现者，关于requirejs在使用过程中遇到的问题记录，主要是路径问题。（CommonJS,AMD,UMD等规范内容找个时间再总结下，以及后边与es6模块的对比）

我的项目结构（测试用，不合理）：

	----gulp
		----app
			----html
					index.html
			----js
					a.js
					b.js
					c.js
			----require.js
			----config.js
		----bower_comonents
			----jquery
				----dist
					jquery.min.js

1. 当引入requirejs的时候指定data-main属性:(index.html)
```html  
<script data-main="../config" src="../require.js"></script>
```
2. data-main作为入口文件，有个作用就是它会把模块的baseUrl设置为config所在目录
3. 然后配置config.js:
```js
require.config({
    paths : {
        jquery: ['../bower_components/jquery/dist/jquery.min']
    }
});
```
4. baseUrl就是config文件所在目录，设定paths值当然根据baseUrl来，如这里就相当于gulp/app/../bower_components/jquery/dist/jquery.min，然后index.html:
```js
<script type="text/javascript">
    require(['jquery'],function($){
        $('body').append($('<p>hello</p>'));
    })
</script>
```
理论上是没问题的，但会报错，get app/jquery.js net::ERR_FILE_NOT_FOUND，这咋回事？paths根本没生效，然后我又试着显示地配置baseUrl属性（注意配置路径是针对于引入requirejs的那个html文件来的），照样报同样的错误，最后，我这样写：
```js
<script type="text/javascript" src="../require.js"></script>
<script type="text/javascript" src="../config.js"></script>
<script type="text/javascript">
    require(['jquery'],function($){
        $('body').append($('<p>hello</p>'));
    })
</script>
```
也就是说，config分开加载，当然此时的config配置会有所变化，因为不是在data-main属性中引入，所有baseUrl默认的会是这里引入require的html文件的路径，所以：
```js
require.config({
    baseUrl:'../../',
    paths : {
        jquery: ['bower_components/jquery/dist/jquery.min']
    }
});
```
这里baseUrl设置后就相当于gulp目录，然后现在在运行html,达到预期效果，why?
#### 应该是这样：开始的例子中，引入require文件，加载执行require.js,然后异步加载config.js，config配置还没生效时，我后面的代码就执行了，由于已经设置data-main属性，所以回去加载app/js/jquery.js，没找到，所以挂了。

我的解决办法：
要么把代码放在config.js中，要么在config里引入我的代码模块,或者同步加载config配置，只不过是baseUrl不同而已。

#### 另外还有一个需要注意到的是，在定义模块的时候，define(id,dep,factory)的时候，dep依赖其他模块的目录有两种方式：

1. 依据config显式地设置的baseUrl或者默认的baseUrl的路径结合paths配置
2. ./开头的路径，那么久不是根据baseUrl了，是根据这个模块本身的位置来的

举个栗子：

index.html:

```html
<script type="text/javascript" src="../require.js"></script>
<script type="text/javascript" src="../config.js"></script>
<script type="text/javascript" src="../js/a.js">
</script>
<script type="text/javascript">
    require(['jquery'],function($){
        $('body').append($('<p>hello</p>'));
    })
</script>
</head>
<body>
    <div>hello wrold</div>
</body>
```
config.js:
```js	
require.config({
    baseUrl:'../../',
    paths : {
        jquery: ['bower_components/jquery/dist/jquery.min']
    }
});
```
a.js:
```js
require(['app/js/b'],function(b){
    b.jq();
});
```
b.js:
```js
//这里也可以define(['jquery','./c'],...)
define(['jquery','app/js/c'],function($,c){
    var b = {};
    b.jq = function(){
        $('body').append($('<p>hello</p>'));
    };
    console.log(c.name);
    return b;
});
```
c.js:
```js
define(function(){
    return {
        name:'luoxia'
    };
});
```
b模块依赖c模块的情况说明这个问题，然后require是一定根据baseUrl来的

#### es2015来了，原生模块语法都支持了，官方是要用原生module语法抛弃CommonJS和AMD,CMD之类的模块规范啊，反正都有babel之类的转换器，为何不用起来呢？es2015的module和之前的一些规范是有区别的，比如引用而非值缓存啊之类的，需要慢慢学习研究哦。