---
layout: post
title:  "Python函数式编程系列012：惰性列表之生成器与迭代器"
date:   2021-10-25 14:16:30 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "函数式编程", "类型", "面向对象"]
---

因为本系列还是基于一些已经对`Python`有一定熟悉度的读者，所以我们在此不做非常多的赘述来介绍基本知识了。而是回我们之前的主题，我们要用迭代器和生成器实现之前的指数函数。

当然，我们这里还是需要回到惰性列表是什么这个问题。事实上，回到原来惰性求值的概念，惰性列表的概念其实是「需要时才计算出值」的列表。我们在调用`iter`的时候，其实对常见的对象并没有特别大的优势。我们可以假想，其实`iter`转化`[1, 2, 3, 4]`的结果其实如下：

```python
def yield_list():
    yield 1
    yield 2
    yield 3
    yield 4
```

唯一的优势，我们之前已经提到过了，就是反复套用函数`f`和`g`时，我们是计算`g(f(x))`而不是先把列表里每个值套用`f`再套用`g`。这里有个极大的优势，就是提前终止时可以避免没有必要的运算。比如，下面一个`for`里面的例子，我们是为了发现列表`ls`中应用`f`函数后如果结果等于`a`就返回index否则返回`None`：

```python
def find_index_apply_f(f, ls, a):
    for i, x in enumerate(ls):
        if f(x) == a:
            return i
        else:
            continue
    return None

>>> find_index_apply_f(lambda x: x + 1, [1, 2, 3, 4, 5], 3)
1
```

现在，这里提前跳出可以减少非常多的运算量，但是如果使用一个普通列表却很难，我们在使用`map`之后必然已经全都计算了，但如果惰性求值，我们可以就在需要的时候停止就行。这个是列表操作替代循环必须实现的东西。

第二个惰性列表的最大应用，就是无穷列表，比如下面一个生成器，我们可以生成一个无限长度的全是`x`的列表。后面我们会聊到我们在各种场合中已经用到了这个抽象。

```python
def yield_x_forever(x):
    while True:
        yield x
```

## 实现一些常用的(惰性)列表操作

大部分操作迭代器/生成器的函数，我们都可以在`itertoools`中找到。但，我们这里还是要实现一些非常**函数式**的函数，方便以后的操作：

### 1. head

`head`很简单，即取出(惰性)列表第一个元素：

```python
head = next
```

### 2. take

`take`的目标是列表前N个值，这个可以实现成触发计算（转化成非惰性对象，一般为一个值或者列表）或者不触发计算的版本。下面我们实现的是触发计算的函数。

```python
def take(n, it):
    """将前n个元素固定转为列表
    """
    return [x for x in islice(it, n)]

take_curry = lambda n: lambda it: take(n, it)
```

### 3. drop

`drop`则相反是删去前N个值。

```python
def drop(n, it):
    """剔除前n个元素
    """
    return islice(it, n, None)
```

### 4. tail

`tail`是删去`head`后的列表，可以用`drop`实现：

```python
from functools import partial

tail = partial(drop, 1)
```

### 5. iterate

`iterate`是重点要用到的函数，就是通过一个迭代函数还有初始值，实现一个无穷列表：

```
def iterate(f, x):
    yield x
    yield from iterate(f, f(x))
```

比如，实现所有正偶数的无穷列表：

```python
positive_even_number = iterate(lambda x: x + 2, 2)
```

当然，更简单地写法是使用`itertools`里面的`repeat`和`accumulate`：

```python
def iterate(f, x):
    return accumulate(repeat(x), lambda fx, _: f(fx))
```

## 简单实践

### 例子一：求指数

我们回到之前求指数的例子中，我们可以实现惰性列表的版本。

第一个思路，我们就是直接用`iterate`从`x`开始，每次乘以`x`，然后取出前`n`个值，拿到最后一个：

```python
power = lambda x, n: take(n, iterate(lambda xx: xx * x, x))[-1]
```

另一个就是先生成一个无穷长度的`x`，取出前`n`个，相乘来`reduce`：

```python
power = lambda x, n: reduce(
    lambda x, y: x * y, 
    take(n, iterate(lambda _: x, x))
)
```

当然，我们还可以用**生成器**生成无穷长列表：

```python
def yield_power(x, init=x):
    yield init
    yield from yield_power(x, init * x)
```

### 例子二：查找

我们回到上面解说的例子，我们要找到一个无穷列表中套用`f`后，第一个等于`a`的值的`index`。如果不是惰性的话，这个必须提前跳出也不可能实现。

```python
def find_a_in_lazylist(f, lls, a):
    return head(filter(lambda x: f(x[1]) == a, enumerat(lls)))[0]
```

## 总结

本章回顾了利用`Python`自带的生成器、迭代器实现惰性列表，并展示如何运用这些概念做一些数据操作应用。当然在其中，我们要深刻感受到，函数式编程与数据是非常亲近的，它关注数据胜于项目结构，这点和对象式编程非常不同。大部分对象式编程的教程倾向于概述分层、结构这些概念，真是因为这个是对象式编程擅长的地方。

在我实现的教学项目[`fppy`(点击这里前往`github`)](https://github.com/threecifanggen/python-functional-programming)中，我用内置的`python`模块实现了一个`LazyList`类，用它可以用链式写法完成上面的所有例子:

```python
power1 = lambda x, n: LazyList.from_iter(x)(lambda xx: x * x).take(n).last
power2 = lambda x, n: LazyList.from_iter(x)(lambda _: x).take(n).reduce(lambda xx, yy: xx * yy)

find_a_in_lazylist = lambda f, lls, a: LazyList(lls)\
    .zip_with(LazyList.from_iter(0)(lambda x: x + 1))\
    .filter(lambda x: f(x[1]) == a)\
    .split_head()[0]
```

{% include comments.html %}
