---
title: JWTs之我的前后端完全分离实践
date: 2016-11-01 22:16:23
tags: [JWTs,OAuth,前端工程]
categories: 
- nodeJS
---
> JWT是一种用于双方之间传递安全信息的简洁的、URL安全的表述性声明规范。JWT作为一个开放的标准（ RFC 7519 ），定义了一种简洁的，自包含的方法用于通信双方之间以Json对象的形式安全的传递信息。因为数字签名的存在，这些信息是可信的，JWT可以使用HMAC算法或者是RSA的公私秘钥对进行签名。

之前发表了一篇关于session相关的文章，当时也说明了session这种方式的一些问题，主要就是一个安全性问题。而作为OAuth标准的Access_Token具体解决方案之一的JWTs,使得一些问题得到很好的解决。

文章主要介绍JWT的相关知识以及Nodejs中的JWT,最后，以一个我最近写的论坛为实例，总结一下我在前后端完全分离方面的实践。

论坛采用Vue+express+mongodb开发，不断完善ing,欢迎访问github项目地址star:[https://github.com/laoqiren/vue-express-forum](https://github.com/laoqiren/vue-express-forum),论坛在线地址:[http://120.77.38.217:3000](http://120.77.38.217:3000)

![JWT](http://7xsi10.com1.z0.glb.clouddn.com/UfIbUjj.png)
<!--more-->

### JWTs:

#### 验证的大致流程:

1. 客户端请求登录;
2. 验证信息，登录成功，server生成Access_Token,响应给客户端;
3. 客户端接收Token,储存，可以在localStorage,cookie,本地数据库等;
4. 以后每次请求server需要授权的API,带上Access_Token;
5. server验证Token

#### JWT标准:

JWT包含三个部分:header头部,payload负载,signature签名;

like this:

```
xxxxx.yyyyy.zzzzz
```

##### header:

header包含两个信息，token类型和加密算法,然后使用Base64编码

```js
{
  "alg": "HS256",
  "typ": "JWT"
} 
```

##### payload:

Payload包含实体，包括一些预定义的如iss（签发者） , exp（过期时间戳） , sub（面向的用户） , aud（接收方） , iat（签发时间）等,自定义字段等。

```js
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

##### Signature

签名通过前面的header,payload加上一个secret,最终通过加密算法生成,如:

```js
var encodedString = base64UrlEncode(header) + "." + base64UrlEncode(payload); 
HMACSHA256(encodedString, 'secret');
```

最终的结果如:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJuaW5naGFvLm5ldCIsImV4cCI6IjE0Mzg5NTU0NDUiLCJuYW1lIjoid2FuZ2hhbyIsImFkbWluIjp0cnVlfQ.SwyHTEx_RQppr97g4J5lKXtabJecpejuef8AqKYMAJc
```

#### 客户端发送Token的方式:

发送Token可以通过异步的请求发送JSON,或者加入请求头，或者加入到URL的查询字符串里;

关于请求头的话，RFC 6750定义了一个Bearer Token的标准，like this:

```
Authorization: Bearer <token>
```

### Node的jwt中间件

上面简略的讲了一下JWT的标准，详细内容可以查看官方文档，接下来谈谈Node中的生成Token,Token的验证，Token过期更新，Token持久化等问题。

采用node 的jwt-simple中间件

#### 生成Token:

```js
var moment = require('moment');
var jwt = require("jwt-simple");
...
app.set("jwtTokenSecret","LuoXia");
...
var expires = moment().add(7,'days').valueOf();
var token = jwt.encode({
    iss: 'admin',
    exp: expires,
	user: newUser.name
}, req.app.get('jwtTokenSecret'));
res.json({
    token : token,
    expires: expires,
    user:{
        name:newUser.name
    }
});
```
这里用到了moment中间件，生成过期时间

#### 验证Token:

```js
...
var token = req.body.access_token;//也可以是请求头或者查询字符串
if(token){
	try{
	    var decoded = jwt.decode(token,req.app.get('jwtTokenSecret'));
	    if(decoded.exp < Date.now()){
	        res.end('token expired',401);
	        return;
		}
	}catch(err){
	    res.send(401);
	    res.end();
	    return;
	}
	//其他处理
} else {
	res.status(401);
    res.end();
}
```
#### 更新Token:

```js
var decoded = jwt.decode(token,req.app.get('jwtTokenSecret'));
if(decoded.exp < Date.now()){
    var expires = moment().add(7,'days').valueOf();
	var token = jwt.encode({
	    iss: decoded.iss,
	    exp: expires,
		user: decoded.user
	}, req.app.get('jwtTokenSecret'));
	res.json({
	    token : token,
	    expires: expires,
	    user:{
	        name:newUser.name
	    }
	});
} else {
	...
}
```

#### 持久化Token:

待补充

### 与session对比

1. session依赖于cookie,容易受CSRF漏洞的攻击，不够安全。而JWTs则不依赖于Cookie,虽然token也可以储存在cookie，但是只作为储存方式，并没有作为验证手段。


2. 基于JWTs的Token验证方式是无状态的(stateless),自包含的，例如上例中生成token时就可以添加自定义项，这样就减少了许多数据库查询。

3. Cookie可以通过设置使得在主域名和子域名都能访问到，而如果储存在localStorage下的话，只要是非同域都是不能互相访问到的，解决方案是在包含token的域下设置cookie,这样在其他子域名下就能访问到了。

由于其JWT的无状态性，我们可以很方便的开发一些公用的API，访问公用API只需带上Token，而一些商业性的API服务，可以解析不同的Token来进行计费。


### 前后端完全分离的尝试:

#### 技术选型:

Vue+express+mongodb

#### 分析整个应用:

1. SPA,客户端需有自己的一套路由
2. 组件化开发，可复用
3. 数据逻辑管理，Vuex
4. Node提供REST API
5. Fetch API异步请求，更好的异步流程控制
6. 基于JWT的Token验证方式
7. Webpack构建方案
8. 跨域问题

论坛主要功能: 登录/注册,发表文章，获取文章,退出登录

#### 客户端:

**在组件渲染之前，异步请求获取用户信息:**

```js
created(){
      var token = localStorage.getItem("token");
      var _this = this;
      var content = JSON.stringify({
            access_token:token
          })
      if(token){
        fetch('http://localhost:3000/user',{
          method:'POST',
          headers: {
                      "Content-Type": "application/json",
                      "Content-Length": content.length.toString(),
                    },
          body: content
        }).then(function(res){
          if(res.ok){
            res.json().then(function(data){
              console.log(data.user.name);
              _this.user = data.user;
            });
          } else {
            _this.user = undefined;
          }
        });
      } else {
        this.user = undefined;
      }
}
```

**登录**
```js
handleLog(){
    let _this = this;
    let content = JSON.stringify({
            name:_this.name,
            password:_this.password
        });
    fetch('http://localhost:3000/log',{
        method:'POST',
        headers: {
            "Content-Type": "application/json",
            "Content-Length": content.length.toString(),
        },
        body: content
        }).then(function(res){
            if(res.ok){
                res.json().then(function(data){
                localStorage.setItem("token",data.token);
                _this.user = data.user;
                _this.$dispatch('log',data.user);
                _this.$router.go('/');
                });
            } else {
                _this.user = undefined;
            }
        });
}
```

注册与登录类似

**退出登录**
```js
logOut(e){
    e.preventDefault();
    localStorage.removeItem("token");
    this.$dispatch("logOut");
    this.$router.go("/");
}
```

**发表文章**

需带上Token
```js
handlePost(){
    let _this = this;
    let content = JSON.stringify({
            access_token:localStorage.getItem("token"),
            title:_this.title,
            content:_this.content
        });
    fetch('http://localhost:3000/post',{
        method:'POST',
        headers: {
            "Content-Type": "application/json",
            "Content-Length": content.length.toString(),
        },
        body: content
        }).then(function(res){
            if(res.ok){  
                _this.$router.go('/');
                window.location.reload();
            } else {
               console.log("发表文章失败")
            }
        });
}
```

**路由权限访问限制**

对于没有登录/注册的，不允许访问/post路由;

对于已经登录的不允许访问/log和/reg路由;

#### 服务端

**登录/注册成功生成Token的API**
```js
var expires = moment().add(7,'days').valueOf();
var token = jwt.encode({
    iss: 'admin',
    exp: expires,
	user: newUser.name
}, req.app.get('jwtTokenSecret'));
res.status(200);
res.json({
    token : token,
    expires: expires,
    user:{
            name:user.name
        }
});
```

**根据Token提供User信息的API**
```js
var token = req.body.access_token;
if(token){
    try{
        var decoded = jwt.decode(token,req.app.get('jwtTokenSecret'));
        if(decoded.exp < Date.now()){
            res.end('token expired',401);
        }
        //console.log(decoded)
        res.status(200);
        res.json({
            user:{
                name:decoded.user
            }
            
        });
    } catch(err){
        res.status(401);
        res.send('no token');
    }
}
```

**验证Token的发布文章API**

```JS
var token = req.body.access_token;
    if(token){
        try{
            var decoded = jwt.decode(token,req.app.get('jwtTokenSecret'));
            if(decoded.exp < Date.now()){
                res.end('token expired',401);
                return;
            }
        }catch(err){
            res.send(401);
            res.end();
            return;
        }
        var newPost = new Post({
            name:decoded.user,
            title:req.body.title,
            content:req.body.content
        });
        console.log(newPost);
        newPost.save(function(err,post){
            if(err){
                console.log("发表文章失败");
                res.status(500);
                res.send({error:1});
            } else {
                console.log('发表文章成功');
                res.status(200);
                res.end();
            }
            //console.log(post);
        });
    } else{
        res.status(401);
        res.end();
    }
```

**跨域设置**

由于要在客户端服weppack sever中访问API,需解决跨域限制:

```js
app.all("*",function(req,res,next){
  res.header("Access-Control-Allow-Origin", "*");
  res.header('Access-Control-Allow-Headers', 'Content-Type, Content-Length, Authorization, Accept, X-Requested-With');
  res.header("Access-Control-Allow-Methods","PUT,POST,GET,DELETE,OPTIONS");
  next();
});
```

### 总结

JWT是一种OAuth2.0标准的Access_Token的具体实现方案，可以通过Authorization Bearer,url查询字符串，异步请求的req.body等方式向服务端发送JWT token, 其不依赖于Cookie,有效避免了CSRF攻击，其无状态性可以方便我们编写公共API，此种方案，更加符合RESTFUL API设计理念,在前后端分离实践中发挥着重要的作用。

**永远把自己当做一个beginner,保持好奇心**