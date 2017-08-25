---
title: 从Functors到Monad
date: 2017-08-17 09:35:33
tags: [Haskell,functional,Monad,Functor,Applicative,Monoid]
categories: 
- Haskell
---
从Functors到Monad，理解Functor,Applicative,Monoids,Monad。文章理解或有偏差，仅作为自己总结用。

<!--more-->
文章参照《Learn you a Haskell》和《mostly　adequate　guide》

## Functor typeclass

先来看看haskell中其type class定义:
```haskell
class Functor (f :: * -> *) where
  fmap :: (a -> b) -> f a -> f b
  (<$) :: a -> f b -> f a
```
从f的kind我们可以看到，其不是一个具体类型，而是一个接受具体类型来构造具体类型的类型构造子（参照我的另一篇文章"Haskell类型系统"）。

在js中的实现：
```js
let Container = function(x) {
  this.__value = x;
}

Container.of = function(x) { return new Container(x); };

Container.prototype.fmap = function(f){
  return Container.of(f(this.__value))
}
```
这里暂时可以简单地把Functor理解为容器，将函数运用到容器内的值上面，再将处理结果用容器进行包装。

### List Functor

以`[]`为例，看看`[]`的`Functor instance`实现:
```haskell
instance Functor [] where
    {-# INLINE fmap #-}
    fmap = map
```
可以看到`fmap`直接就是`map`，这个不难理解：`map :: (a -> b) -> [a] -> [b]`。

### Maybe Functor
```haskell
instance  Functor Maybe  where
    fmap _ Nothing       = Nothing
    fmap f (Just a)      = Just (f a)
```
总之接受一个通过类型构造子包装的具体类型数据，最终返回的也要是经过它封装的具体类型数据。

js中实现:
```js
let Maybe = function(x) {
  this.__value = x;
}

Maybe.of = function(x) {
  return new Maybe(x);
}

Maybe.prototype.isNothing = function() {
  return (this.__value === null || this.__value === undefined);
}

Maybe.prototype.fmap = function(f) {
  return this.isNothing() ? Maybe.of(null) : Maybe.of(f(this.__value));
}
```
一个例子:
```js
let showWelcome = _.compose(_.concat( "Welcome "), _.prop('name'))

let checkActive = function(user) {
 return user.active ? Right.of(user) : Left.of('Your account is not active')
}

let ex6 = _.compose(fmap(showWelcome),checkActive)

console.log(ex6({active:false,name:"luoxia"})) // Left {__value: 'Your account is not active'}
```

### Either Functor

Either可以用于纯的错误处理，这里的纯可以对比一般的`try/catch`。

haskell中关于Either的Functor instance:
```haskell
instance Functor (Either a) where
    fmap f (Right x) = Right (f x)
    fmap f (Left x) = Left x
```
可以看到，Left构造子构造的类型数据可以充当报告错误信息的功能:
```
Prelude> fmap (\s -> "the result: " ++ s) $ Left "error:overflow"
Left "error:overflow"
Prelude> fmap (\s -> "the result: " ++ show s) $ Right 66
Right "the result: 66"
```
来看看js中的例子:
```js
var moment = require('moment');

//  getAge :: Date -> User -> Either(String, Number)
var getAge = curry(function(now, user) {
  var birthdate = moment(user.birthdate, 'YYYY-MM-DD');
  if(!birthdate.isValid()) return Left.of("Birth date could not be parsed");
  return Right.of(now.diff(birthdate, 'years'));
});

getAge(moment(), {birthdate: '2005-12-12'});
// Right(9)

getAge(moment(), {birthdate: 'balloons!'});
// Left("Birth date could not be parsed")
```

