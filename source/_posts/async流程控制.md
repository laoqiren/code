---
title: async流程控制
date: 2016-08-30 21:17:51
tags: [异步编程]
categories: 
- JavaScript
---

async是一个异步操作工具集，其集合操作完美地融合了函数式编程。其各种流程控制API适合各种应用场景。本文总结一些常用的。

<!--more-->

## async.series()

前后无依赖地串行执行：
```js
async.series([
    async.apply(fs.readFile,path.join(__dirname,"../assets/txt.txt"),'utf-8'),
    async.apply(fs.readFile,path.join(__dirname,"../assets/txt2.txt"),'utf-8'),
    async.apply(fs.readFile,path.join(__dirname,"../assets/txt4.txt"),'utf-8')
],(err,results)=>{
    console.log(results);
})
// [ 'hello\nworld', 'hello,I\'m showData', 'yohe,I\'m txt4.txt' ]
```
这个相当于串行版本的`Promise.all()`,这里`async.apply()`类似于`_.curry()`；

## async.waterfall()

前后可多依赖的串行执行:
```js
async.waterfall([
    async.apply(fs.readFile,path.join(__dirname,"../assets/txt.txt"),'utf-8'),
    function(content,cb){
        cb(null,"hello,I'm showData");
    },
    function(content,cb){
        console.log(content); //hello,I'm showData
        fs.writeFile(path.join(__dirname,"../assets/txt3.txt"),content,err=>{
            cb(null,"hello,I'm writeData");
        })
    }
],(err,result)=>{
    console.log(result); //hello,I'm writeData
})
```
这个相当于`Promise.then()`链式调用。

## async.parallelLimit()

可控制并发量的并行执行：（async.reflect()用于包裹任务，使其发生错误时不会影响其他任务执行，错误信息将作为最后结果之一）：

```js
async.parallelLimit([
    async.reflect(async.apply(fs.readFile,path.join(__dirname,"../assets/txt.txt"),'utf-8')),
    async.reflect(async.apply(fs.readFile,path.join(__dirname,"../assets/notExist.txt"),'utf-8')),
    async.reflect(async.apply(fs.readFile,path.join(__dirname,"../assets/txt4.txt"),'utf-8'))
],2,(err,results)=>{
    if(err) console.log(err.stack);
    console.log(results);
    /*
     [  { value: 'hello\nworld' },
        { error: 
            { Error: ENOENT: no such file or directory...} 
        },
        { value: 'yohe,I\'m txt4.txt' } ]
  */

})
```
或者`reflectAll()`改进版：
```js
let tasks = [
    async.apply(fs.readFile,path.join(__dirname,"../assets/txt.txt"),'utf-8'),
    async.apply(fs.readFile,path.join(__dirname,"../assets/txt2.txt"),'utf-8'),
    async.apply(fs.readFile,path.join(__dirname,"../assets/txt4.txt"),'utf-8')
];

async.parallelLimit(async.reflectAll(tasks),2,(err,results)=>{
    if(err) console.log(err.stack);
    console.log(results);
});
```

## async.queue()

可控制并发量的，且可动态添加任务的并发执行：
```js
let q = async.queue((file,cb)=>{
    //console.log(`hello,${file}`);
    fs.readFile(dirName + file,'utf-8',cb);
},2)

q.drain = function(){
    console.log("all tasks have completed!");
}

fs.readdirSync(dirName).forEach(file=>{
    q.push(file,(err,data)=>{
        console.log(data)
    })
})
```

## async.compose()

类似于单依赖地串行执行：
```js
function readData(fileName,cb){
    fs.readFile(path.join(__dirname,"../assets/" + fileName),'utf-8',cb);
}

function showData(content,cb){
    cb(null,"Hello,I'm showData");
}

function writeData(content,cb){
    fs.writeFile(path.join(__dirname,"../assets/txt3.txt"),content,err=>{
        cb(null,"hello,I'm writeData");
    })
}

let readShowWrite = async.compose(writeData,showData,readData);

readShowWrite("txt.txt",(err,result)=>{
    console.log(result); // hello,I'm writeData
})
```

## async.times()

