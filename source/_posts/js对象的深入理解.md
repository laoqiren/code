---
title: js对象的深入理解
date: 2016-07-13 09:47:01
tags: [JavaScript,面向对象]
categories: 
- JavaScript
---
JavaScript作为一门非常灵活的动态语言，可以有许多不同的编程范式，而面向对象也是其中之一，但是对比传统的面向对象语言，Js有着很多不同，比如原型链的继承方式等等。由于很多容易混淆的概念问题，这篇文章总结一下我在学习和实践当中对js对象的理解。
<!--more-->

###### 从刀耕火种到工程化开发，从es5到es2015+，语言本身在不断完善，es2015的Class便是将js创建对象的最佳模式和继承的最佳方式规范化，实质还是基于原型对象和原型链，这样就减轻了开发人员的负担，这便是语言的进步吧。

### 一. js里的对象是什么？

#### js的七种数据类型：

基本数据类型：Undefined,Null,Boolean,Number,String，Symbol(es6新增);

引用数据类型：Object

##### 它们之间的联系：

我们有一个typeof操作符可以判断数据类型，有两个特性情况：
```js
typeof null //object
typeof function //function
```
可见虽然null是一种基本类型值，但是它表示空的对象引用，这里就有个问题：

对象为空和空对象引用的区别？

###### 对象为空: 
```js
let obj = {};
```
看似空空，其实非也，不要忘了，这个对象是继承自Object类型的，而Object原型上又有许多方法和属性，比如toString()等；

###### 空对象引用：

可以类比C/C++的空指针吧，没有任何指向

你经常会看到：

```js
const str = 'hello luoxia's blog';
console.log(str.length);
```
你会想，奇了怪了，这个str明明是基本数据类型啊，并非对象，为何它会有length属性呢？
听我慢慢道来：

我们在访问str的length属性时，其实内部机制是这样的：

1. 创建String基本包装类型的实例；
2. 在实例上调用这个方法
3. 销毁这个实例

这你就应该明白了，并不是说str就是对象，而是内部调用了它的基本包装类型实例，基本包装类型实例是对象，当然有属性和方法了。就好比 2.toString();并不是说2是对象，而是它的基本包装类型Number类型的作用呢

###### js中一切皆对象，我的理解：

对于Object类型毫无疑问，js中数组，函数都是对象，比如数组对象：
```js
const arr = [1,0,2,4]
//等价于:{
	0:1,
	1:0,
	2:2,
	3:4,
	length:4
	}
```
而对于基本类型的值，就像刚刚说的，并不是说2就是对象，而是说我吗可以利用他的包装类型对它进行一系列的属性和方法的访问。

对于es6新增的symbol值，我现在还有个疑问，Symbol值非对象，但是它也有一系列方法和属性，则不就类似的表面它也有其包装类型，那么这个包装类型是什么呢？Symbol? No,我们在创建Symbol值的时候是这样子的：
```js
let sy = Symbol('describtion');
```
可以看到并不能new Symbol创建基本包装类型

### 二. 让我们一起new对象吧（这个我的理解就是封装）

#### 两种方式：

1. new+构造函数+()
```js
let obj = new Object();
```
2. 对象字面量
```js
let obj = {
	name:'luoxia'
	}
```
这个构造函数可以是语言本身就有的，就比如Number这个基本包装类型，也可以是自定义的构造函数，而我要讲的主要是自己定义的构造函数，毕竟在实际应用中，往往需要自定义类型满足需求。

