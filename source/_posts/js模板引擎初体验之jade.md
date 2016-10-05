---
title: js模板引擎初体验之jade
date: 2016-04-22 19:22:42
tags: [js模板引擎,jade]
categories:
- JavaScript
---
#### 序言
jade是一款基于NodeJS的js模板引擎，采用缩进的风格书写html,使得我们可以不用关注标签的闭合，模板引擎可供服务端渲染和客户端渲染，模板引擎能够将数据和视图分离，通过动态的传入数据，模板引擎可以渲染出结构相同，数据有差异的页面。
 <!--more-->
### 因为nodejs而接触了jade模板引擎，它可以提高前端开发的效率
#### 这篇博文我想写一下利用jade模板引擎更好的和后台对接，动态生成html页面

之前的项目中，我是这么干的：通过ajax获取服务器返回的json数据,然后通过拼接字符串的形式来生成dom，再插入到页面。

jade在html元素内容中引入变量
```html
p #{content}  //content变量表示文章内容
```
通过each循环变量数组或对象
```html
- var array = ['jack','jane','marry']
- var obj = {name:'luoxia',girlfriend:'null'}
ul
	each item in array
		li item

each key,value in obj
	p #{key}:#{value}
```
输出结果":

    <ul>
		<li>jack</li>
		<li>jane</li>
		<li>marry</li>
	</ul>
    <p>name:luoxia</p>
	<p>girlfriend:null</p>

现在加入要从服务器获取某个表格内容，而这个表格的内容是不断变化的，而且表格项数不确定，我们可以通过ajax获取一个包含表格内容的json对象（这里运用bootstrap框架)

	//backData:
    {
		sections:[{class:'active',name:'罗峡',addr:'中国重庆'},{class:'info',name:'小明',addr:'火星'},
			{class:'warning',name:'大明',addr:'遥远的地方'}]
	};
书写jade模板

	//table.jade
    table(class='table table-bordered table-hover')
    thead
        tr
            td 姓名
            td 地址
    tbody
        each section in sections
            tr(class='#{section.class}')
                td #{section.name}
                td #{section.addr}
编译成模板js脚本

    $ jade --client --no-debug table.jade
	rendered table.js
在页面中引入jade的runtimejs文件和table.js文件,最后通过将json传递给table.js暴露出的方法

    var addHtml = template(backData);
    document.body.innerHTML = addHtml;

这样，动态生成格式类似的代码块就完成了。一个漂亮的表格:
![table](http://7xsi10.com2.z0.glb.clouddn.com/table.png)

在es2015语法中新增了一种叫做模板字符串的字符串字面量，模板字符串以反引号代替一般字符串的引号和双引号，变量或者表达式通过占位符${}语法来引入。但模板字符串本身不支持条件或循环等模板引擎类似的语法:
		
		ul
			for (var k in lis)
				li= lis[k]

嗯，但是es2015里有个东西叫做标签模板的东西，什么叫标签模板呢，就是一个标签（实际上是一个函数）后跟上模板字符串，而这个模板字符串会被作为参数传入前面的标签函数，第一个参数是未被占位符替换的部分组成的数组，而后面的参数都是各个变量，我们可以通过标签模板自定义功能：

	let lis = ['luoxia is good','luoxia is very good','luoxia can\'t be more good'];
	function addLis(s,...array){
	    let string = '';
	    string += s[0];
	    for(let i=0; i<array[0].length;i++){
	        string +=
	        `
	        <li>
	            ${array[0][i]}
	        </li>`;
	    }
	    string += s[1];
	    return string;
	}
	console.log(addLis `<ul>
	            ${lis}</ul>`);
		/*
				$ node es2015.js
	<ul>
	
	        <li>
	            luoxia is good
	        </li>
	        <li>
	            luoxia is very good
	        </li>
	        <li>
	            luoxia can't be more good
	        </li></ul>
		*/

不过略显麻烦啊感觉有木有（或许是我实现得麻烦）

前端技术瞬息万变，模板引擎的时代或许早已过去，但是我们可以在学习中去体会它的思想，发现它的不足，在对比中成长，然后在实践中找到更适合具体场景的技术，不在追逐中迷失自我，而是让自己的心智，能力得到成长。（菜鸟的自白，不喜勿喷）