针对同一异步操作并发执行指定次数。这里使用`timesLimit`控制并发量，一个抓取CNode论坛帖子的例子：
```js
const async = require("async");
const superagent = require('superagent');
const cheerio = require('cheerio');
const url = require('url');

const cnodeUrl = 'https://cnodejs.org/';

function afterGetTopicUrls(topicUrls){
    async.timesLimit(topicUrls.length,5,(n,next)=>{
        let topicUrl = topicUrls[n];
         superagent.get(topicUrl)
             .end((err,res)=>{
                 next(null,[topicUrl,res.text]);
             })
     },(err,topics)=>{
         // 开始行动
        topics = topics.map(function (topicPair) {
            var topicUrl = topicPair[0];
            var topicHtml = topicPair[1];
            var $ = cheerio.load(topicHtml);
            return ({
            title: $('.topic_full_title').text().trim(),
            href: topicUrl,
            comment1: $('.reply_content').eq(0).text().trim(),
            });
        });
    
        console.log('final:');
        console.log(topics);
    });
}

superagent.get(cnodeUrl)
  .end(function (err, res) {
    if (err) {
      return console.error(err);
    }
    var topicUrls = [];
    var $ = cheerio.load(res.text);
    // 获取首页所有的链接
    $('#topic_list .topic_title').each(function (idx, element) {
      var $element = $(element);
      var href = url.resolve(cnodeUrl, $element.attr('href'));
      topicUrls.push(href);
    });

    console.log(topicUrls);
    afterGetTopicUrls(topicUrls);
  });
```

## async.map()

上面的抓取例子同样可以用`async.mapLimit()`改写（加入了并发量监测）:
```js
var concurrencyCount = 0;

const cnodeUrl = 'https://cnodejs.org/';

function fetchUrl(url,cb){
    concurrencyCount++;
    console.log('现在的并发数是', concurrencyCount, '，正在抓取的是', url);
    superagent.get(url)
    .end((err,res)=>{
        concurrencyCount--;
        cb(null,[url,res.text]);
    })
}

superagent.get(cnodeUrl)
  .end(function (err, res) {
    if (err) {
      return console.error(err);
    }
    var topicUrls = [];
    var $ = cheerio.load(res.text);
    // 获取首页所有的链接
    $('#topic_list .topic_title').each(function (idx, element) {
      var $element = $(element);
      var href = url.resolve(cnodeUrl, $element.attr('href'));
      topicUrls.push(href);
    });

    async.mapLimit(topicUrls, 5, function (url, callback) {
        fetchUrl(url, callback);
      }, function (err, topics) {
            topics = topics.map(function (topicPair) {
            var topicUrl = topicPair[0];
            var topicHtml = topicPair[1];
            var $ = cheerio.load(topicHtml);
            return ({
            title: $('.topic_full_title').text().trim(),
            href: topicUrl,
            comment1: $('.reply_content').eq(0).text().trim(),
            });
        });
    
        console.log('final:');
        console.log(topics);
      });
  });
```

运行截图：

![http://7xsi10.com1.z0.glb.clouddn.com/crawler.png](http://7xsi10.com1.z0.glb.clouddn.com/crawler.png)

## async.auto()

这个可以所是最灵活的流程控制方法了，可以根据依赖情况进行流程控制:

沿用官方例子:
```js
async.auto({
    get_data: function(callback) {
        console.log('in get_data');
        // async code to get some data
        callback(null, 'data', 'converted to array');
    },
    make_folder: function(callback) {
        console.log('in make_folder');
        // async code to create a directory to store a file in
        // this is run at the same time as getting the data
        callback(null, 'folder');
    },
    write_file: ['get_data', 'make_folder', function(results, callback) {
        console.log('in write_file', JSON.stringify(results));
        // once there is some data and the directory exists,
        // write the data to a file in the directory
        callback(null, 'filename');
    }],
    email_link: ['write_file', function(results, callback) {
        console.log('in email_link', JSON.stringify(results));
        // once the file is written let's email a link to it...
        // results.write_file contains the filename returned by write_file.
        callback(null, {'file':results.write_file, 'email':'user@example.com'});
    }]
}, function(err, results) {
    console.log('err = ', err);
    console.log('results = ', results);
});
```

## 总结

异步流程控制方案有许多，包括`eventproxy`事件订阅的思路，`connect`模块的`next`尾触发的方案等。async支持函数式的集合操作，流程控制，和一些常用工具，支持并发控制等。