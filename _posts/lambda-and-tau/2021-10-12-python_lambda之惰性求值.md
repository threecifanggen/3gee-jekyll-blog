---
layout: post
title:  "Python函数式编程系列007：惰性求值"
date:   2021-10-12 09:35:10 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "函数式编程", "惰性求值"]
---

> 本系列文章一些重要的函数、方法、类我都实现的一遍，你可以在[github(点击此处)](https://github.com/threecifanggen/python-functional-programming)中找到代码和测试例子（如果网速过慢我也放了一份在[gitee(点击此处)](https://gitee.com/sancifanggen/fppy)上，但请勿在gitee上提`issue`或者留言），欢迎`star`/`fork`。

## 缘起

我们回到介绍高阶函数的一章，我们提到了高阶函数特别是科里化的一个好处便是「提前求值」和「推迟求值」，通过这些操作，我们可以大大优化很多代码。比如，我们使用之前的例子：

```python
def f(x): # x储存了某种我们需要的状态
    ## 所有可以提前计算的放在这里
    z = x ** 2 + x + 1
    print('z is {}'.format(z))
    def helper(y):
        ## 所有延迟计算的放在这里
        return y * z
    return helper
```

我们在调用`f(1)`的时候，其实就已经事先计算了`z`的部分，如果我们临时保存这个值，反复调用时就可以节省很大的时间：

```python
>>> g = f(1)
z is 3
>>> g(2) + g(1) # 可以看到这次就不会打印`z is xxxx`的输出了
9
```

也就是说适时的「提前求值」和「推迟求值」都可以帮助我们大大地减少很多运算开销。这就引入我们这一篇要讲的「惰性求值」的概念，惰性求值的概念主要是：调用时才计算，并且只计算一次。

## 惰性属性与惰性值

我们考虑下面一个例子：

定义一个圆的类，通过圆心和半径来描述，但是当我们知道圆心和半径之后我们能知道很多事，比如：

1. 周长(`perimeter`)
2. 面积(`area`)
3. 圆最上面坐标的位置(`upper_point`)
4. 圆心到原点的距离(`distance_from_origin`)
5. ...

这个列表可能非常非常多，而且随着软件功能的增加，这个列表可能还会添加。我们可能有两种方法实现。第一种就是在初始化的时候都给设定为圆的属性：

```python
@dataclass
class CircleInitial:
    x: float
    y: float
    r: float

    def __init__(self, x, y, r):
        self.x = x
        self.y = y
        self.r = r

        self.perimeter = 2 * r
        self.area = r * r * 3.14
        self.upper_point = (x, y + r)
        self.lower_point = (x, y - r)
        self.left_point = (x - r, y)
        self.right_point = (x + r, y)
        self.distance_from_origin = (x ** 2 + y ** 2) ** (1/2)
```

我们马上可以看出问题：如果这样的属性非常多，而且涉及的计算也非常多的话，那么当我们实例化一个新的对象的时候，耗费的时间将会非常长。然而，大部分的属性，我们可能都不会用到。

于是，就有了第二个方案，把这些实现成一个方法（我们这里仅举例一个`area`方法）：

```python
@dataclass
class CircleMethod:
    x: float
    y: float
    r: float

    def area(self):
        print("area calculating...")
        return self.r * self.r * 3.14
```

当然，因为这个值是一个「常」量的概念，我们也可以使用`property`修饰器，这样我们就可以不用带括号地调用它了：

```python
@dataclass
class CircleMethod:
    x: float
    y: float
    r: float

    @property
    def area(self):
        print("area calculating...")
        return self.r * self.r * 3.14
```

我故意在其中加入了一行打印代码，我们可以发现，我们每次调用`area`时，都会被计算一次：

```python
>>> a = CircleMethod(1, 2, 3)
>>> a.area ** 2 + a.area + 1
area calculating...
area calculating...
827.8876000000001
```

这又是另外一种浪费了，于是我们发现，第一种方案适合需要经常被反复调用的属性，第二个方案实现很少被调用的属性。但是，可能我们在维护代码的时候，没法事先预判一个属性是不是经常被调用，而且这也不是一个长久之计。但我们发现我们需要的就是那么一个属性：

1. 这个属性不会初始化的时候计算
2. 这个属性只在被调用时计算
3. 这个属性只会计算一次，后面不会调用

这个就是「惰性求值」的概念，我们也把这种属性叫「惰性属性」。`Python`没有内置的惰性属性的概念，不过，我们可以很容易从网上找到一个实现（你也可以在我的[`Python-functional-programming`](https://github.com/threecifanggen/python-functional-programming)中的`lazy_evaluate.py`中找到）：

```python
def lazy_property(func):
    attr_name = "_lazy_" + func.__name__

    @property
    def _lazy_property(self):
        if not hasattr(self, attr_name):
            setattr(self, attr_name, func(self))
        return getattr(self, attr_name)

    return _lazy_property
```

具体的使用，只是切换一下修饰器`property`：

```python
@dataclass
class Circle:
    x: float
    y: float
    r: float

    @lazy_property
    def area(self):
        print("area calculating...")
        return self.r * self.r * 3.14
```

我们采用和上面一样的调用方式，可以发现，`area`只计算了一次（只打印了一次）：

```python
>>> b = Circle(1, 2, 3)
>>> b.area ** 2 + b.area + 1
area calculating...
827.8876000000001
```

同样的理由我们也可以实现一个惰性值的概念，不过因为`python`没有代码块的概念，我们只能用**没有参数**的函数来实现：

```python
class _LazyValue:

    def __setattr__(self, name, value):
        if not callable(value) or value.__code__.co_argcount > 0:
            raise NotVoidFunctionError("value is not a void function")
        super(_LazyValue, self).__setattr__(name, (value, False))      
        
    def __getattribute__(self, name: str):
        try:
            _func, _have_called = super(_LazyValue, self).__getattribute__(name)
            if _have_called:
                return _func
            else:
                res = _func()
                super(_LazyValue, self).__setattr__(name, (res, True))
                return res
        except:
            raise AttributeError(
                "type object 'Lazy' has no attribute '{}'"
                .format(name)
            )

lazy_val = _LazyValue()
```

具体调用方法如下，如果你要设计一个模块而这个变量不在类中，那么就可以很方便地使用它了：

```python
def f():
    print("f compute")
    return 12

>>> lazy_val.a = f
>>> lazy_val.a
f compute
12
>>> lazy_val.a
12
```

## 惰性迭代器/生成器

此外，`Python`内置了一些惰性的结构主要就是迭代器和生成器，我们可以很方便验证它们只计算/保留一次（这里只验证迭代器）：

```python
>>> a = (i for i in range(5))
>>> list(a)
[0, 1, 2, 3, 4]
>>> list(a)
[]
```

我们可以设计下面两个函数：

```python
def f(x):
    print("f")
    return x + 1

def g(x):
    print("g")
    return x + 1
```

然后我们思考下面的结果：

```python
>>> a = (g(i) for i in (f(i) for i in range(5)))
>>> next(a)
```

它可能有两种结果，一个它可能的计算方式是这样的：

```python
>>> temp = [f(i) for i in range(5)]
>>> res = g(temp[0])
```

如果是这种结果，则它会打印出5个`f`然后再打印出`g`

另一种可能性则是：

```python
>>> res = (g(f(i)) for i in range(5))
```

则，这样子便只会打印一个`f`和一个`g`。如果根据惰性求值的定义，`i=1`并没有被真实调用，所以它应该不用求值，所以，如果他符合第二个打印情况，则它就是惰性的对象。事实也就真如此。

当然，这个特性已经非常的Fancy了，但是我们基于此可以联想出的一个非常奇妙的引用，因为在迭代器计算中，我们并不是在生成的时候，就计算出了迭代器中的每个值，因此，我们可以用这个方式存储一个无穷系列。通过上面的方式计算后返回结果。一个最简单的例子是内置模块中的`itertools.repeat`，我们可以生成一个无穷的全为`1`的线性结构：

```python
from itertools import repeat

repeat_1 = repeat(1)
```

这样，我们就可以用上面的列表表达式来做一些计算再通过`next`调用了。

```python
res = (g(i) for i in (i * 3 for i in repeat_1))
next(res)
```

我们也将这些线性结构称为「惰性列表」（这里的`repeat_1`则是一个「无穷惰性列表」的例子），在下面的文章中，我们将详细地用这个方式来完成一些有趣的事情。

{% include comments.html %}
