---
title: webpack构建方案
date: 2016-07-25 15:35:05
tags: [webpack]
categories: 
- 前端工程
---
webpack管理模块依赖然后打包输出文件，采用loader机制处理各个模块，并提供开发者环境自动化构建。本文通过实例，记录我的使用总结及遇到的问题，前面介绍一下webpack主要的内容，然后以react+ES2015+webpack为例演示，通过react-hot-loader局部地热更新组件提高开发效率。

<!--more-->

#### 为什么使用webpack？

官方关于webpack出现的目的

1. Split the dependency tree into chunks loaded on demand
2. Keep initial loading time low
3. Every static asset should be able to be a module
4. Ability to integrate 3rd-party libraries as modules
5. Ability to customize nearly every part of the module bundler
Suited for big projects

可见webpack能够采用代码分块机制来防止一些无谓的资源加载，减少http请求优化渲染速率，把每个文件都视为模块采用loader机制进行处理，支持各种模块化规范，并且还能够提供自动化解决方案。

初步来看，webpack既能够像browserify那样管理依赖打包文件，又能像gulp那样对不同的文件进行处理并且能够实现自动化。种种优点，值得一试。

本文是我在学习中的总结，有些理解还不够，所以文章会持续更新，本来写文章的目的之一就是记录成长嘛~

#### 开始使用webpack

**全局安装webpack**

	npm install -g webpack

**局部安装webpack**

	npm install --save-dev webpack

**配置文件**

	项目根目录下webpack.config.js

#### 配置webpack.config.js

##### 资源入口及打包输出：
```js
entry:['./src/js/app.js'],
output:{
    path: __dirname + '/dist/js',
    filename:'test.bundle.js',
    publicPath:'/public'
}
```
这里的path要是绝对路径，__dirname是nodejs中的当前文件所在目录，关于publicPath,这个我折腾了些时间才理解。。。这个后面实例的时候仔细说明一下。

##### 资源的处理
```js
module:{
    loaders:[
        {
            test:/\.jsx?$/,
            exclude:/node_modules/,
            loaders:["react-hot","babel"]
        },
        {
            test:/\.css$/,
            loader:'style!css'
        },
        {
            test: /\.scss$/,
            loader: 'style!css!sass'
        },
        {
            test:/\.(jpg|png)$/,
            loader:'url'
        }
    ]
}
```
module配置项，通过loader机制对不同的文件（模块）进行处理，webpack视各种文件为模块，loader需要自己npm安装依赖，test是匹配文件类型，这里采用正则的方式进行匹配，exclude即忽略某些文件，避免去识别加载不必要的文件，loader(loaders)支持两种写法：
```js
loader:"react-hot!babel"

loaders:["react-hot","babel"]
```

两种写法都是从右往左进行处理，类似于gulp的pipe流，右侧的loader处理后，传递给左边的loader，直到最后。

##### 资源加载
```js
resolve: {
    root: [process.cwd() + '/src', process.cwd() + '/node_modules'],
    alias: {},
    extensions: ['', '.js', '.css', '.scss', '.ejs', '.png', '.jpg']
}
```
extensions即扩展名，当加载依赖模块时，可以智能化的扩充资源扩展名，而root的设置，使得我们可以直接加载npm包。

##### 插件的使用：
```js
plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoErrorsPlugin(),
    new webpack.BannerPlugin('This file is created by LuoXia')
]
```
如上例，banner-plugin能够在最终的打包文件当中写明文件作者。。。

##### 代码分块(code splitting)：
```js
//static imports
import _ from 'lodash'

// dynamic imports
require.ensure([], function(require) {
  let contacts = require('./contacts')
})
```
这个暂时还没有实践,后面补充。

##### 实时更新（自动化构建）:

	web-dev-server文件有更新会全局的刷新页面

	react-hot-loader监听组件变化，局部刷新

#### 实践（ReactJS+ES2015+webpack):

前阵子参与了学校60周年校庆网站的前端开发部分，子页有很多可以复用的部分，所以我把他们实现了组件化

##### 项目目录结构：

	---node_modules //npm包
	---dist //最终输出目录
		--- js //bundle文件
		--- index.html //入口页面
	---src //生成环境
		---components //组件所在目录
			---header //头部组件
				---style //组件样式
					---header.scss
				---js //组件js/jsx文件
					---header.js
				---images //图片资源
			---footer
				---style
					---footer.scss
				---js
					footer.js
				---images
			---app.js //入口文件
	---package.json //npm包依赖管理
	---webpack.config.js //webpack配置文件
	---.babelrc //babel配置文件
	---.eslintrc.json //eslint配置
	---server.js //nodejs服务器
	---npm-debug.log //日志文件

##### 编写组件:

依赖的其他组件，依赖的模块，依赖的图片资源文件，这些都可以当做模块来加载：
```js
//footer.js
import React from "react";
import ReactDOM from "react-dom";
let weibo = require('../images/fuwu.jpg');
let wechat = require('../images/dingyue.png');
```

通过es2015的class来创建组件:
```js
export default class FooterComponent extends React.Component {
    constructor(){
        super();
    }
    render(){
        return (<div id="footer">
            <div className="footer-left">
                <p>{this.props.introContent[0]}<br/>
                    {this.props.introContent[1]}<br/>
                    {this.props.introContent[2]}
                </p>
            </div>
            <div className="footer-right">
                <div className="ewm1"><img src={weibo}/><span>杭电官方微博</span></div>
                <div className="ewm2"><img src={wechat}/><span>杭电官方微信</span></div>
            </div>
        </div>);
    }
}
```
##### loaders:

