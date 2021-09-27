---
layout: post
title:  "Python函数式编程系列005：离题之定义自然数"
date:   2021-09-27 10:52:10 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "函数式编程", "递归"]
---

## 前言

在我们已经掌握了那么多建管子的方法之后，我们开始离题，看看我们能用最少的概念做哪些自举产生的事。在这一章中我们讲仅使用字符串`"e"`，函数，`if-else`分支，`=="e"`运算，这四个概念来实现一个自然数的概念（实际中还用到了`bool`值，不过`bool`本身也可以用`"e"`和`f("e")`表示）。

## 皮亚诺公理

我们首先回顾一下，数学如何定义即皮亚诺公理如何定义自然数，事实上，皮亚诺公理定义的是「无限可数集」的概念

> * (1) $$e \in S$$
> * (2) $$(\forall a \in S)(f(a) \in S)$$
> * (3) $$(\forall b \in S)(\forall c \in S)(f(b) = f(c) \rightarrow b = c)$$
> * (4) $$(\forall in S)(f(a) \ne e)$$
> * (5) $$(\forall A \subseteq S)(((e \in A) \land (\forall a \in A)(f(a) \in A)) \rightarrow (A = S))$$

* (1) 表示我们需要一个初始值，来表述我们可以从第一个东西开始数数，在这个符号集里叫$$e$$。对应于自然数的「1」的概念。
* (2) 表示往下数一个数的操作，这个符号集里用$$f$$表述，我们一般也把这个操作叫**后继**。对应自然数中「加一」/「往下数一」的概念
* (3) 确定恒等关系。
* (4) 确定$$e$$不是任何数的后继，保证它是第一个被数的数。
* (5) 归纳法

## 实现

我们仅仅需要下面两行代码就已经实现了自然数的全部定义，我们使用递归表示向下数数，用`"e"`表达了起始值「1」

```python
one = "e" # 1
f = lambda x: lambda : x # 后继
```

当然，这么一个定义，是没有任何意义的，我们还需要实现**判断相等**、**加法**、**乘法**这三个最简单的算法。首先判断全等的方法就是我们将两个函数无限地求值下去，看到最后是不是同时得到`"e"`值，这也是对应了性质(3)：

```python
def equal(x, y) -> bool:
    if x =="e" and y =="e":
        return True
    elif x =="e":
        return False
    elif y =="e":
        return False
    else:
        return equal(x(), y())


not_equal = lambda x, y: not(equal(x, y))
```

注意我在上面的实现中使用了`x =="e"`这种前面带空格而后面不带空格的写法，其实是为了强调，`=="e"`是一个一元运算，我们仅使用到了它，而不需要其他概念。而仔细探究这个算式，我们发现其实它也隐式地用到定义(4)，只要一个不为`"e"`我们就可以确定它们是不相等的。

定义加法其实也是一个非常容易的操作，我们只需要让一个参数计算后继，一个参数求值产生前继的概念：

```python
def add(x, y):
    if y =="e":
        return f(x)
    else:
        return add(f(x), y())
```

最后是乘法的概念，这个我们可以调用`add`来递归实现：

```python
def multiply(x, y):
    if y =="e":
        return x
    else:
        return add(multiply(x, y()), x)
```

其实我们也可以同理获得一个自然数求幂的函数，非常类似上面`multiply`的实现

```python
def power(x, y):
    if y =="e":
        return x
    else:
        return multiply(power(x, y()), x)
```

这样我们可以非常快速地给20以内的数取名字了：

```python
one = "e"
two = f(one)
three = f(two)
four = f(three)
five= f(four)
six = f(five)
seven = f(six)
eight = f(seven)
nine = f(eight)
ten = f(nine)
eleven = f(ten)
twelve = f(eleven)
thirteen = f(twelve)
fourteen = f(thirteen)
fifteen = f(fourteen)
sixteen = f(fifteen)
seventeen = f(sixteen)
eighteen = f(seventeen)
nineteen = f(eighteen)
```

OK，最后我们可以通过`equal`验证我们的算法对不对：

```python
>>> assert equal(add(two, one), three)
>>> assert not_equal(power(two, three), seven)
>>> assert equal(power(two, three), eight)
>>> assert equal(multiply(three, five), fifteen)
```

## 结语

这一篇我们偏题地完成了一个「自然数」的定义，目的是为了展现，函数式编程的魅力在于：

1. 我们可以用非常少的概念（在这个例子中是4个）就可以自举地实现非常多的事情。这个也是早期LISP语言（一种常见的动态函数式语言）会那么在AI领域或者一些对语言内核大小非常敏感的领域的原因。
2. 因为函数式编程中的函数和数学上的函数非常接近，这使得在数学上使用的代数运算，都可以非常方便的实现（当然这一点我们在后面也会一一例举出来）。

{% include comments.html %}