### IO Functor
I/O同样也是`functor`：
```haskell
instance  Functor IO where
   fmap f x = x >>= (pure . f)
```
这里还得联系`Applicative Functor`和`Monad`:
```haskell
instance Applicative IO where
    pure  = returnIO
    (*>)  = thenIO
    (<*>) = ap
    liftA2 = liftM2

-- | @since 2.01
instance  Monad IO  where
    (>>)      = (*>)
    (>>=)     = bindIO
    fail s    = failIO s

returnIO :: a -> IO a
bindIO :: IO a -> (a -> IO b) -> IO b

```
而这里的`pure`实际上是对某常规值进行一个`IO`包装，例如
```haskell
fmap (\() -> "hello  world") (putStrLn "hello") :: IO [Char]
```
`putStrLn "hello"`返回值为`IO ()`，经过fmap的f参数一处理返回`[Char]`，再经过`pure`函数包装。

其与下面这种异曲同工:
```haskell
instance Functor IO where
    fmap f action = do
        result <- action
        return (f result)
```

我们再来看看js中实现:
```js
var IO = function(f) {
  this.__value = f;
}

IO.of = function(x) {
  return new IO(function() {
    return x;
  });
}

IO.prototype.fmap = function(f) {
  return new IO(_.compose(f, this.__value));
}
```
我们可以看到IO context中的值是一个函数，而这个函数可以对一些非纯的操作进行包装延迟执行，从而使得程序看起来还是纯的。

举个例子:
```js
//  io_window_ :: IO Window
var io_window = new IO(function(){ return window; });

io_window.fmap(function(win){ return win.innerWidth });
// IO(1430)

io_window.fmap(_.prop('location')).fmap(_.prop('href')).fmap(_.split('/'));
// IO(["http:", "", "localhost:8000", "blog", "posts"])
```
上面的结果注释实际上是概念性的，实际上的返回值是诸如:
```js
IO {
    __value: _.compose(_.split('/'),_.prop('href'),_.prop('location'),function(){ return window; })
}
```
每一次调用`fmap`实际上是把新的function加入到function compose队列最后。这样我们把一系列的不纯的操作包裹在了这样一个IO Functor context里。

就如同haskell中的做法一样，我们在集中的一个地方去释放这些不纯的操作：

```js
////// 非纯调用代码: main.js ///////

// 调用 __value() 来运行它！
IO.of(impureOperation.__value());
```

haskell也是将不纯操作放到`main do`块里，在`runhaskell`时，就释放里面的不纯操作并执行:
```haskell
main = do line <- fmap (intersperse '-' . reverse . map toUpper) getLine  
          putStrLn line
```

### (->) r Functor
刚刚在IO Functor的js实现中我们谈到了function composition,这可以联想到函数本身也是一个在`(->) r` functor context下的值，其fmap实现就等同于function composition。

看看instance定义:
```haskell
instance Functor ((->) r) where
    fmap = (.)
```
而`(.) :: (b -> c) -> (a -> b) -> a -> c`，而根据`fmap`定义:`fmap :: Functor f => (a -> b) -> f a -> f b`，用`(->) r`替换f，得到:`fmap :: (a -> b) -> (r -> a) -> (r -> b)`恰好符合`.`的定义。即一个function composition。

```haskell
ghci > fmap (show . (*3)) (*100) 2
600
```

### Functor laws

1. `fmap id = id`即用id去map over functor会返回跟原functor一样的functor。

2. `fmap (f . g) = fmap f . fmap g`

或者js实现表示：
```js
// identity
map(id) === id;

// composition
compose(map(f), map(g)) === map(compose(f, g));
```

## Applicative Functor type class

之前讲到的Functor，我们只能对functor运用一个单参数函数。但是如果我们要将一个接受多个参数的函数运用到functor呢？这就要提到Applicative Functor,它可以让一个接受多个参数的函数运用到多个Functor上。

首先是其定义:
```haskell
class Functor f => Applicative (f :: * -> *) where
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
  (*>) :: f a -> f b -> f b
  (<*) :: f a -> f b -> f a
```
来看看一个js关于`<*>`的实现：
```js
Container.prototype.ap = function(other_container) {
  return other_container.fmap(this.__value);
}
```

