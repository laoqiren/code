---
title: Let's Encrypt
date: 2017-03-13 17:03:26
tags: [HTTPS,TLS,Web安全]
categories: 
- Web安全
---
迁移到HTTPS已经成为必然，许多API已经必须部署HTTPS才能使用，如我之前分享过的WebRTC等，ios已经强制app使用HTTPS,HTTP2.0必须基于HTTPS之上。。。

那么这篇文章分享下HTTPS系列之实战部署，从Node server和Nginx Server两种情景来分析。

<!--more-->

# 证书

上篇文章的TLS原理篇提到了证书，用于安全交换公钥，包含了证书订阅者信息，以及CA的数字签名（HASH + 私钥加密，上篇有讲）。

###  字段
![certificate](http://7xsi10.com1.z0.glb.clouddn.com/certificateshow.jpg)
证书字段包含了版本，序列号，签名算法，颁发者，颁发对象，有效期，使用者公钥，证书扩展等信息。

### 证书链
根CA -> 中间CA(可能多个) -> 最终实体证书

client（即信赖方）会存放知名根CA的证书。当server发送过来自己的证书时，client会由该证书知道颁发它的上一级CA的证书，接着从上一级的证书里找到再上一级的证书，一级一级直到根CA证书进行验证。

### 证书吊销

**CRL（证书吊销列表）**，这个列表记录了未过期但是已经被吊销的证书,之前证书字段里的CRL分发点就用于记录CRL地址

**OCSP（在线证书状态协议）**支持实时查询

# 用openssl生成自签名证书

### 生成server RSA私钥:
```
openssl genrsa -out server.key 1024
```
### 生成server RSA公钥:
```
openssl rsa -in server.key -pubout -out server.pem
```
### CA准备
```
$ openssl genrsa -out ca.key 1024 //生成CA私钥
$ openssl req -new -key ca.key -out ca.csr //通过CA私钥生成证书申请文件
$ openssl x509 -req -in ca.csr  -signkey ca.key -out ca.crt //生成CA证书
```
### 用CA颁发server证书
```
$ openssl req -new -key server.key -out server.csr //通过server私钥生成证书申请文件
$ openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt //生成server 证书
```
生成成功:
![generate](http://7xsi10.com1.z0.glb.clouddn.com/generate.jpg)

# Node server部署HTTPS


### 用http-proxy做反向代理，HTTPS->HTTP,起多个服务

```js
const https = require('https');
const proxy = require('http-proxy').createProxyServer({});
const fs = require('fs');
const options = {
    key: fs.readFileSync('./keys/server.key'),
    cert: fs.readFileSync('./keys/server.crt')
}

const server = https.createServer(options,(req,res)=>{
    const host = req.headers.host,
        ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
    switch(host){
        case 'test.localhost':
            proxy.web(req,res,{
                target: 'http://localhost:3000',
                ssl: {
                    key: fs.readFileSync('./keys/server.key', 'utf8'),
                    cert: fs.readFileSync('./keys/server.crt', 'utf8')
                }
            });
            break;
        default:
            res.writeHead(200);
            res.end('welcome to node https proxy server');
    }
}).listen(443);
```
### 将HTTP的80端口请求重定向到HTTPS的443端口
在client输入网址默认是http协议的，所以需要重定向一下
```js
const httpServer = http.createServer((req,res)=>{
    res.writeHead(301,{
        'Location': `https://${req.headers.host}${req.url}`
    })
    res.end();
})
httpServer.listen(80);
httpServer.on('listening',()=>{
    console.log('the http server has been listened at port 80')
})
```

### 综合代码
```js
const https = require('https');
const http = require('http');
const proxy = require('http-proxy').createProxyServer({});
const fs = require('fs');
const options = {
    key: fs.readFileSync('./keys/server.key'),
    cert: fs.readFileSync('./keys/server.crt')
}

const server = https.createServer(options,(req,res)=>{
    const host = req.headers.host,
        ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
    console.log(`client ip: ${ip}`);
    switch(host){
        case 'test.localhost':
            proxy.web(req,res,{
                target: 'http://localhost:3000',
                ssl: {
                    key: fs.readFileSync('./keys/server.key', 'utf8'),
                    cert: fs.readFileSync('./keys/server.crt', 'utf8')
                }
            });
            break;
        default:
            res.writeHead(200);
            res.end('welcome to node https proxy server');
    }
}).listen(443);

