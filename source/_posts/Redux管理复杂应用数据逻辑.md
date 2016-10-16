---
title: Redux管理复杂应用数据逻辑
date: 2016-10-04 19:57:32
tags: [Redux,React]
categories: 
- React
---

也许你也感受到web世界的变化，web客户端从简单的展示性页面向着越来越复杂的web应用转化。而在前后端分离下的开发模式下，客户端数据逻辑变得纷繁复杂，难以维护。React设计理念之一为单向数据流，这从一方面方便了数据的管理。但是React本身只是view，并没有提供完备的数据管理方案。应由而生的其周边类库，诸如ImmutableJS不变数据解决方案，flux作为state管理方案。而redux即为类似于flux的数据解决方案。虽然文章分类于react，但是并不代表redux就与react有着必然的联系。其作为独立的库，适用于多个应由框架甚至是原生JavaScript。

类似于Redux的，有为Vue设计的Vuex。Redux/Vuex的数据管理方案，深深地透露出函数式编程(FP)的思想。对于习惯了面向对象(OO)思想的我们来说，函数式编程也是一大挑战。所以本人也打算对函数式编程的一些主要概率进行总结，作为单独的一篇文章。
![原理图](http://7xsi10.com1.z0.glb.clouddn.com/redux-flow.jpeg)

**国庆节还在写代码的，一定是单身狗**

<!--more-->
### 整体分析:

1. 应用分为两个大部分，一个是UI,一个是数据，	UI根据数据来渲染，不同的数据渲染出不同的UI，即是UI的各个状态state;


2.  State如何改变：用户产生操作，触发事件，组件层次设计为两类，一类为展示性组件(Dumb/Presentational Components)，不处理数据逻辑相关，一类为容器组件（Smart/Container Components)展示性组件根据用户操作触发事件，向容器组件派发，容器组件发生dispatch。


3.  传入dispatch的为Action生成函数，Action为一个含有type属性的js对象，但是通过Middleware即中间件，可以让action生成函数不一定直接返回对象，如redux-thunk中间件可以让其返回函数，根据这个，可以处理异步的数据流。


4.  reducer指定如何更新Sate,其为纯函数，并且不直接修改原始state，而是保持原有state不变，返回全新的state


5.  state更新，触发渲染，UI更新。

### 三大原则:

#### 单一数据源
即整个应用只有一个store来管理state，与flux明显不同，这样使得管理更加方便，但也需要我们自己去分解store,这个可以通过分解reducer来实现

#### State的改变只能通过分发action
即State是只读的，应用中不能直接修改state,只能产生dispatch,靠action生成函数生成action,然后reducer更新state

#### 通过纯函数修改State

纯函数的问题，也是来自于函数式编程思想，我们在中学时学的函数就是纯函数，对于同一个输入，必然有相同的输出。这就保证了数据的可控性。什么情况下导致函数不纯呢？

如：

. 修改传入参数；

. 执行有副作用的操作，如 API 请求和路由跳转；

. 调用非纯函数，如 Date.now() 或 Math.random()。

state每次更新，原本的state保持不变，这使得利用redux实现时间旅行，撤销重做更加容易。如更改一个state对象的某个属性:

```js
return Object.assign({},state,{
           [action.reddit]: posts(state,action)
       });
```

或者使用ES7的对象rest语法:

```js
return {...state,[action.reddit]: posts(state,action)}
```
或者可以使用immutable.js库，生成不变的数据类型。

### 直接上实例，逐步分析

以官方的Reddit API实例来分析，这个例子涉及到主要的知识点，包括一些高级的中间件，异步数据流，以及组件层次的设计等。

#### index.js入口文件：

```js

import React from 'react';
import ReactDOM from 'react-dom';
import {createStore,applyMiddleware,compose} from 'redux';
import {Provider} from 'react-redux';
import thunk from 'redux-thunk';
import createLogger from 'redux-logger';
import reducer from './reducers/index';
import App from './containers/App';

const middleWare = [thunk,createLogger()];

const store = createStore(
    reducer,compose(
        applyMiddleware(...middleWare),
        window.devToolsExtension ? window.devToolsExtension() : f => f
    )   
);

ReactDOM.render(
    <Provider store={store}>
        <App/>
    </Provider>,
    document.getElementById('root')
);
```

