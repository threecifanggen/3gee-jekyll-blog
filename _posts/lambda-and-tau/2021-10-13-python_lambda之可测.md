---
layout: post
title:  "Python函数式编程系列008：可测"
date:   2021-10-13 10:59:12 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "函数式编程", "测试"]
---

我们在之前的文章之中，已经反复地强调了很多函数式编程的优点，例如表达能力，延迟计算的好处之类的。但其实一个更大的有点其实是可测性。本篇文章也是传达整个系列要表达的核心，我们不是要完全排除过程式、副作用等概念，而是有限的使用，并且能在现有代码的基础上做改良。

## 缘起

下面，我们看一个例子：一个公司希望设计一个基于时间的调度器，它们可以提供一个比`crontab`更完善的语法，比如可以基于每个月前三天、每周周末、每个月第二周的第一天之类这些表述。设计这个调度器时候，就会涉及到很多有关时间的函数，比如，下面是一个可能要实现的函数：

```python
from datetime import datetime, timedelta

def yesterday_str() -> str:
    """获取昨日的时间的字符串(YYYYMMDD)
    """
    return (
        datetime.now() - timedelta(days=1)
        ).strftime("%Y%m%d")
```

这是一个最直观的实现，但是，这个函数我们发现是不可测的。原因大家应该看出来，就是因为`datetime.now()`是带有副作用的。具体我们可以把测试中可能遇到的问题例举如下：

### 单元测试例子中的问题

我们会如何写这个函数的单元测试呢，很显然，大部分人会这么写：

```python
def test_yesterday_str():
    assert yesterday_str() == (
        datetime.now() - timedelta(days=1)
    ).strftime("%Y%m%d")
```

很显然，这个单元测试一眼就看出了几个问题：

1. 事实上，我们只是重新写了一遍原有的代码，并没有真的测试。
2. 即使我们承认这种写法，也有一定概率在接近凌晨的时候（23:59:59秒时），这个测试不通过，但这又不是因为功能实现的问题导致的错误。

### 整合测试中的问题

在实际测试中，可能某些整合的部分更难测试到，比如我们下面一个调用上面函数的函数，它的功能是在每个月1号执行一个任务：

```python
def run_at_first_day():
    if yesterday_str()[-2:] == '01':
        do_something()
```

这个例子不仅把副作用又一步步传递下去了，而且在测试中，我们如果不是在1号进行测试，我们就只能测到`do_something`的逻辑而测不到`run_at_first_day`这个调度的逻辑。而可以想象，在这个系统内，这种例子会非常多。

## 如何解决

### 常规解决方案

常规的解决方案，第一个就是修改系统时间。`Python`中有一个`FreezeGun`的模块，就是做类似事的：

```python
from freezegun import freeze_time
import datetime

@freeze_time("2012-01-14")
def test():
    assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)
```

当然，这个解决方案是针对**时间**这个事的，我们遇到的副作用可能不止这一种，可能是读取配置、数据库交互等等，这种方案无法解决这些事。

另一类就是测试领域的概念，比如`fake`、`mock`、`stub`之类的概念了，我们在下面的工作中当然也会用到`fake`的概念，但是不需要纠结于这些复杂的概念。

### 把副作用函数作为参数

我们改写成下面的方式，就发现整个函数变得可测了：

```python
def yesterday_str(now_func = date.now) -> str:
    assert yesterday_str() == (
        now_func() - timedelta(days=1)
    ).strftime("%Y%m%d")
```

具体的测试写法如下：

```python
def fake_now(now_str):
    def helper():
        return datetime.strptime(now_str, "%Y-%m-%d")
    return helper

def test_yesterday_str():
    return yesterday_str(fake_now('2020-01-01')) == '2019-12-31'
```

我们发现这么写有诸多好处了：

1. 整个函数变成了无副作用了，副作用被隔离在了参数里面
2. 因为无副作用了，我们只需要自己制作相应的「假」函数就可以模拟要的输入了，特别是针对`Void -> A`这种类型的函数。
3. 我们可以通过假函数的方式模拟任何一个状态时的操作，这使得我们上面说的调度逻辑可以变得可以测试了。


