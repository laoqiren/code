---
title: gulp前端自动化
date: 2016-04-26 16:16:05
tags: [gulp,前端自动化]
categories: 
- 前端工程
---
#### 序言
在项目开发过程中，我们会遇到许多重复枯燥的操作，比如当代码修改时，刷新浏览器，代码压缩，编译文件等等，这会降低我们的开发效率，而gulp就是一款前端自动化开发工具，采用pipe流机制，能够对指定文件进行处理，并且能够实时监控文件变化而自动执行相应的任务。这篇文章记录一下gulp的配置使用，这段序言也是后来加上的。
<!--more-->
#### gulp是前端开发过程中对代码进行构建的工具，是自动化项目的构建利器；她不仅能对网站资源进行优化，而且在开发过程中很多重复的任务能够使用正确的工具自动完成；使用她，我们不仅可以很愉快的编写代码，而且大大提高我们的工作效率。

分享我在使用过程中的心得

使用gulp：

1. 初始化项目，package.json可以帮助我们管理依赖，一个node package有两种依赖，一种是dependencies一种是devDependencies，其中前者依赖的项该是正常运行该包时所需要的依赖项，而后者则是开发的时候需要的依赖项，像一些进行单元测试之类的包。
```
$ npm init
```
2. 全局安装gulp：
```
$ npm install -g gulp
```
3. 局部安装gulp:

    	$ npm install --save-dev gulp

4. 安装依赖插件:

    	$ npm install --save-dev gulp-ruby-sass

5. 我的配置:

```js				
'use strict';

//引入依赖模块
var gulp = require('gulp'),
   uglify = require('gulp-uglify'),
   babel = require('gulp-babel'),
   watch = require('gulp-watch'),
   sass = require('gulp-ruby-sass'),
   autoprefix = require('gulp-autoprefixer'),
   rename = require('gulp-rename'),
   concat = require('gulp-concat'),
   browserSync = require('browser-sync');

//编译sass到css
gulp.task('sass', function () {
  return sass('app/sass/*.scss')
    .on('error', sass.logError)
    .pipe(gulp.dest('build/css/'));
});

//合并多个js文件,压缩，并重命名
gulp.task('concat', function () {
    gulp.src('build/js/*.js')
        .pipe(concat('all.js'))
        .pipe(uglify())
        .pipe(rename('all.min.js'))
        .pipe(gulp.dest('dist/js'));
});

// 自动补全css3浏览器兼容前缀
gulp.task('styles', function() {
  gulp.src(['./build/css/test.css'])
    .pipe(autoprefix('last 2 versions'))
    .pipe(gulp.dest('./dist/css'));
});

//babel转换es2015到es5
gulp.task('babel',function(){
   return gulp.src("app/js/*.js")// ES6 源码存放的路径
    .pipe(babel({presets: ['es2015']}))
    .pipe(gulp.dest("build/js")); //转换成 ES5 存放的路径
   });

//监控es2015文件，sass文件，自动编译为es5文件，css文件
gulp.task('watch', function () {
   gulp.watch(['app/js/es2015.js','app/sass/*.scss'], ['babel','sass']);
});

//浏览器实时自动更新状态，不用手动刷新
gulp.task('browser-sync', function () {
   var files = [
      'app/**/*'
   ];
   browserSync.init(files, {
      server: {
         baseDir: ['./app/html','./app/']
      }
   });
});
```