首先上面的store常量即为整个应用的唯一store,通过createStore()方法生成，其参数有reducer,以及使用的默认state,使用的中间件，applyMiddleWare使用一系列的中间件，它的作用是对action creater返回的数据进行处理，然后交给reducer。

react-redux使得react更好的和redux结合，提供Provider组件包裹我们的组件，这样我们的组件就能直接访问到dispatch方法和state.我们可以利用后面讲到的connect方法将组件和redux连接。

#### action

```js
export const REQUEST_POSTS = 'REQUEST_POSTS';
export const RECEIVE_POSTS = 'RECEIVE_POSTS';
export const SELECT_REDDIT = 'SELECT_REDDIT';
export const INVALIDATE_REDDIT = 'INVALIDATE_REDDIT';

export const selectReddit = reddit => ({
    type: SELECT_REDDIT,
    reddit
});

export const requestPosts = reddit => ({
    type: REQUEST_POSTS,
    reddit
});

export const receivePosts = (reddit,json)=>({
    type: RECEIVE_POSTS,
    reddit,
    posts: json.data.children.map(child=>child.data),
    receivedAt: Date.now()
});

export const invalidateReddit = reddit => ({
    type: INVALIDATE_REDDIT,
    reddit
});

const fetchPosts = reddit => dispatch => {
    dispatch(requestPosts(reddit));
    return fetch(`https://www.reddit.com/r/${reddit}.json`)
        .then(response=>response.json())
        .then(json=>dispatch(receivePosts(reddit,json)));
};

const shouldFetchPosts= (state,reddit)=>{
    const posts = state.postsByReddit[reddit];
    if(!posts) return true;
    if(posts.isFetching) return false;
    return posts.didInvalidate;
};

export const fetchPostsIfNeeded = reddit => (dispatch,getState) => {
    if(shouldFetchPosts(getState(),reddit)){
        return dispatch(fetchPosts(reddit));
    }
};
```

可以看到上述的fetchPosts函数，返回的并不是一个action对象，而是返回了一个函数，这个默认redux是没法处理的，这就需要使用中间件处理了，redux-thunk中间件用于处理返回函数的函数，thunk函数在我之前的一篇'异步编程解决方案'也提到过，即接受回调函数的函数。这里返回的这个函数，其又返回一个promise对象，即通过fetch API异步发起请求，最终返回Promise对象。这里传入的dispatch也是中间件封装dispatch()方法的作用结果。

#### reducer

```js
import {combineReducers} from 'redux';
import {
    REQUEST_POSTS,RECEIVE_POSTS,
    SELECT_REDDIT,INVALIDATE_REDDIT
} from '../actions/index.js';

const selectReddit = (state = 'reactjs', action) => {
    switch(action.type){
        case SELECT_REDDIT:
            return action.reddit;
        default:
            return state;
    }
};
const posts = (state = {
    isFetching:false,
    didInvalidate:false,
    items:[]
},action) => {
    switch(action.type){
        case INVALIDATE_REDDIT:
            return Object.assign({},state,{didInvalidate:true});
        case REQUEST_POSTS:
            return Object.assign({},state,{isFetching:true,didInvalidate:false});
        case RECEIVE_POSTS:
            return Object.assign({},state,{isFetching:false,didInvalidate:false,items:action.posts,lastUpdated:action.receivedAt});
        default:
            return state;
    }
};

const postsByReddit = (state={ },action)=>{
    switch(action.type){
        case INVALIDATE_REDDIT:
        case REQUEST_POSTS:
        case RECEIVE_POSTS:
            return Object.assign({},state,{
                [action.reddit]: posts(state,action)
            });
        default:
            return state;
    }
};

const rootReducer = combineReducers({
    postsByReddit,
    selectReddit
});