### Maybe Applicative

```haskell
instance Applicative Maybe where
    pure = Just

    Just f  <*> m       = fmap f m
    Nothing <*> _m      = Nothing
```

```
ghci> pure (+) <*> Just 3 <*> Just 5  
Just 8  
ghci> pure (+) <*> Just 3 <*> Nothing  
Nothing  
ghci> pure (+) <*> Nothing <*> Just 5  
Nothing
```
这让我们能够以一种从左到右的方式去应用函数到多个Functors:
```js
Maybe.of(_.add).ap(Maybe.of(2)).ap(Maybe.of(3));
```

而`pure (+) <*> Just 3`这种完全可以用`fmap (+) Just 3`代替，有一个`<$>`就是做这个的:
```haskell
(<$>) :: (Functor f) => (a -> b) -> f a -> f b  
f <$> x = fmap f x
```

这样就可以这样写:`(++) <$> Just "johntra" <*> Just "volta"`就可以直接将一个普通函数用到applicative functor上了。

从js角度来看：
```js
F.of(x).fmap(f) == F.of(f).ap(F.of(x))
```
那么上面的例子同意可以转换为：
```
Maybe.of(2).fmap(_.add).ap(Maybe.of(3));
```

### List Applicative
```haskell
instance Applicative [] where
    pure x    = [x]
    fs <*> xs = [f x | f <- fs, x <- xs]
```
可以看到`<*>`是将左右两个`[]`进行任意组合，左边的包含了多个函数，右边包含了多个原始值。
```haskell
ghci> (++) <$> ["ha","heh","hmm"] <*> ["?","!","."]  
["ha?","ha!","ha.","heh?","heh!","heh.","hmm?","hmm!","hmm."]
```

来看看ramdajs关于List的`ap`实现：
```js
R.ap([R.multiply(2), R.add(3)], [1,2,3]); //=> [2, 4, 6, 4, 5, 6]
```
显然它们是异曲同工。

### IO Applicative
首先是定义:
```haskell
instance Applicative IO where
    pure  = returnIO
    (<*>) = ap
ap                :: (Monad m) => m (a -> b) -> m a -> m b
ap m1 m2          = do { x1 <- m1; x2 <- m2; return (x1 x2) }
```
终于看到了`ap`，也就是`<*>`的别名。换种写法:
```haskell
instance Applicative IO where  
    pure = return  
    a <*> b = do  
        f <- a  
        x <- b  
        return (f x)
```
即`(<*>) :: IO (a -> b) -> IO a -> IO b`。

以`(++) <$> getLine <*> getLine`为例，`getLine :: IO String`，`fmap (++) getLine`结果就是`fmap (++) getLine :: IO ([Char] -> [Char])`，即从第一个`getLine`里取出结果`str1`，map over`(++)`得到`IO (++str1)`，来到`<*>`，将`(++str1)`取出应用到第二个参数的IO上，最后通过`return`封装成`IO ([Char])`。

来看一个js中的例子：
```js
// 帮助函数：
// ==============
//  $ :: String -> IO DOM
var $ = function(selector) {
  return new IO(function(){ return document.querySelector(selector) });
}

//  getVal :: String -> IO String
var getVal = _.compose(map(_.prop('value')), $);

// Example:
// ===============
//  signIn :: String -> String -> Bool -> User
var signIn = _.curry(function(username, password, remember_me){ /* signing in */  })

IO.of(signIn).ap(getVal('#email')).ap(getVal('#password')).ap(IO.of(false));
// IO({id: 3, email: "gg@allin.com"})
```

按照上一节关于IO fmap实现，实际最后的返回结果：
```js
IO {
    unsafePerformIO: _.compose(remember_me=>{},()=>false)
}
```

