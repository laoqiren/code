---
title: Haskell类型系统
date: 2017-03-16 09:32:24
tags: [Haskell,functional]
categories: 
- Haskell
---

这篇文章主要总结Haskell的类型系统。文章参照《Learn you a Haskell》。

<!--more-->

## 表达式类型

Int,Integer,Float,Double,Bool,Char,[]中各个项类型必须相同，()的类型跟其中的每一个元素的类型相关，只有每个元素对应类型相同，元素数目相同的两个Tuple类型才相同。

函数同样有类型声明。类型必须以大写开头。

## 类型参数

如`head`函数: `head :: [a] -> a`，表示与具体类型无关

## typeclass

如 `(==) :: Eq a => a -> a -> Bool`可以看到类型类约束`Eq a`,相当于interface定义行为

常见typeclass:

Eq:可判断相等性；Ord：可比较大小（必须先加入Eq)；Show：可以用字符串表示，Read：可以将一个字符串转换成某成员类型。（对于`read "4"`这种要加入类型注释:`read "4"::Int`）；Enum:可枚举，如`['a'..'e']`；Bounded:有一个上限和下限`minBound :: Int`: `-2147483648`；Num:具有数字特征，包含所有数字，如 `:t (*)`:`(*) :: (Num a) => a -> a -> a`；Integral:仅饱含整数，成员有:Int, Integer；Floating:仅包含Float,Double。

一个有用的函数`fromIntegral`：用于将Integral typeclass成员转换成更通用的`Num`　typeclass成员:
`fromIntegral :: (Num b, Integral a) => a -> b`，比如我们求一个List的length,再加上一个Float，就会报错，因为`length :: [a] -> Int`，我们可以`fromIntegral (length [1,2,3,4]) + 3.2`来解决。

## 构造自己的Types和Typeclasses

### ADT（代数数据类型）

如Bool:`data Bool = False | True`,False,True就是值构造子，更复杂的例子:

```haskell
data Shape = Circle Float Float Float | Rectangle Float Float Float Float
```
其中Circle,Rectangle两个值构造子后面跟着三项，实际为参数，因为值构造子相当于函数，如`Circle :: Float -> Float -> Float -> Shape`

通过` deriving`将类型派生为指定`typeclass`的成员

ADT的每个值构造子的每个参数必须有具体类型

### Record Syntax

例如表示一个人的type:
```haskell
data Person = Person String String Int Float String String deriving (Show)
```
分别表示firstName,lastName等等，我们要获取这些firstName啥的，还得这样:
```haskell
firstName :: Person -> String
firstName (Person firstname _ _ _ _ _) = firstname
```
显然麻烦，而解决方案:
```haskell
data Person = Person { firstName :: String
                     , lastName :: String
                     , age :: Int
                     , height :: Float
                     , phoneNumber :: String
                     , flavor :: String
                     } deriving (Show)
```
即自动创建了这些取值函数，如`firstName :: Person -> String`

### Type parameters

如`data Maybe a = Nothing | Just a`，`a`就是类型参数，这有点类似于C++的类模板，a可以是任意type。`Maybe`只是一个类型构造子，最终的type是根据a来的,如 `Just "hey" :: Maybe [Char]`。还有如`[]`，同样可以根据类型参数的实际type来构造不同的type：`[Char],[Int]`等。这种情况`read`的时候要加入完整的类型注释:`read "Just 't'" :: Maybe Char`

永远不给类型参数加typeclass约束

类型构造子和值构造子的区分是相当重要的。在声明数据类型时，等号=左端的那个是类型构造子，右端的(中间可能有|分隔)都是值构造子。

### Derived instances

派生为指定typeclass的成员，如
```haskell
data Day = Monday | Tuesday | Wednesday | Thursday | Friday | Saturday | Sunday
           deriving (Eq, Ord, Show, Read, Bounded, Enum)
```
可以:
```
[Thursday .. Sunday]
[Thursday,Friday,Saturday,Sunday]

[minBound .. maxBound] :: [Day]
[Monday,Tuesday,Wednesday,Thursday,Friday,Saturday,Sunday]
```

### Type synonyms

类型同义词:`type String = [Char]`

### Recursive data structures (递归地定义数据结构)

比如一个List:`5:6:8:[]`即[5,6,8]：
```haskell
infixr 5 :-:
data List a = Empty | a :-: (List a) deriving (Show, Read, Eq, Ord)
```
再比如一个二叉树：
```haskell
data BinaryTree a = Node a (BinaryTree a) (BinaryTree a)
                  | Empty
                  deriving (Show)

simpleTree = Node "parent" (Node "left child" Empty Empty)
                           (Node "right child" Empty Empty)
```

### 构造自己的typeclass和type instance

