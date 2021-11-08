---
layout: post
title:  "Python函数式编程系列014：函数性质之优化"
date:   2021-11-08 17:31:30 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "基于性质优化", "函数式编程", "Property-based Testing"]
---

我们离开惰性列表的讨论，重新看看，单就函数而言，如果我们承诺，它仅仅代表符号代换，或者无副作用，或者同一输入只会有同一输出，我们还能做哪些过程式无法做到的事。

## 基于符号的优化

第一件我们要谈论的是关于符号的优化的问题，我们这里不是指数值分析中的优化，而是指一些在写法上和速度上都会让代码更佳的方法。既然是完全符合符号代换原则，即我们可以通过推导化简很多复杂的表达式。这个也就是基于符号优化的思路。

一个最简单的例子就是我们可以发现`map`符合下面的结论：

$$
map(f)·map(g) = map(f·g)
$$

这个也可以表达列表和惰性列表计算之间的差异，是先整体对`Iterable`对象操作完`f`再操作完`g`；还是，我们先结合`f.g`再套用到`Iterable`。这两个事情本质上是一样的。当然，如果用我作为教程目的的包来表达这个就是(参考[这个`github`包](https://github.com/threecifanggen/python-functional-programming))：

```python
assert LazyList(xs).map(f).map(g) == LazyList(xs).map(and_then(f, g)) 
```

当然，这个例子中，代码量没有显著减少，并且速度上优化可能也不明显。下面一个例子我们会非常好用：

$$
sum(map\; (\lambda x. f(x) + 1)\; xs)) = sum(map \; f \; xs) + len(xs)
$$

在这个例子里，如果我们对xs套用一个函数$f(x) + 1$最后求和，其结果其实就等于套用$f(x)$后加上`xs`的长度。这样我们就简化了`+1`操作`len(xs) - 1`次了，效率提高很多。甚至我们还能推广：

$$
sum(map\; (\lambda x. f(x) + N)\; xs)) = sum(map \; f \; xs) + len(xs)·N
$$

写成代码就是：

```python
assert LazyList(xs).map(lambda x: f(x) + N).foldleft(lambda x, y: x + y, 0) == \
    LazyList(xs).map(f).foldleft(lambda x, y: x + y , 0) + LazyList(xs).len() * N
```

这样的例子或许看起来过于数学，在工作中，真的做这种四则运算的可能性很多。不过，这里就提醒我们，我们其实针对的可能是「算术」/「运算」的抽象。这就会牵扯到后面说「抽象代数」这一数学分支要研究的内容，即算术本身的特性。比如，结合律：

$$
x \oplus y \oplus z = x \oplus (y \oplus z)
$$

就是一个非常常见的「算术过程」。这个$\oplus$可以是$+$可以是$\times$，甚至可以是字符串合并、列表排序。我们后面会单独抽象这种特征的运算（也叫「半群」），并且思考我们基于这些抽象过程，是不是可以速度上的优化。甚至从某种程度上说，`map`本身也是有结合律特征的。

在下篇文章中，我们将会基于函数的性质(property)来看看一个函数式特有的测试方式——**基于性质的测试**(*Property-based Testing*)

{% include comments.html %}