### (->) r　Applicative
```haskell
instance Applicative ((->) a) where
    pure = const
    (<*>) f g x = f x (g x)　-- f <*> g = \x -> f x (g x)
```
`(+) <$> (+3) <*> (*100) $ 5 `结果为`508`，`(+) <$> (+3)`相当于`(+)`和`(+3)`组合，而这两个组合的结果`(+) <$> (+3) :: Num a => a -> a -> a`，返回一个接受两个参数的函数，第一个参数用于(+3)，结果再和第二个参数一起调用(+)。

```
ghci> (\x y z -> [x,y,z]) <$> (+3) <*> (*2) <*> (/2) $ 5  
[8.0,10.0,2.5]
```

### lift
```haskell
liftA2 :: (Applicative f) => (a -> b -> c) -> f a -> f b -> f c  
liftA2 f a b = f <$> a <*> b
```
同理，在js中实现:
```js
var liftA2 = curry(function(f, functor1, functor2) {
  return functor1.map(f).ap(functor2);
});

var liftA3 = curry(function(f, functor1, functor2, functor3) {
  return functor1.map(f).ap(functor2).ap(functor3);
});
```
或者看看ramda关于lift实现:
```js
var madd3 = R.lift((a, b, c) => a + b + c);

madd3([1,2,3], [1,2,3], [1]); //=> [3, 4, 5, 4, 5, 6, 5, 6, 7]
```
这样程序写法上就更加通用，因为没有体现具体的functor:
```js
liftA2(add, Maybe.of(2), Maybe.of(3));
// Maybe(5)

liftA2(renderPage, Http.get('/destinations'), Http.get('/events'))
// Task("<div>some page with dest and events</div>")

liftA3(signIn, getVal('#email'), getVal('#password'), IO.of(false));
// IO({id: 3, email: "gg@allin.com"})
```

### 与Functor,Monad转换
```haskell
-- Applicative实现fmap:
fmap ("hello "++) (Just "luoxia")
pure ("hello "++) <*> Just "luoxia"

-- Monad实现fmap:
fmap ("hello "++) (Just "luoxia")
Just "luoxia" >>= (\name -> Just ("hello " ++ name))

-- Monad实现 <*>:
pure ("hello "++) <*> Just "luoxia"
pure ("hello "++) >>= (\f -> fmap f (Just "luoxia"))
```


js角度：
```js
// 从 of/ap 衍生出的 fmap
X.prototype.fmap = function(f) {
  return this.constructor.of(f).ap(this);
}
// 从 chain 衍生出的 map
X.prototype.fmap = function(f) {
  var m = this;
  return m.chain(function(a) {
    return m.constructor.of(f(a));
  });
}

// 从 chain/map 衍生出的 ap
X.prototype.ap = function(other) {
  return this.chain(function(f) {
    return other.fmap(f);
  });
};
```

## Monoids

幺半群是一个存在单位元（幺元）的半群。满足结合律，有单位元。一个 monoid 是你有一个遵守结合律的二元函数还有一个可以相对于那个函数作为 identity 的值。

haskell中关于Monoids class的定义:
```haskell
class Monoid a where
  mempty :: a          -- 表示一个特定 monoid 的 identity
  mappend :: a -> a -> a    -- 接受两个 monoid 的值并回传另外一个
  mconcat :: [a] -> a
```
这样就可以得出monoid的laws:

1. mempty `mappend` x = x
2. x `mappend` mempty = x
3. (x `mappend` y) `mappend` z = x `mappend` (y `mappend` z)

### Monoids instance
List :

```haskell
instance Monoid [a] where
        mempty  = []
        mappend = (++)
        mconcat xss = [x | xs <- xss, x <- xs]
```

Product and Sum:

由于数值类型的数据可以有不同的monoid实现，如`*做二元函数，1作幺元`或者`+做二元函数,0作幺元`。

要通过两种方式实现，就如同用随机组合和链式两种方式实现List的`Applicative instance`定义一样，可以用`newtype`方式实现。这方面我的另外一篇关于`Haskell类型系统`的总结有说明。