server.on('listening',()=>{
    console.log('https server has been listened at port 8000');
})

const httpServer = http.createServer((req,res)=>{
    console.log(`the url is ${req.url}`)
    res.writeHead(301,{
        'Location': `https://${req.headers.host}${req.url}`
    })
    res.end()
})

httpServer.listen(80);
httpServer.on('listening',()=>{
    console.log('the http server has been listened at port 80')
})

const testServer = http.createServer((req,res)=>{
    res.writeHead(200,{
        'Content-Type': 'text/html'
    });
    res.end(fs.readFileSync('./index.html'));
}).listen(3000);
testServer.on('listening',()=>{
    console.log('the test server has been listened at port 3000')
})
```
将我们的CA证书加入客户端受信任CA证书列表，再访问看看：

![nice](http://7xsi10.com1.z0.glb.clouddn.com/localhostresult.jpg)
可爱的小绿锁！

自己充当CA虽然快捷方便，但是客户端并没有信任该CA证书，还得让客户端将你的CA导入才行。这是很伤感的。

# 使用Let's Encrypt免费证书
Let's Encrypt是一个开源免费的公钥证书颁发机构，为了促进web https化进程而生。**

**我这里的环境Centos6 + Nginx
### 安装certbot
```
$ wget https://dl.eff.org/certbot-auto
$ chmod a+x certbot-auto
$ ./certbot-auto
```
### 配置Nginx

Let's Encrypt自助进行颁发证书，需要验证域名所属权限，在申请证书时，Cerbot会在服务器生成随机文件，然后Cerbot服务器会来尝试访问这个随机文件，以此来验证。

在nginx监听80端口且代理了将要申请证书的域名的server里加匹配路由：
```
location ^~ /.well-known/acme-challenge/ {
    default_type "text/plain";
    root /usr/local/webserver/nginx/html;
}

location = /.well-known/acme-challenge/ {
    return 404;
}
```
接着要根据自己的配置在相应目录下新建 **.well-known/acme-challenge/**目录

如这里的**/usr/local/webserver/nginx/html/.well-known/acme-challenge/**

重启nginx

### 申请免费证书
这里我拿自己的一个子域名做测试:**ppt.luoxia.me**
```
./certbot certonly --webroot -w /usr/local/webserver/nginx/html/ -d ppt.luoxia.me
```
**申请成功**
![okla](http://7xsi10.com1.z0.glb.clouddn.com/okla.jpg)

另外注意一个问题，由于Centos 6自带的Python版本是2.6.6，Cerbot不支持，需要升级Python,另外，在申请过程中老是报错：
```
No matching distribution found for acme==0.12.0
```
原因是我的阿里云主机里pip是阿里源，解决办法，修改pip源就好了。
### 更新证书
申请的免费证书有效期为90天，可以进行更新
```
./certbot-auto renew --dry-run 
```


# Nginx  server https部署

原理和Node server类似

### https -> http的反向代理

```
upstream ppt {
    server  localhost:3000;
}
server {
    listen  443 ssl;

    ssl_certificate /etc/letsencrypt/live/ppt.luoxia.me/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ppt.luoxia.me/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/ppt.luoxia.me/chain.pem;

    server_name ppt.luoxia.me;
    location ^~ /.well-known/acme-challenge/ {
            default_type "text/plain";
            root     /usr/local/webserver/nginx/html;
    }
    location = /.well-known/acme-challenge/ {
            return 404;
    }
    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_pass http://ppt/;
    }
}
```
其中ssl_trusted_certificate用于开启OCSP。
### http服务重定向到https
```
server {
    listen 80;
    server_name ppt.luoxia.me;
    rewrite ^(.*)$  https://$host$1 permanent;
}
```
部署成功,可爱的小绿锁。
![nice https](http://7xsi10.com1.z0.glb.clouddn.com/httpsnice.jpg)

# 结束语

博客网站托管在github主机上的，所以暂时没法给自己的博客域名升级到https，在移动端访问会被流氓的运营商加广告，找个机会将博客迁移到自己的主机上。

后面还会有不定期的https系列分享，欢迎大家一起讨论。