es5及之前版本，js是没有class(类）的说法的，不想java那种，es当中只有构造函数的概念，这在开始，是有些难以理解的，es2015当中，新增了class关键字，但其内部的原理，比如原型继承都是类似的，只是加了语法糖便于更好的理解而已。

#### 创建对象：

先举个栗子
```js
function Obj(name,age){
	this.name= name;
	this.age = age;
	this.sayName = function(){
		console.log(this.name);
	}
}
var obj1 = new Obj('luoxia',19);
var obj2 = new Obj('Jane',18);
......
```
这个和java某个类中的构造函数是那么的相似：
```js
public class Obj {
	private String name;
	private Int age;
	Obj(String name,int age){
		this.name = name;
		this.age = age;
	}
	public void sayName(){
		System.out.println(this.name);
	}
}
```
js的构造函数可以在this对象上直接加方法哎。。。这是外观上的差别，而理解js对象，关键是要理解两个核心：原型对象和原型链，而原型链是继承的实现机制，我们先来看看原型对象：

##### 原型对象：

这个东西最好是有图解，但是好麻烦，我还是总结一下我的理解算了吧，需要图解的可以看书或者其它文章

对于任意一个构造函数（函数），就如上例的Obj,函数也是对象，而函数会自动拥有一个属性，那便是prototype属性，它的值是一个指针，这个指针指向哪里呢，指向的便是这个函数的原型对象，prototype，这个原型对象拥有一个默认的属性constructor,其指向原型对象的拥有者，即构造函数本身:
```js
console.log(Obj.prototype.constructor)//Obj
```
上述的构造函数的this指向哪里呢？this的话题又是一个很多坑的话题，this的定义本来就是函数赖以执行的环境，当我们通过构造函数实例化一个对象的时候，这个this就指向了这个实例对象，这个实例对象就拥有了上例中初始化的属性和方法，而这些属性和方法都在哪里呢？就是在实例对象上，

对，没错，这些属性和方法由于通过构造函数的方式初始化，当new Obj()的时候，便初始化给了实例对象，而除了这些自定义初始化的属性和方法，实例对象内部有一个属性：\__proto__，这个属性的值也是一个指针，它指向了构造函数的原型对象prototype:
```js
console.log(obj1.__proto__)//Obj.prototype
```
下面几个情形:
```js
Object.__proto__//Function.prototype
Function.__proto__//Function.prototype
Function.prototype.__proto__//Object.prototype
Object.prototype.__proto__//null
```
可见，Object构造函数是Function的实例，而Function的原型对象又是Object的实例，即Function继承于Object，任何构造函数，其本身作为对象，它们都是Function的实例，当然\__proto__指向Function

我们会发现，基于构造函数的方式创建对象有一个不足，那就是对于一些公用的方法和属性，我们在实例化不同的对象时，会创建多个本来可以公用的方法，这不就是一种浪费吗？

再来看看下面的例子：
```js
function Obj2(){
}
Obj2.prototype.name = 'luoxia';
Obj2.prototype.age = 19;
Obj2.prototype.sayname = function (){
	console.log(this.name);
}
```
这个便是基于原型的方式创建对象，所谓基于原型，便是将属性和方法初始化给构造函数的原型对象，而所有通过这个构造函数实例化的对象将共享这些属性和方法，共享有木有，这就有一个问题：
```js
Obj2.prototype.arr = [1,2,3];
let obj1 = new Obj2();
let obj2 = new Obj2();
obj1.arr.push(4);
console.log(obj2.arr);//[1,2,3,4]
```
有没有发现，两个不同的实例，对一个实例的数据修改，本来不应该反映到另一个实例，但由于它们共享了这个数组的引用，这就导致它们互相牵连，所以，这也是原型创建对象方法的一个不足。

其中有一个问题要注意，当用对象字面量改变构造函数的原型对象时，会改写原型对象的 constructor属性指向：
```js
Obj2.prototype = {
	name:'luoxia',
	age:19
}
console.log(Obj2.prototype.constructor)//Object
```
本该如此，因为此时的原型对象本来就是Object的实例

我们可以手动的设置它的constructor属性为构造函数

#### 创建对象的几种模式

1. 工厂模式，即在一个函数内部新建一个对象，初始化后返回这个对象，封装内部细节，返回，但是这个模式是无法识别对象类型的。。。
2. 构造函数模式，上例
3. 原型模式，上例
4. 组合使用构造函数模式和原型模式:

可见，构造函数模式和原型模式都有自己的不足，我们可以组合使用它们，这个模式也是最常用的模式:
```js
function Person(name,age){
	this.name = name;
	this.age = age;
	this.arr = [name,age];
}
Person.prototype.sayname = function(){
	console.log(this.name);
}
let person1 = new Person('luoxia',19);
let person2 = new Person('jane',18);
person1.arr.push('add');
console.log(person2.arr)//['jane',18];
```
我们将公用的sayname方法初始化到原型对象上，将实例属性（包括一些引用类型的属性）初始化到实例对象上，可见，各个实例属性的引用类型就不会互相影响了。

这个模式既做到了公共方法和属性的 共享，又做到了私有属性的互不干扰，而且还可以通过构造函数传递参数，这确实是一个不错的创建对象模式。

5. 其他模式：动态原型模式，寄生构造函数模式，稳妥构造函数模式，适用于特定场景。

### 三.继承 

js的继承和其他语言不同，她是基于原型链的继承。用个例子说话：
```js
function Super(){
	this.superName = 'super';
}
function Sub(){
	this.subName = 'sub';
}
Sub.prototype = new Super();
let super = new Super();
let sub = new Sub();
console.log(sub.superName);//super
```
这就是一个继承的例子，Sub的原型对象是Super的实例，这样，在Super实例对象的属性就成为了Sub原型对象的属性，Sub的实例就可以对它进行访问，而此时Super原型对象就有了一个内部属性\__proto__指向了Super的原型对象，而subName属性存在于Sub的实例对象当中。

这样一层层的继承，便形成了一条原型链，处于顶端的是Object类型。

上述例子便是继承的一种模式，基于原型链的模式，这个存在一个类似于原型创建对象模式的问题,那便是引用类型的互相影响:
```js
function Super(){
	this.arr = [1,2,3];
}
function Sub(){
	
}
Sub.prototype = new Super();
let sub1 = new Sub();
let sub2 = new Sub();
sub1.arr.push(4);
console.log(sub2.arr);//[1,2,3,4];
```
发现子类型的两个不同实例的引用类型即这里的数组值会互相影响，这是由于这个数组属性是存在于Sub原型对象内的，这样Sub实例对象就共享了这个数组，当然会互相影响。

#### 继承的方式

1. 原型链的方式，上例；
2. 借用构造函数:

```js
function Super(){
	this.arr = [1,2,3];
}
function Sub(){
	Super.apply(this);
}
let sub1 = new Sub();
let sub2 = new Sub();
sub1.arr.push(4);
console.log(sub2.arr);//[1,2,3];
```
原理就是在new Sub()的时候，实际上是执行了内部的Super.apply(this)，这样Super函数执行，内部的this又指向的是Sub的实例对象，自然而然的就让所有父类型构造函数的属性和方法给了子类实例，但是这个方式如同构造函数模式创建对象一样，无法公用属性和方法，浪费。
3. 组合继承:

可以这么想，和组合方式的创建对象模式类似的思想。
```js
function Super(){
	this.arr = [1,2,3];
}
Super.prototype.sayName(){
	console.log('luoxia');
}
function Sub(){
	Super.apply(this);
}
Sub.prototype = new Super();
let sub1 = new Sub();
let sub2 = new Sub();
sub1.arr.push(4);
console.log(sub2.arr);//[1,2,3];
```
看看上面的例子，内部机制是如何的?我们让Sub的原型对象等于Super的实例，这样Super构造函数定义的属性就存在于Sub的原型对象上，而Super.apply(this)则让Super构造函数再次执行，而这时，this指向的是Sub实例对象，所以Super构造函数的属性就存在于实例对象上了，好嘞，这下在Sub原型对象上有个arr属性，在Sub的实例上有个arr属性，由于实例上的同名属性会优先于原型对象上的属性，这样我们访问的就是实例的属性了，当然互不影响，而且对于不同的Sub实例，它们现在都可以访问sayName方法了，因为这个方法存在于Super的原型对象中，可以共享。

4. 原型式继承，即Object.create()
```js
let obj = Object.create({name:'luoxia'});
```
其实就相当于 
```js
function object(o){
	let obj = {};
	obj.__proto__ = o;
}
let obj = object();
```
就是将参数对象作为子类型的\__proto__值，好比其构造函数的原型对象。这个对于引用类型的属性，同样会互相影响

5. 寄生式继承，和原型式类似，不过是在内部对新对象加了一些扩展再返回而已。


6. 寄生组合继承:


这个是对组合继承方式的一种改良，在前面的组合继承方式中,存在一个不足，那便是我们既在子类型的原型对象上创建了属性，又在实例对象上创建了同名属性，那么原型对象上的那些属性岂不是浪费时间浪费金钱的产物？而这个方式就是解决这个问题的：
```js
function inheritPrototype(sub,super){
	let prototype = object(super.prototype);
	prototype.constructor = sub;
	sub.prototype = prototype;
}


function Sup(){
	this.arr = [1,2,3];
}
Sup.prototype.sayName(){
	console.log('luoxia');
}
function Sub(){
	Sup.apply(this);
}
inheritPrototype(Sub,Sup); //实际上相当于Sub.prototype.__proto__ = Sup.prototype
let sub1 = new Sub();
let sub2 = new Sub();
sub1.arr.push(4);
console.log(sub2.arr);//[1,2,3];
```
大家可以看到，这个例子并没有显式地设置Sub的原型对象，而是将Super的原型对象复制了一份给Sub，这样的话在Super实例上的属性当然不会跑到Sub上了，这个模式最理想。

### 四.多态

在java中，所谓多态，便是某个对象既属于子类型，又属于父类型，js中也有类似的概率，instanceof操作符就是用来判断实例和原型的关系：
```js
console.log(sub1 instanceof Object);//true
console.log(sub1 instaceof Sub);//true
console.log(sub1 instanceof Super);//true
可见，sub1是属于子类型，父类型，也是Object类型的
```
### 五.ES2015当中的Class及继承

直接上栗子：
```js
class A{
    constructor(name){
        this.name = name;
    }
    get test(){
        return this.name;
    }
    set test(name){
        this.name = name;
        return 'setter';
    }
}
class B extends A{
    constructor(name,age){
        super(name);
        this.age = age;
    }
	static sayAge(){
		console.log('age');
	}
}
let b = new B('luoxia',20);
let a = new A('jane');
console.log(b.test);//luoxia
b.test = 'jack';
console.log(b.test);//jack
console.log(B.sayAge);//age
```
#### class的对象创建：

es2015中的class的类型就是function,但是不能直接执行，只能new实例，而constructor就是它的构造函数，即es5中的构造函数就好比es2015 clss的constructor。

虽然加了语法糖，其实内部机制类似，class内的方法（非this指定）的都会存在于类的原型对象上，即上述class A：
```js
A.prototype = {
	get test(){
		...
	}
	...
}
console.log(b.constructor === B.prototype.constructor);//true
console.log(B === B.prototype.constructor);//true
console.log(b.__proto__);//B.prototype
```
而通过构造函数初始化的属性和方法存在于类的实例化对象上

#### class的继承

class的继承其内部还是原型的继承，不过有区别

上例中，A和B存在两条继承关系：

类作为对象的时候：
```js
B.__proto__ = A;
```
作为构造函数：

	B.prototype.__proto__ = A.prototype;
主要是由于在创建实例的时候，内部是这样运行的：

```js	
// B的实例继承A的实例
Object.setPrototypeOf(B.prototype, A.prototype);

// B继承A的静态属性（方法）（static，即直接通过类名调用）
Object.setPrototypeOf(B, A);
```
经过检验：
```js
console.log(b.hasOwnProperty('name'));//true
console.log(b.__proto__.hasOwnProperty('name'));//false
```	
说明name是存在于b实例对象上的,B的原型对象上没有这个name属性。

#### es2015和es5继承的联系

发现它们的联系了吗？前面讲es5的继承方式有个组合继承方式和寄生组合方式，两种的区别就在于，组合继承方式会在子类型原型上创建多余的属性，而后者，我也在代码里说过，其实质就是设置：
```js
Sub.prototype.__proto__ = Sup.prototype
```
而这个，正好是es2015 class的继承关系。

聪明的你会想，我们在
```js		
Sub.prototype = new Sup()；
```
时不就是让Sub.prototype.\_\_proto\_\_等于Sup.prototype吗，没错，其结果确实是这样，但是用new的时候我们就执行了构造函数，让内部实例属性存在于prototype里了，而我们直接设置Sub.prototype.\_\_proto__ = Sup.prototype只是创建prototype的实例

#### es5和es2015继承的区别：

1. 对于父类型的构造函数，es5通过apply()或者call()方法指定其在子类型的实例上运行，这里是先有子类型实例的this,再去调用父类型构造函数的，而es2015就不一样了，先要通过super()的运行，才有this,即子类型的构造函数内的this是父类型构造函数给与的，指向子类型实例，所以必须先调用super，才能使用this
2. es5子类型实例的constuctor默认指向的是父类型构造函数，而es2015指向子类型构造函数：
```js	
console.log(b.constructor);//Function B
console.log(sub.constructor);//Function Sup
```	
###### es2015的class实质上是和es5差不多的，不过让书写方式更于理解，而且提供了最佳实践的创建对象的方式，就是使用组合构造函数和原型模式来创建对象，而且继承也是最合理的寄生组合方式，这就让开发人员不必去自我揣度最好的方案，让精华成为约定俗成，，这大概就是语言的不断进步吧，希望js变得越来越强大，保持进步，未来会更好！

#### 啊，这算是目前最长的一篇文章了吧，头都大了，js中的坑真是又大又多，但相信我们会在总结中不断成长，文中有什么欠妥的地方，渴望大家的建议。