以`Product`为例:
```haskell
newtype Product a =  Product { getProduct :: a }  
    deriving (Eq, Ord, Read, Show, Bounded)

instance Num a => Monoid (Product a) where  
    mempty = Product 1  
    Product x `mappend` Product y = Product (x * y)
```
即`getProduct $ Product 3 `mappend` Product 9  `得　27.

Any and ALL用于定义Bool数据的两种不同monoid实现。

The Ordering monoid:

```haskell
instance Monoid Ordering where
        mempty         = EQ
        LT `mappend` _ = LT
        EQ `mappend` y = y
        GT `mappend` _ = GT
```
这个可以运用到对于两个数据进行多种方式比较，但不同方式之间有先后顺序时，如:
```haskell
lengthCompare :: String -> String -> Ordering  
lengthCompare x y = (length x `compare` length y) `mappend`  
                    (vowels x `compare` vowels y) `mappend`  
                    (x `compare` y)  
    where vowels = length . filter (`elem` "aeiou")
```

Maybe the monoid:
```haskell
instance Monoid a => Monoid (Maybe a) where
  mempty = Nothing
  Nothing `mappend` m = m
  m `mappend` Nothing = m
  Just m1 `mappend` Just m2 = Just (m1 `mappend` m2)
```

## Monad type class
前面讲了Functor和Applicative Functor，接下来我们要讲讲Monad，这玩意儿着实不好理解。这里仅作为个人学习总结，不代表能够作为学习参考用。

之前的Functor是: `(a -> b) -> f a -> f b`，而Applicative是:`f (a -> b) -> f a -> f b`，然后Monad要求是：`(Monad m) => m a -> (a -> m b) -> m b`


先来看看typeclass定义:
```haskell
class Applicative m => Monad m where
    (>>=)       :: m a -> (a -> m b) -> m b

    (>>)        :: m a -> m b -> m b
    m >> k = m >>= \_ -> k

    return      :: a -> m a
    return      = pure

    fail        :: String -> m a
    fail s      = errorWithoutStackTrace s
```
我们可以联想到fmap实现，它们的区别是，fmap的函数会返回普通值，而`>>=`的函数会返回经过Monad context包装的值，如果考虑到直接将函数通过fmap运用到monad,函数的返回值还会再在外面包装一层Monad context,这样就成了两层context:
```
Prelude> fmap (\s -> Just (s++",haha")) (Just "luoxia")
Just (Just "luoxia,haha")
```
js例子：
```js
//  safeProp :: Key -> {Key: a} -> Maybe a
var safeProp = curry(function(x, obj) {
  return new Maybe(obj[x]);
});

//  safeHead :: [a] -> Maybe a
var safeHead = safeProp(0);

//  firstAddressStreet :: User -> Maybe (Maybe (Maybe Street) )
var firstAddressStreet = compose(
  map(map(safeProp('street'))), map(safeHead), safeProp('addresses')
);

firstAddressStreet(
  {addresses: [{street: {name: 'Mulburry', number: 8402}, postcode: "WC2N" }]}
);
// Maybe(Maybe(Maybe({name: 'Mulburry', number: 8402})))
```
可以看到，`firstAddressStreet`组合函数的最后一个map调用为:`map(map(safeProp('street')))`，因为经历过前两个函数调用，实际的结果为：`Maybe(Maybe({street: {name: 'Mulburry', number: 8402}, postcode: "WC2N" }))`，用得到的对象被包了两层context,对其使用map，就得剥开两层。

这就好比洋葱，一层一层剥开你的心，会悲伤，会流泪。

