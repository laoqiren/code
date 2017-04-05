---
title: Docker初步实践
date: 2017-04-01 19:41:23
tags: [docker,云计算]
categories: 
- docker
---
以Node+Mongodb+Nginx为例（上次的同构项目），记录Docker初步实践，希望对同阶段的同学有一点启发。
<!--more-->

## 单个服务

### Mongo

**关于用户创建**

先在没有加--auth情况下run一个容器，进行用户创建，再以--auth方式起一个容器（最后工作的容器），怎么实现的：

启动容器时创建数据卷并挂载到容器，这里创建用于存放数据库数据的数据卷，这样可以让数据在多个容器间共享，包括数据库的用户配置:
```
$ docker run --name db -d -p 27520:27017 -v /home/open/mymongo/data:/data/db mongo --auth
```
**关于CMD & ENTRYPOINT**

在Dockerfile里同时定义ENTRYPOINT和CMD时，CMD的值将作为ENTRYPOINT的默认参数，而在run一个容器的时候，在镜像后添加的参数将会覆盖掉CMD的值而作为ENTRYPOINT的参数。

当我看到mongodb的Dockerfile:
```
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 27017
CMD ["mongod"]
```
一开始我有这样一个困惑:

以非--auth方式run时没有问题，但是当加上--auth时问题来了，这里默认的参数是mongod即运行mongo数据库服务，加上--auth后岂不是替换了mongod即:
```
docker-entrypoint.sh --auth
```
那么mongod又是怎么被启动的呢？

感谢stackoverflow前辈的解答，确实上面分析得没问题，奥秘就在这docker-entrypoint.sh里:
```
if [ "${1:0:1}" = '-' ]; then
	set -- mongod "$@"
fi
```
上面的shell命令作用是当识别到run时所加参数为-开头时，就会在其前加上mongod，这也难怪了!

### Node
在原有的node镜像上，构建自己的镜像:

#### Dockerfile:
```
FROM node:slim
RUN npm install -g cnpm --registry=https://registry.npm.taobao.org
WORKDIR /app
COPY ./package.json .
RUN cnpm install
COPY . .
RUN npm run build
ENTRYPOINT ["node","./server/index.js"]
```
这里其实更好的方式是将项目目录以数据卷方式挂载到容器里

#### 暴露端口
将node server暴露给宿主机:
```
$ docker run --name nodeserver -d -p 3000:3000 mynode:v1
```
其实这里根本是运行不了的，因为我们node server还没有与数据库连接，接下来会讲连接容器

### nginx
主要是更改默认的配置文件,这里的服务是反向代理到node服务的，所以并没有挂载项目文件:
```
FROM nginx:latest
COPY ./nginx.conf /etc/nginx/nginx.conf
EXPOSE 8080 80
```
同样运行的时候暴露端口

## 容器连接
上面是对项目用到的每个服务进行单独的说明，但是这几个服务之间是有联系的，要组建一个项目需要将它们进行连接。

其实很简单

以nginx和node连接为例:
```
$ docker run --name webserver -d -p 8080:8080 --link cnode:node mynginx:v1
```
这样node容器的3000端口就会在新起的nginx容器里有了映射,而且还会在新起的nginx容器/etc/hosts 加上一条解析,将cnode(node容器名),node(link别名)解析到node容器的ip。

来看看截图，进入nginx容器看看:
![link](http://7xsi10.com1.z0.glb.clouddn.com/dockerlink.jpg)

这样还有一个好处，如果不是最终需要暴露到宿主机的服务，它们的端口就不会被暴露，比如Mongodb的数据库端口27017就只会暴露给连接它的node服务，而不能被公网访问，这样比较安全。

然后我们就可以在nginx下将服务代理到node服务了:

```
upstream bbs {
      server  node:3000;
    }
server {
        listen       8080;
	server_name localhost;
        location / {
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Host $http_host;
          proxy_set_header X-NginX-Proxy true;
          proxy_pass http://bbs/;
          proxy_redirect off;
        }
}
```
## docker-compose项目

从上面可以发现，当一个项目需要多个服务配合完成时，一个一个手动管理连接是很麻烦的，Compose 定位是 “定义和运行多个 Docker 容器的应用“

### docker-compose.yml
以我的这个项目为例:
```yml
version: "2"
services:
    mongodb:
        image: mongo:latest
        expose:
            - "27017"
        volumes:
            - "/home/open/mymongo/data:/data/db"
        command: --auth
    nginx:
        build: /home/open/mynginx/
        ports:
            - "8080:8080"
            - "80:80"
        links:
            - node_server:node
    node_server:
        build: /home/laoqiren/workspace/isomorphic-redux-CNode/ 
        links:
            - mongodb:mongo
        ports:
            - "3000:3000"
        expose:
            - "3000"

```
ports定义向宿主机暴露端口，expose定义向其他服务暴露的用于连接的端口,links定义连接。

### 一键式启动项目

**先build各个镜像**

根据服务指定的build目录下的Dockerfile构建镜像:
![images](http://7xsi10.com1.z0.glb.clouddn.com/dockerimages.jpg)
看到docker-compose构建了nginx和node两个新镜像，而mongo是已经存在的，不用构建

**启动项目**
```
docker-compose up -d nginx
```
该命令十分强大，它将尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。

链接的服务都将会被自动启动，除非已经处于运行状态。

我们启动了nginx,nginx连接了node,则node启动，而node又连接了mongo,mongo启动，这样整个应用就一键式启动了!
![done](http://7xsi10.com1.z0.glb.clouddn.com/upnginx.jpg)

Done!
## 结束语

更细节的研究有待继续。