**对于js/jsx文件**，用babel-loader处理，babel是一款非常厉害的编译工具，使得js成为一门为数不多的能够实时使用新特性的语言。通过.babelrc配置或者直接在webpack中配置,支持对jsx,es2015的转换：
```js	
{
  "presets": ["es2015","react"],
  "plugins": ["transform-react-jsx"]
}
```
细心的你会发现前面对于js/jsx文件还有个react-hot-loader文件，这就是实时热更新用到的loader，后面会详细说明

**对于scss文件**，依次用sass-loader,css-loader,style-loader进行处理：
```js		
test:/\.css$/,
loader:'style!css'
```
**图片文件**,url-loader处理
```js
test:/\.(jpg|png)$/,
loader:'url'
```
##### 入口文件，出口文件，publicPath:
```js
entry:['webpack-dev-server/client?http://localhost:3000',
'webpack/hot/only-dev-server','./src/components/app.js'],
output:{
    path: __dirname + '/dist/js',
    filename:'test.bundle.js',
    publicPath:'/public'
}
```
app.js作为入口文件，层层管理依赖，最终打包到pah目录，publicPath见下文

##### 自动化构建，webpack-dev-server&react-hot-loader:

###### 命令行webpack-dev-server:


安装:

	npm install --save-dev webpack-dev-server

关键字--inline,--hot:

webpack支持命令行配置,webpack-dev-server --inline时，会自动添加webpack-dev-server的入口文件到entry:

	'webpack-dev-server/client?http://localhost:8080'
当浏览器访问localhost:8080时，会进入inline模式，而访问localhost:8080/webpack-dev-server时，会进入iframe模式，iframe模式通过一个iframe包裹页面,提供顶部的状态更新标识。

当--hot命令时，标识实时热更新，会自动在webpack配置文件当中添加插件：
```js
new webpack.HotModuleReplacementPlugin()
```
需要注意的是，webpack-dev-server没有入口文件情况下是没办法访问webpack.config.js的
```js
devServer:{
	stats:{
		colors:true
	},
	hot：true
}
```
###### nodeJS API下的webpack-dev-server:

提供webpack-dev-server和热更新的入口：
```js
entry:['webpack-dev-server/client?http://localhost:3000',
'webpack/hot/only-dev-server','./src/js/app.js']
```
提供插件：
```js		
new webpack.HotModuleReplacementPlugin(),
new webpack.NoErrorsPlugin()//非必须，阻止报错
```
然后server.js:
```js
var webpack = require('webpack');
var WebpackDevServer = require('webpack-dev-server');
var config = require('./webpack.config');

new WebpackDevServer(webpack(config), {
  publicPath: config.output.publicPath,
  hot: true,
  stats:{colors:true},
  historyApiFallback: true
}).listen(3000, 'localhost', function (err, result) {
  if (err) console.log(err);
  console.log('Listening at localhost:3000');
});
```
当node server.js的时候，服务器监听在localhost:3000端口，可以把这个命令写入npm script:
```js
"scripts": {
"start": "node server.js"
	}
```
###### react-hot-loader:

只是在对js/jsx文件的loaders中，新增react-hot-loader，webpack-dev-server本身是整个页面刷新的，这样不必要，而react-hot-loader采用异步方式局部刷新。

###### 关于publicPath，这个我纠结了很久，最后在segamentfault社区前辈的帮助下终于开窍了。

publicPath是出口的打包文件在服务器中的位置，如这里的:
```
publicPath:'/public'
```
那么我们的test.bundle.js文件就存在于：

	localhost:3000/public/test.bundle.js

我们在index.html文件里就可以这样引用:
```html
<script src="http://localhost:3000/public/test.bundle.js"></script>
```
记住webpack-dev-server监听文件变化，实时更新，但是并不会实时编译，只是将变化存在于内存当中，我们必须访问服务器地址里的bundle文件，否则就不能监听到文件变化,比如如果我们这样引入：
```html
<script src="./js/test.bundle.js"></script>
```

可见我们这样访问的是真实路径当中的bundle文件，而如上所说，bundle文件并没有被实时编译，我们就会发现页面不论怎么都没有变化了。

此外，在scipt中引入的路径也可以是相对路径，相对于publicPath:
	
	publicPath:"/dist/js"
	
	<script src="./js/test.bundle.js"></script>

##### 最终测试：

我们的app.js文件：
```js	
import $ from 'jquery';
import React from 'react';
import ReactDOM from 'react-dom';
import FooterComponent from '../js/footer';
import HeaderComponent from '../js/header';
import ContentListComponent from '../js/content-list';
import stylesheet from '../style/main.scss';
let introContent = ["地址：杭州市杭州经济技术开发区白杨街道2号大街1158号 邮编:310018 电话查号:86915114","党委宣传部制作维护 浙ICP备05018777","Copyright 2016 杭州电子科技大学版权所有 All right reserved"];

$(document).ready(function(){
    ReactDOM.render(<div><HeaderComponent/><ContentListComponent/><FooterComponent introContent={introContent}/></div>,$('#container')[0]);
});
```
**npm run start**

服务器运行在localhost:3000

**访问http://localhost:3000/webpack-dev-server/dist/index.html**:

**截图1:**

![60-1](http://7xsi10.com1.z0.glb.clouddn.com/MUP2%250N@BCC9X%7D8EMUSN3YC.png)

**截图2:**

![60-2](http://7xsi10.com1.z0.glb.clouddn.com/60-footer.png)

当我们更改任意依赖文件内容，页面都能够实时局部热更新。good job!

#### 可见webpack确实能够提供开发效率，本文记录的只是冰山一角，文中有些不够深入或者有偏差的地方，所以文章将持续更新，如果你也对它感兴趣的话可以查看[官方文档](https://webpack.github.io/docs)。

### 探寻高效的开发之道，追寻工程化开发之路，我们在路上。。。