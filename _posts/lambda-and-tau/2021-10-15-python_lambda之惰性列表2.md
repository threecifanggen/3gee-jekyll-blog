---
layout: post
title:  "Python函数式编程系列010：惰性列表之动手实现List"
date:   2021-10-15 16:40:30 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "函数式编程", "测试"]
---

这篇文章，我们要动手实现一个`List`，不过和一般的文章不同，我们这里不用类来实现，而是用基本的数据结构，二元元组`(a, b)`和空元组`()`来实现。这两个都可以通过`lambda`直接定义出来，具体方法可以参考上一篇的内容。

我们考虑一下，`List`（也叫链表），最关键的是创建一个模式，可以无穷展开自己，保存一个值和下一个数据的，例如`[1, 2, 3, 4]`我们可以用`(1, (2, (3, (4, ()))))`。我们必须指定一个结尾，这个就是`()`在其中的作用，`()`同时代表空列表和列表结尾的含义。很容易地，我们可以将列表定义如下（我这里包了个函数，只是为了将数据隔离，防止我们使用自带的比较来实现一些功能）：

```python
def cons(head, tail):
    def helper():
        return (head, tail)
    return helper
```

然后我们定义两个函数，来获取里面的数据，类似上一篇接口中的`first`、`second`：

```python
head = lambda cons_list: cons_list()[0]
tail = lambda cons_list: cons_list()[1]
```

我们可以定义一个函数表示空的变量`empty_list_base`，这之后，为了方便计算，我们可以写一个生成`cons`的的方便的方法（当然这个实现用了`*arg`的概念，我们默认使用这个语法糖特性）：

```python
def cons_apply(*args):
    if len(args) == 0:
        return empty_list_base
    else:
        return cons(args[0], cons_apply(*args[1:]))
```

这样我们就可以很方便地完成新建`List`了：

```python
>>> cons_apply(1, 2, 3) # 返回cons(1, cos(2, cos(3, ())))
```

为了方便比较，我们也可以定义一个判断列表是否相等的函数：

```python
def equal_cons(this: ListBase[S], that: ) -> bool:
    if this == empty_list_base and that != empty_list_base:
        return False
    elif this != empty_list_base and that == empty_list_base:
        return False
    elif this == empty_list_base and that == empty_list_base:
        return True
    else:
        return head(this) == head(that) and equal_cons(tail(this), tail(that))
```

现在我们就可以很方便地做一些验证了。

```python
>>> assert equal_cons(cons_apply(1, 2, 3), cons(1, cons(2, cons(3, ()))))
```

现在我们需要就是要实现一些不需要循环实现的列表运算，就是上一篇说的`map`、`fold_left`和`filter`。

`map`的作用是将函数`f`带入到列表的每一个值，即我们带入`f`到列表的头之后，再把`map`应用到`tail`中，即：

```python
def map_cons(f, cons_list):
    if cons_list == ():
        return empty_list_base
    else:
        return cons(f(head(cons_list)), map_cons(f, tail(cons_list)))
```

同理，我们可以实现`filter`和`fold_left`:

```python
def filter_cons(f, cons_list):
    if cons_list == ():
        return empty_list_base
    else:
        hd, tl = head(cons_list), tail(cons_list)
        if f(hd):
            return cons(hd, filter_cons(f, tl))
        else:
            return tl

def fold_left_cons(f, init, cons_list):
    if cons_list == ():
        return init
    else:
        return fold_left_cons(f, f(init, head(cons_list)), tail(cons_list))
```

这样，我们就可以实现一些基本功能了，比如将`[1, 2, 3, 4, 5]`每个元素加一，筛选偶数求和，就可以写成：

```python
>>> res = fold_left_cons(lambda x, y: x + y, 0,
>>>    filter_cons(lambda x: x % 2 == 0, 
>>>        map_cons(lambda x: x + 1,
>>>            cons_apply(1, 2, 3, 4, 5)
>>>    )))
>>> res == 12
```

当然，这种风格的代码，嵌套的可读性很差，这里我们就想到了之前我们实现的`and_then`或`compose`函数，可以组合这些水管构造的东西。不过，我们将这些函数改成科里化会更方便的写。这样就可以用函数组合的风格了：

```python
map_cons_curry = lambda f: lambda cons_list: map_cons(f, cons_list)
filter_cons_curry = lambda f: lambda cons_list: filter_cons(f, cons_list)
fold_left_cons_curry = lambda f: lambda init: lambda cons_list: fold_left_cons(f, init, cons_list)
```

具体的调用就是下面的方法了：

```python
>>> f = and_then(
>>>    map_cons_curry(lambda x: x + 1),
>>>    filter_cons_curry(lambda x: x % 2 == 0),
>>>    fold_left_cons_curry(lambda x, y: x + y)(0),
>>> )
>>>
>>> assert f(cons_apply(1, 2, 3, 4, 5)) == 12
```

如果你使用了我维护的这个[`fppy`](https://github.com/threecifanggen/python-functional-programming)的例子的话，你也可以使用一个`F_`的修饰器轮子，这样就可以实现另一种基于类的链式写法：

```python
from fppy.base import F_, I

F_(I)\
    .and_then(map_cons_curry(lambda x: x + 1))\
    .and_then(filter_cons_curry(lambda x: x % 2 == 0))\
    .and_then(fold_left_cons_curry(lambda x, y: x + y)(0))\
    .apply(cons_apply(1, 2, 3, 4, 5)) # 返回12
```

这篇之中，我们简单仅用二元元组、相等、函数的概念，维护了一个列表的结果，并能通过一些列表函数对齐进行遍历计算、筛选。下一篇之中，我们讲开始粗略地讨论类、类型这些概念，这将方便我们以后的讨论。

{% include comments.html %}