export default rootReducer;
```

可见，这种写法用到了一种技巧，即对大的state进行了分解，一部分为postsByReddit，一部分为selectReddit,避免了代码的冗长，最后只需通过combineReducers()方法将分解的reducer结合即可。

#### 设计组件层次

按照之前所述的组件层次设计哲学思想，将组件分为container组件和展示组件，只让container来产生dispatch,而这个container哪些组件才有资格担任呢？通常是App即最外层父组件，然后当我们在使用react-router的时候，每个route都可以作为单独的container与redux相连接。

来看看这个例子的App.js:

```js

import React,{Component} from 'react';
import {connect} from 'react-redux';
import {selectReddit,invalidateReddit,fetchPostsIfNeeded} from '../actions/index';
import Picker from '../components/Picker';
import Posts from '../components/Posts';

class App extends Component {
    componentDidMount(){
        const {dispatch,selectReddit} = this.props;
        dispatch(fetchPostsIfNeeded(selectReddit));
    }
    handleChange(nextReddit){
        this.props.dispatch(selectReddit(nextReddit));
    }
    handleRefreshClick(e){
        e.preventDefault();

        const {dispatch,selectReddit} = this.props;
        dispatch(invalidateReddit(selectReddit));
        dispatch(fetchPostsIfNeeded(selectReddit));
    }
    componentWillReceiveProps(nextProps) {
        if (nextProps.selectReddit !== this.props.selectReddit) {
            const { dispatch, selectReddit } = nextProps;
            dispatch(fetchPostsIfNeeded(selectReddit));
        }
    }
    render(){
        const {posts,selectReddit,lastUpdated,isFetching} = this.props;
        const isEmpty = posts.length === 0;
        return (
            <div>
                <Picker value={selectReddit}
                        onChange={this.handleChange.bind(this)}
                        options={['reactjs','frontend']}/>
                <p>
                    {
                        lastUpdated && 
                        <span>
                            最后更新时间:{new Date(lastUpdated).toLocaleTimeString()}
                        </span>
                    }
                    {
                        !isFetching && 
                        <a href="javascript:;" onClick={this.handleRefreshClick.bind(this)}>刷新
                        </a>
                    }
                </p>
                {
                    isEmpty? (isFetching?<h3>正在加载当中...</h3>:<h3>没有相关文章</h3>)
                            :<div style={{backgroundColor:isFetching?0.5:1}}>
                                <Posts posts={posts}/></div>
                }
                
            </div>
        );
    }
}

const mapStateToProps = state => {
    const {selectReddit,postsByReddit} = state;
    const {isFetching,lastUpdated,items:posts=[]} = postsByReddit[selectReddit] ||
            {
                isFetching:true,
                items: []
            };

    return {
        selectReddit,
        isFetching,
        lastUpdated,
        posts
    };
};

export default connect(mapStateToProps)(App);
```

可见我们将App与Redux进行了连接，通过connect方法,我们知道，当组件与redux连接时，我们就能获得dispatch和store,而mapStateToProps正是根据获得的state生成了App需要用到的props，这样我们的state就与组件联系在一起了。

### 结束语

可见，redux的数据管理方案相对于之前的组件单独管理自己的state确实是清晰了不少，但是要求我们的学习成本也更高了，我们需要有一定的函数式编程思想，我们还得自己去设计组件的层次。这种数据管理方式，对于复杂应用确实比较适合，但是当我们的应用足够简单的时候，使用redux是否是画蛇添足，反而使得问题复杂化了。具体的应用场景还得在实践当中去感悟。。。

因为最近项目的需要，其复杂程度也足够需要使用类似于redux的数据管理模式，还有一些问题没有很好的解决，如react父路由与子路由的数据传输。之前的vue可以直接通过router-view那种类似于父子组件的props传递方式进行数据传递，但react貌似没这种处理方式，所以就在想除了各个router都与redux相连的处理方式，还有什么办法，等着解决。

**国庆节假期即将结束，紧张又充实的生活又将开启**