---
title: 使用React SSR构建Isomorphic应用
date: 2017-02-07 17:18:56
tags: [server-rendering,react,redux,react-router,webpack,同构]
categories: 
- React
---
所谓Isomorphic应用即为前后端共用代码的同构应用。与之对应的还有Universal应用，得益于virtual DOM，使得同一套js代码可以运行在各个环境如web, 安卓/IOS,桌面应用等。

为什么要同构？SPA大大改善了用户体验，但是也存在一些问题，如SEO,首屏渲染效率问题。于是就有一种方案便是SSR即服务端渲染。如React的SSR,通过在server端获取初始state,返回已经含有数据的DOM结构，再交给客户端实例化React，之后客户端接手路由渲染任务，而这一部分能够共用一套代码再好不过了。

文章以我的实际项目为例，分析整个应用的技术架构，总结一系列问题如服务端渲染组件时的静态资源问题等。

项目运用React,redux,react-router,React SSR, express,mongoose, Ant design, webpack技术栈，欢迎star:[https://github.com/laoqiren/isomorphic-redux-forum](https://github.com/laoqiren/isomorphic-redux-forum)

<!--more-->

## React SSR的好处

1. 利于SEO：React服务器渲染的方案使你的页面在一开始就有一个HTML DOM结构，方便Google等搜索引擎的爬虫能爬到网页的内容。
2. 提高首屏渲染的速度：服务器直接返回一个填满数据的HTML，而不是在请求了HTML后还需要异步请求首屏数据
3. 前后端都共用代码，提供开发效率
4. 摒弃Template,以组件方式组织代码

## React SSR提供的API

React SSR提供了一些API,可以和React-router, Redux进行配合，渲染首屏.

### renderToString 和 renderToStaticMarkup方法用于将Virtual DOM渲染成HTML字符串

```js
import {renderToString} from 'react-dom/server';
const html = renderToString(
            <Provider store={store}>
                <RouterContext {...renderProps}/>
            </Provider>
        )
```
renderToString：将React Component转化为HTML字符串，生成的HTML的DOM会带有额外属性：各个DOM会有data-react-id属性，第一个DOM会有data-checksum属性。

![图1](http://7xsi10.com1.z0.glb.clouddn.com/WBPD@L8EK306G~EE%7D4~AS%5BO.png)

renderToStaticMarkup：同样是将React Component转化为HTML字符串，但是生成HTML的DOM不会有额外属性，从而节省HTML字符串的大小。

### 渲染原理

服务端渲染出来的只是静态DOM，还不能使用State, props,事件等，必须在客户端实例化React,而在客户端渲染之前会根据DOM结果和props计算组件的checksum值，然后与服务端渲染结果中的checksum进行比较，如果相同则不会重新渲染该组件，而如果不想同，则会重新渲染，并抛出错误。当没有找到checksum值时，也会重新渲染。

这里需要注意一个问题，要将renderToString渲染出来的DOM用额外的div包裹起来，不然会导致checksum不同:

```js
<div id="container"><div>${html}</div></div>
```

### 和Redux配合:

** 服务端Redux做如下工作:**

. 为每次请求创建全新的 Redux store 实例；

. 按需 dispatch 一些 action；

. 从 store 中取出 state；

. 把 state 一同返回给客户端。

```js
const store = storeApp({});
Promise.all([
    store.dispatch(selectAuthor('all')),
    store.dispatch(fetchPostsIfNeeded('all'))
])
.then(()=>{
    const html = renderToString(
        <Provider store={store}>
            <RouterContext {...renderProps}/>
        </Provider>
    )
    const finalState = store.getState();
    res.end(renderFullPage(html,finalState));
})
```
在renderFullPage方法里将初始state挂在window上，方便客户端渲染用:

```js
window.__INITIAL_STATE__ = ${JSON.stringify(initState)}
```

**客户端:**
```js
const initState = window.__INITIAL_STATE__;

const store = storeApp(initState);

ReactDOM.render(
    <Provider store={store}>
       <Router history={browserHistory}>
        {routesApp}
       </Router>
    </Provider>,
    document.getElementById('container'));
```

### 与React-router配合:

使用match方法匹配路由，使用RouterContext渲染路由组件

路由组件生成: routes.js:
```js
import React from 'react';
import {Route,IndexRoute} from 'react-router';
import App from '../common/components/App';
import Item from '../common/components/Item';
import List from '../common/components/List';
import Publish from '../common/components/Publish';
import Space from '../common/components/Space';
import LogIn from '../common/components/LogIn';
import Reg from '../common/components/Reg';
const routes = (
    <Route path="/" component={App}>
            <IndexRoute component={List}/>
            <Route path="/item/:id" component={Item}/>
            <Route path="/space" component={Space}/>
            <Route path="/publish" component={Publish}/>
            <Route path="/logIn" component={LogIn}/>
            <Route path="/reg" component={Reg}/>
    </Route>
    );

export default routes;
```

render.js:
```js
export default function handleRender(req,res){
    match({routes:routesApp,location:req.url},(err,redirectLocation,renderProps)=>{
        if(err){
            res.status(500).end(`server error: ${err}`)
        } else if(redirectLocation){
            res.redirect(redirectLocation.pathname+redirectLocation.search)
        } else if(renderProps){
            const store = storeApp({});
            Promise.all([
                store.dispatch(selectAuthor('all')),
                store.dispatch(fetchPostsIfNeeded('all'))
            ])
            .then(()=>{
                const html = renderToString(
                    <Provider store={store}>
                        <RouterContext {...renderProps}/>
                    </Provider>
                )
                const finalState = store.getState();
                res.end(renderFullPage(html,finalState));
            })
        } else {
            res.status(404).end('404 not found')
        }
    })
}
```

## 实例分析

###  技术选型:

. React

. Redux管理state

. React-router管理前端路由

. React 服务端渲染

. Webpack构建

. Ant design UI

. Express

. Mongoose操作数据库

### 项目目录结构:

```
--- assests //静态资源
--- public //开发环境的bundle文件
--- client //客户端部分
	--- index.js //客户端入口文件，渲染React实例
	--- devTools.js //开发工具配置
--- server //服务端部分
	--- Models //操作数据库
	--- api //提供REST API
	--- app.js //服务器主文件
	--- render.js //React 服务端渲染逻辑
	--- index.js //服务器入口文件
--- common //同构部分
	--- components //组件
	--- actions //Redux action
	--- reducers //Redux reducer
	--- configureStore.js //store生成器
	--- routes.js //router生成器
---webpack
	--- run-webpack-server.js //运行webpack服务的入口文件
	--- webpack-assests.json // webpack-isomorphic-tools生成的静态资源路径文件
	--- webpack-dev-server.js // webpack服务器
	--- webpack-isomorphic-tools.configuration.js //webpack-isomorphic-tools配置文件
	--- webpack-status.json // webpack-isomorphic-tools的日志文件
	--- webpack.config.js //webpack配置文件

--- .babelrc
--- .eslintrc
--- .gitignore
--- LICENSE
--- package.json
--- README.md

```

### webpack:

webpack配置:
```js

const HtmlWebpackPlugin = require('html-webpack-plugin');
const webpack = require('webpack');
const Webpack_isomorphic_tools_plugin = require('webpack-isomorphic-tools/plugin');
const path = require('path');

const webpack_isomorphic_tools_plugin = 
  new Webpack_isomorphic_tools_plugin(require('./webpack-isomorphic-tools-configuration'))
  .development()

const HtmlWebpackPluginConfig = new HtmlWebpackPlugin({
    template: `${__dirname}/../client/index.html`,
    filename: 'index.html',
    inject: false
});

module.exports = {
    context: path.join(__dirname,'..'),
    entry:[
        'webpack-hot-middleware/client?path=http://localhost:3001/__webpack_hmr',
        './client/index.js'
    ],
    output:{
        path: `${__dirname}/../dist`,
        publicPath: 'http://localhost:3001/public/',
        filename: '[name].[hash].js'
    },
    module: {
        loaders: [
            {
                test:/\.jsx?$/,
                loaders: ["react-hot-loader","babel-loader"],
                exclude: /node_modules/
            },
            {
                test:/\.css$/,
                loaders: ['style-loader','css-loader']
            },
            {
                test:/\.scss$/,
                loaders: ['style-loader','css-loader','sass-loader']
            },
            {
                test: webpack_isomorphic_tools_plugin.regular_expression('images'),
                loader: 'url-loader?limit=10240', // any image below or equal to 10K will be converted to inline base64 instead
            }
        ]
    },
    plugins: [
        HtmlWebpackPluginConfig,
        webpack_isomorphic_tools_plugin,
        new webpack.HotModuleReplacementPlugin(),
        new webpack.NoEmitOnErrorsPlugin(),
        new webpack.BannerPlugin("This file is created by Luo Xia")
    ]
}
```

webpack服务: webpack-dev-server.js:
```js
import express from 'express';
import path from 'path';
import qs from 'qs';
import webpack from 'webpack';
import webpackDevMiddleware from 'webpack-dev-middleware';
import webpackHotMiddleware from 'webpack-hot-middleware';
import webpackConfig from './webpack.config';


const app = new express();
const port = 3001;
const compiler = webpack(webpackConfig);

compiler.plugin('compilation',compilation=>{
    compilation.plugin('html-webpack-plugin-after-emit', (data, cb)=> {
        webpackHotMiddleware(compiler).publish({ action: 'reload' });
        cb();
    });
});
app.use(webpackDevMiddleware(compiler,{
    noInfo:true,
    publicPath: webpackConfig.output.publicPath,
    hot: true,
    stats: {
            colors: true,
            chunks: false
        },
     headers: {
         'Access-Control-Allow-Origin': '*',
        "Access-Control-Allow-Methods":"PUT,POST,GET,DELETE,OPTIONS"
    },
    historyApiFallback: true
}));
app.use(express.static(path.join(__dirname,'../dist')))

app.use(webpackHotMiddleware(compiler,{
    path: '/__webpack_hmr'
}));

app.use('*',(req,res,next)=>{
    res.header("Access-Control-Allow-Origin", "*");
    res.header('Access-Control-Allow-Headers', 'Content-Type, Content-Length, Authorization, Accept, X-Requested-With');
    res.header("Access-Control-Allow-Methods","PUT,POST,GET,DELETE,OPTIONS");
    next()
})

app.listen(port,err=>{
    if(err){
        console.error(err);
    } else {
        console.info(`the webpack server has been listened at port: ${port},haha`)
    }
})
```
#### 服务端渲染时静态资源问题

webpack可以将任意资源打包，那是在客户端，但是当其运行在服务端时，对于非js文件是无法正常import的，这里我的处理办法是webpack-isomorphic-tools

在index.js里:
```js
require('babel-register');

const Webpack_isomorphic_tools = require('webpack-isomorphic-tools')
const project_base_path = require('path').join(__dirname, '..')
global.webpack_isomorphic_tools = new Webpack_isomorphic_tools(require('../webpack/webpack-isomorphic-tools-configuration'))
    .server(project_base_path)
    .then(()=>{
        require('./app');
    })
```
即在运行应用服务器时，先启动webpack-isomorphic-tools服务，它会在指定路径下生成webpack-assests.json和webpack-status.json，其中webpack-assests.json就指定了各个静态文件的路径:

![图2](http://7xsi10.com1.z0.glb.clouddn.com/%60TXZ%28%6098%7DI1%5BQ%5D40RFU10MI.png)

**注意**
起初我是将webpack-dev-server和应用服务器弄在一起的，这就会导致无法生成webpack-assests.json，就会导致静态资源加载失败

#### 使用webpack-hot-middleware热更新问题

之前一直是使用webpack-dev-server形式试想HMR的，但是这里使用了webpack-dev-middleware和webpack-hot-middleware，会有一些区别:

入口文件:
```js
entry:[
        'webpack-hot-middleware/client?path=http://localhost:3001/__webpack_hmr',
        './client/index.js'
    ]
```
之所以path配置为3001服务地址是因为webpack-dev-server是运行在3001端口的，然而我在开发中是用应用服务3000端口的，所以这也涉及到跨域问题，需要设置CORS头

react-hot-loader:
```js
test:/\.jsx?$/,
loaders: ["react-hot-loader","babel-loader"],
exclude: /node_modules/
```

注意这里别忘了exclude,不然会报错:ReactHotAPI is not a function

插件:
```js
plugins: [
        HtmlWebpackPluginConfig,
        webpack_isomorphic_tools_plugin,
        new webpack.HotModuleReplacementPlugin(),
        new webpack.NoEmitOnErrorsPlugin(),
        new webpack.BannerPlugin("This file is created by Luo Xia")
    ]
```

指定路径:
```js
app.use(webpackHotMiddleware(compiler,{
    path: '/__webpack_hmr'
}));
```
这里不能加http://localhost:3001，否则404

### 同构部分

和Redux一般写法没有差别，只是这里要抽象出一层store生成器和routes生成器:

configureStore.js:
```js
import {createStore,applyMiddleware,compose} from 'redux';
import thunkMiddleware from 'redux-thunk';
import reducerApp from '../common/reducers/index';

import { composeWithDevTools } from 'redux-devtools-extension';

export default function(initState){
    
    return createStore(reducerApp,initState,composeWithDevTools(
        applyMiddleware(thunkMiddleware)
    ))
}
```

注意这里的redux-dev-tools部分，如果按照如下所示使用:
```js
const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose;
```

这样是行不通的，因为服务端并不能使用window，所以使用模块方式引入

routes.js:
```js
import React from 'react';
import {Route,IndexRoute} from 'react-router';
import App from '../common/components/App';
import Item from '../common/components/Item';
import List from '../common/components/List';
import Publish from '../common/components/Publish';
import Space from '../common/components/Space';
import LogIn from '../common/components/LogIn';
import Reg from '../common/components/Reg';
const routes = (
    <Route path="/" component={App}>
            <IndexRoute component={List}/>
            <Route path="/item/:id" component={Item}/>
            <Route path="/space" component={Space}/>
            <Route path="/publish" component={Publish}/>
            <Route path="/logIn" component={LogIn}/>
            <Route path="/reg" component={Reg}/>
    </Route>
    );

export default routes;
```

### 完整的服务端渲染部分:
render.js
```js
import {renderToString} from 'react-dom/server';
import qs from 'qs';
import {Provider} from 'react-redux';
import reducerApp from '../common/reducers/index';
import React from 'react';
import {RouterContext,match} from 'react-router';
import {selectAuthor,fetchPostsIfNeeded} from '../common/actions/actions'
import storeApp from '../common/configStore';
import routesApp from '../common/routes';
import fetch from 'isomorphic-fetch'
import fs from 'fs';
import path from 'path';


function renderFullPage(html,initState){
    const main = JSON.parse(fs.readFileSync(path.join(__dirname,'../webpack/webpack-assets.json'))).javascript.main;
    return `
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8">
            <title>react-ssr</title>
        </head>
        <body>
            <div id="container"><div>${html}</div></div>
            <script>
                window.__INITIAL_STATE__ = ${JSON.stringify(initState)}
            </script>
            <script src=${main}></script>
        </body>
        </html>
    `
}

export default function handleRender(req,res){
    match({routes:routesApp,location:req.url},(err,redirectLocation,renderProps)=>{
        if(err){
            res.status(500).end(`server error: ${err}`)
        } else if(redirectLocation){
            res.redirect(redirectLocation.pathname+redirectLocation.search)
        } else if(renderProps){
            const store = storeApp({});
            Promise.all([
                store.dispatch(selectAuthor('all')),
                store.dispatch(fetchPostsIfNeeded('all'))
            ])
            .then(()=>{
                const html = renderToString(
                    <Provider store={store}>
                        <RouterContext {...renderProps}/>
                    </Provider>
                )
                const finalState = store.getState();
                res.end(renderFullPage(html,finalState));
            })
        } else {
            res.status(404).end('404 not found')
        }
    })
}
```

### 其他部分

sever部分除了react首屏渲染部分外，其余和我之前的项目vue-express-forum差不多，由于篇幅限制，这里就不详解了。感兴趣移步我的文章:[JWTs之我的前后端完全分离实践](http://luoxia.me/code/2016/11/01/JWTs%E4%B9%8B%E6%88%91%E7%9A%84%E5%89%8D%E5%90%8E%E7%AB%AF%E5%AE%8C%E5%85%A8%E5%88%86%E7%A6%BB%E5%AE%9E%E8%B7%B5/).

另外开发过程中遇到了许多坑，其中有些坑都是一些依赖的版本问题。所以当遇到问题无法解决时，更改依赖版本，或许就能解决。如这个项目的node-sass，react-router使用最新版都会存在不兼容问题哦。

## 结语

可见react的生态是非常丰富的，它的诞生引入了许多多新的概念，也使得JS的发展焕发蓬勃生机。这里也不得不提到Vue2.0，它也正式支持了服务端渲染。这方面还有许多东西有待研究。

这个寒假也只剩下最后的10天了，珍惜时光，不负青春。


