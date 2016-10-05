---
title: 以一个WebApp开始探寻Vue的思想
date: 2016-09-01 23:19:25
tags: [VueJS,WebApp]
categories: 
- Vue
---
**感想**

我在茫茫的海洋里探寻着前进的方向，那是支撑我走下去的动力。当面对复杂的业务逻辑时，当想要干件大事却力不从心时，多少迷茫，多少灰心丧气。

人类的欲望促使着大脑不断思考，改进技术，进而提高生产力。人类的懒惰，促使着我们去寻找更加方便的工具解放我们的双手。

各种生产活动亦如此，应用开发亦是如此。为了提高生产力，我们要自动化； 为了方便复用，我们要组件化... ...

**文章**
见识过React后再来看看Vue,发现它如此轻盈易于上手，但又不失优雅。奉上一个应用实例，初步感受一下思想的美。

应用UI比较简单，主要是一些数据的逻辑。先上一幅效果图：

![showMemo](http://7xsi10.com1.z0.glb.clouddn.com/showMemo.png)

<!--more-->

Vuejs作为一款MVVM框架，既有像Angular那样的丰富指令集，数据双向绑定，又与React一样强调组件化开发。更重要的，作者是中国人啊！作者是中国人的一个好处那就是，文档支持非常好，学习起来是非常容易的。。。

细节不多讲，因为文档已经够好，我这里来个实践，就感受下写起来爽不爽。

### 分析整个应用结构

拿到一个需求最先开始的是什么，当然得先分析分析，我们看看我们最终要的结果：

![create](http://7xsi10.com1.z0.glb.clouddn.com/createMemo.png)
![home](http://7xsi10.com1.z0.glb.clouddn.com/Mainpage.png)


可以看到，整个应用分为三个功能，首页，查看备忘录，创建备忘录。它们虽然处于不同的页面，但是它们有一个共同点，那就是它们要共享一个数据，那便是备忘录数据。

这份数据应该怎么管理呢？是每个页面都自己做自己的？NO,那样的话你会发现，数据是不好维护的。

我们的做法是: 一个公共组件用于管理数据，三个功能分别是一个子组件，通过父子组件的通信来完成数据的通信。本文的例子比较简单，所以这种方式绰绰有余，对于更复杂的应用，Vue提供了类似React的flux实现:Vuex用于管理大型应用的状态。

而父子组件的关系建立，我们可以用到Vue-router，它的设计本身就是为开发单页面应用(SPA)设计的，平时我们分页怎么做？要么全写出来，然后设置显示隐藏，要么发起异步请求，获取不同页面，我们现在怎么做？整个应用的内容动态渲染，通过路由设置不同url下的组件，这样应用非常流畅。

单页应用虽好，但也有一些问题，需要不断去寻求解决办法，如首屏加载优化，SEO问题等等。

### 我们的技术栈

分析需求了，现在该干什么？当然是技术栈的选型了。

**UI框架**：就用bootstrap吧；
**开发工具**:Webpack
**代码**: Vue,ES2015
**服务器**: express
**数据库**: 为了方便，我就用本地数据库吧

### 搭建开发环境

Vue开源了自己的一款脚手架工具：Vue-cli,用于快速搭建开发环境，并提供了好几种模板，我选用的是Webpack模板，因为平时Webpack用得比较多。

**初始化项目**
```
vue init webpack testApp
```
**进入testApp安装依赖**
```
npm install
```
**运行**
```
npm run dev
```
我是将所有生产的文件都放在dist目录的，包括index.html，只需要修改webpack-html-plugin的配置就好了。

现在已经处于开发环境了，访问localhost:8080/dist/index.html就开始开发了。

Webpack使用就不用说了吧，前面文章我也写过。

### Let's GO

#### 入口文件:main.js

```js
import Vue from 'vue';
import App from './components/App.vue';
import Home from './components/Home.vue';
import Show from './components/Show.vue';
import Create from './components/Create.vue';
import Stylesheet from './assets/style/main.css';
var VueRouter = require('vue-router');

Vue.use(VueRouter);
var router = new VueRouter();

router.map({
    '/Home':{
        component: Home
    },
    '/Show':{
        component: Show
    },
    '/Create':{
        component: Create
    }
});
router.redirect({
  '*':'/Show'
});

router.start(App,'#app');
```
看到了什么，我们定义一个路由器，管理三个路径，对应三个功能，App作为公告的组件

#### 编写公共组件，用于全局管理数据,模板和样式我就省略了吧，完整版我的[github](https://github.com/laoqiren)上有呢。

本文的例子比较简单，所以这种方式绰绰有余，Vue提供了类似React的flux实现:Vuex用于管理大型应用的状态。

App.vue
```js
    import DataBase from './storage.js';
    export default{
        created(){
            var memos = DataBase.query();
            //console.log(memos)
            this.memos = memos;
            console.log(this.memos);
        },
        data(){
            return {
                memos:[],
                hasNew: false
            }
        },
        events: {
            memoUpdate(memo){
                this.hasNew = true;
                this.memos.push(memo);
                console.log(memo.memo)
                DataBase.add(memo.date,memo.time,memo.place,memo.memo);
            },
            memoDelete(index){
                //console.log(index);
                this.memos.splice(index,1);
                DataBase.deleteM(index);
            }
        },
        methods: {
            clearNew(){
                this.hasNew = false;
            }
        }
    }
```
可以看到，它是怎么管理数据的呢?

**初始化数据**:从数据库访问到数据
**分发数据**:通过props向各个子组件传递数据
**响应修改**这个通过事件来完成响应，包括删除，增加，而它们分别是在子组件触发的，然后一路冒泡通知给App组件，这样，子组件的行动尽在掌握之中，操作数据库的行为只交给公共组件。这样是不是感觉整个应用是如此的清晰？

#### 编写数据库
采用本地SQLite数据库，storage.js暴露一个对象，拥有各个操作数据库的方法，然后让App.vue导入

代码省略，详情看[我的github](https://github.com/laoqiren)

#### 编写首页Home.vue

首页的组件要获取数据，计算出备忘录总数，比较简单:
```js
export default{
    props:['memos'],
    computed: {
        total(){
            return this.memos.length;
        }
    }
}
```
#### 编写显示目录Show.vue
获取数据，然后v-for列表渲染，这里有一个删除的戏码，在显示的时候，每条可以选择性地删除啊。所以就要监听删除的事件，并派发给父组件。
```js
export default {
    props: ['memos'],
    methods: {
        deleteM(memo){
            let index = this.memos.indexOf(memo);
            this.$dispatch('memoDelete',index);
        }
    }
}
```

#### 剩下的创建目录Create.vue

这里有一个增加数据的戏码。要监听增加数据的事件，然后还要派发上去，还得将增加的数据以对象的形式传参，这个数据对象是要通过双向数据绑定来不断地随表单输入而获取最新版本的

```js
export default {
    data(){
        return {
            memo:{
            }
        }
    },
    methods: {
        save(){
            var memo = this.memo;
            this.$dispatch('memoUpdate', memo);
        }
    }
}
```

### 结语

有没有感觉很爽？反正我是这么觉得的。这个例子比较简单，但是从这一个简单的例子中我们可以看到用Vue开发大型应用的潜力，而且能够我们以享受。

写代码要写得舒心，就好比我要选择一款自己喜欢的编辑器背景一样，我想让我的开发过程乐趣无穷，而编码的过程，要想转枯燥的工作为享受是要做多方面努力的，你得有工具来代替你做复杂重复的工作，你得有好多应用结构，让自己写得那么随心所欲，尽在掌握，你得。。。。。。

**总之，我们追寻的不是一款最好最牛逼的框架，而是一种思想的美丽，有了思想才会进步。**

以一篇实例文章开始，揭开一探究竟的序幕。

这或许是开学前的最后一篇文章了，大二了，我有许多要学习的，也有许多自己要去改进的，还有美丽的大学生活。加油。