而`>>=`的作用就是替我们剥开那多的一层外衣：
```js
Maybe.prototype.join = function() {
  return this.isNothing() ? Maybe.of(null) : this.__value;
}

//  join :: Monad m => m (m a) -> m a
var join = function(mma){ return mma.join(); }

//  firstAddressStreet :: User -> Maybe Street
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

firstAddressStreet(
  {addresses: [{street: {name: 'Mulburry', number: 8402}, postcode: "WC2N" }]}
);
// Maybe({name: 'Mulburry', number: 8402})
```
显然这还不够，要是我们能够将剥开多余外衣的操作和map操作合并就好了，那样就是我们的`>>=`真正要实现的:
```js
//  chain :: Monad m => (a -> m b) -> m a -> m b
var chain = curry(function(f, m){
  return m.map(f).join(); // 或者 compose(join, map(f))(m)
});

// map/join
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

// chain
var firstAddressStreet = compose(
  chain(safeProp('street')), chain(safeHead), safeProp('addresses')
);
```

或者换一种写法：
```js
let result = Maybe.of(3).chain(three => 
    Maybe.of(2).fmap(_.add(three))
).chain(sum => 
    Maybe.of(_.append(sum,[7,6]))
)
```
咋这么眼熟呢？对，跟Promise贼像。这就是链式调用。

### Maybe Monad
```haskell
instance  Monad Maybe  where
    (Just x) >>= k      = k x
    Nothing  >>= _      = Nothing

    (>>) = (*>)

    fail _              = Nothing
```
例子:
```haskell
landLeft :: Birds -> Pole -> Maybe Pole  
landLeft n (left,right)  
    | abs ((left + n) - right) < 4 = Just (left + n, right)  
    | otherwise                    = Nothing  

landRight :: Birds -> Pole -> Maybe Pole  
landRight n (left,right)  
    | abs (left - (right + n)) < 4 = Just (left, right + n)  
    | otherwise                    = Nothing

return (0,0) >>= landLeft 1 >> Nothing >>= landRight 1  --Nothing
```
这里的`>>`对于Maybe是在`Applicative instance`定义的：
```haskell
Just _m1 *> m2      = m2
Nothing  *> _m2     = Nothing
```
换成`do`写法:
```haskell
routine :: Maybe Pole  
routine = do  
    start <- return (0,0)  
    first <- landLeft １ start  
    Nothing  
    second <- landRight １ first  
    landLeft 1 second
```

### List Monad
```haskell
instance Monad []  where
    xs >>= f             = [y | x <- xs, y <- f x]
    (>>) = (*>)
    fail _              = []
```
例子：
```
ghci> [1,2] >>= \n -> ['a','b'] >>= \ch -> return (n,ch)  
[(1,'a'),(1,'b'),(2,'a'),(2,'b')]
```
换种写法:
```haskell
listOfTuples :: [(Int,Char)]  
listOfTuples = do  
    n <- [1,2]  
    ch <- ['a','b']  
    return (n,ch)
```
![https://learnyoua.haskell.sg/content/zh-cn/ch12/concatmap.png](https://learnyoua.haskell.sg/content/zh-cn/ch12/concatmap.png)

实际上`[ (n,ch) | n <- [1,2], ch <- ['a','b'] ]`就是上面写法的一种语法糖。

再来个例子:
```haskell
guard :: (MonadPlus m) => Bool -> m ()  
guard True = return ()  
guard False = mzero

[1..50] >>= (\x -> guard ('7' `elem` show x) >> return x) 　--[7,17,27,37,47]
```
这里引用`Control.Monad`中的`guard`函数，传递一个Bool参数，回传一个`MonadPlus instance`包装的数据，这里List在`guard True`下返回`[()]`，否则返回`[]`。

而对于List的`>>`在`Applicative instance`定义:

```haskell
xs *> ys  = [y | _ <- xs, y <- ys]
```
当`[()] >> return x`的时候，返回`[x]`，当`[] >> return x`的时候，返回`[]`，这个不难理解。这样就起到了filter的作用。

也可以换成下面这种写法:
```haskell
sevensOnly :: [Int]  
sevensOnly = do  
    x <- [1..50]  
    guard ('7' `elem` show x)  
    return x
```
可见guard这句不进行`<-`，就相当于在其后用`>>`

**未完待续...**