定义typeclass，如`Eq`:
```haskell
class Eq a where
    (==) :: a -> a -> Bool
    (/=) :: a -> a -> Bool
    x == y = not (x /= y)
    x /= y = not (x == y)
```
可以看到`==`和`x /= y`相互依赖定义，这样使得`Eq`的`minimal complete definition`成了`==`和`/=`其中一个，即只需要定义其中一个就能完整定义出`Eq`的instance了。比如:
```haskell
data TrafficLight = Red | Yellow | Green
instance Eq TrafficLight where
    Red == Red = True
    Green == Green = True
    Yellow == Yellow = True
    _ == _ = False
```

还可以定义sub typeclass:
```haskell
instance (Eq m) => Eq (Maybe m) where
    Just x == Just y = x == y
    Nothing == Nothing = True
    _ == _ = False
```

### Kind

用于类型的类型。对于有类型参数的构造子如`Maybe a = Nothing | Just a`这种，`:k Maybe`就是:`*->*`即接受一个具体类型返回一个具体类型。

这里具体类型指有具体形态的，比如一个函数的类型声明:`:k Int -> Int`结果是`*`，因为这是个具体的类型。对于没有类型参数的构造子，如`:k Int`结果就是`*`

类型构造子可以做curry:`:k Either String`结果是`Either String :: * -> *`。

就比如定义Functor的instance的时候，就必须接受一个`* -> *`类型>。

Functor:
```haskell
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```

看看这个类型:
```haskell
data Barry t k p = Barry { yabba :: p, dabba :: t k }
```
`Barry :: (* -> *) -> * -> * -> *`

要想它成为functor，就必须:
```haskell
instance Functor (Barry a b) where
    fmap f (Barry {yabba = x, dabba = y}) = Barry {yabba = f x, dabba = y}
```
因为`Barry a b`的Kind这个时候就是`* -> *`。


### 关键字"newtype"

将现有类型包装为新的类型。

比如对于`ZipList`:
```haskell
newtype ZipList a = ZipList {getZipList :: [a]}
```
不过是将`[]`进行了包装，而这么做的意义是，因为对原有类型的包装得来的新类型不会自动成为原类型定义的那些typeclass的instance。比如原有`[]`对于`Applicative instance`的`<*>`实现是一种交叉结合的方式，如果需要换种方式实现，就可以用我们的新的类型`ZipList`。

On newtype laziness

```haskell
newtype CoolBool = CoolBool {getCoolBool :: Bool}

helloMe :: CoolBool -> String
helloMe (CoolBool _) = "hello"
```
然后我们`helloMe undefined`结果正常为"hello",不同于`data`定义的时候抛出错误。

因为这里进行模式匹配的时候不用进行值构造子的计算。当我们使用 newtype 的时候，Haskell 内部可以将新的类型用旧的类型来表示。他不必加入另一层 box 来包住旧有的类型。他只要注意他是不同的类型就好了。而且 Haskell 会知道 newtype 定义的类型一定只会有一个构造子，他不必计算喂给函数的值就能确定他是 (CoolBool _) 的形式，因为 newtype 只有一个可能的值跟单一字段！

## 关于多态
与其他语言进行对比总结。

C++中：
多态分为编译时多态（静态多态）和运行时多态（动态多态），而真正意义上的多态实际上只有运行时多态，通过虚函数和继承实现，即属于晚绑定，在运行时函数调用地址才能确定。

针对编译时多态，有模板，函数（操作符）重载(即Ad-hoc多态），模板特化，Duck Typing。

运行时多态：[http://blog.csdn.net/hackbuteer1/article/details/7475622](http://blog.csdn.net/hackbuteer1/article/details/7475622)

haskell中：

**参数化多态：**

Prelude> :type last
last :: [a] -> a

可以看到last函数的类型签名包含类型变量，不在意具体的类型，对应C++的模板（函数模板，类模板）即泛型实现

**ad-hoc多态：**
这里的ad-hoc多态与C++有一定区别，因为在haskell中通过type class来实现，而type class又类似于C++的虚函数，每个instance对type class定义的函数都有自己的实现。这勉强叫做“函数重载”，这或许就是它被称为ad-hoc多态的原因。而在C++中ad-hoc多态和虚函数多态完全不一样的。

## 其他
实践过程中经常碰到类型转换问题，由于强类型的原因，遇到了很多麻烦，这里占个坑，需要补充这方面的解决方案。

![http://upload-images.jianshu.io/upload_images/1023733-27c30ca1b9b1af69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240](http://upload-images.jianshu.io/upload_images/1023733-27c30ca1b9b1af69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 数值类型转换：

## 总结
Haskell有一套强大的类型系统，虽然是functional,OO的有些思想却在其中有一些体现。

详情参考书籍[https://learnyoua.haskell.sg/content/zh-cn](https://learnyoua.haskell.sg/content/zh-cn)

