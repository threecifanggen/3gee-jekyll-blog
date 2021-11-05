---
layout: post
title:  "Python函数式编程系列013：惰性列表之流式处理"
date:   2021-11-05 10:00:30 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "函数式编程", "类型", "面向对象"]
---

如果对于惰性列表更多地理解，就会发现，无限惰性列表的概念和「列队」的概念很相符，或者我们说的「消息队列」。这是一个很好的类似`SparkStreaming`或者`Flink`之类的流式处理的抽象。

所以，我们可以看到许多的和列队有关的模块是以(无限)生成器的方式维护的，而处理这种数据结构就是函数式编程特别擅长的。但函数式特别不适合层级结构，当然高阶函数可以完成这些事，但这依赖过多的人为规范；**类编程会提供一个很好的代码骨架，然后在细部处理数据时调用函数式方法**，这个才是整个系列所要推崇的。

我们可以模拟一个一直产生数据的`faker`，当然，你也可以把它当来自于任何消息队列传来的数据：

```python
from faker import Faker
def data_stream_gen():
    fk = Faker()
    while True:
        yield {
            "create_time": (
                fk
                .date_time_between_dates()
                .strftime("%Y-%m-%d %H:%M:%S")),
            "name": fk.name(),
            "email": fk.email(),
            "age": fk.random_int(14, 50),
            "address": fk.address(),
            "device": {
                "browser": fk.chrome()
            }
        }
```

这样，我们就可以使用我封装好的`fppy`中的`LazyList`来装载这个生成器（当然不是必须的，只是为了方便一种类似**Point Free**/管道的写法，具体的代码仓库[点击此处](https://github.com/threecifanggen/python-functional-programming)），并获取前三个验证一下：

```python
>>> LazyList(data_stream_gen()).take(3).collect()
[{'create_time': '2021-11-05 13:57:30',
  'name': 'Kevin Parrish',
  'email': 'mooremichael@example.net',
  'age': 19,
  'address': '4311 James Divide\nPacehaven, PA 64853',
  'device': {'browser': 'Mozilla/5.0 (Macintosh; U; PPC Mac OS X 10_10_8) AppleWebKit/535.2 (KHTML, like Gecko) Chrome/44.0.879.0 Safari/535.2'}},
 {'create_time': '2021-11-05 13:57:30',
  'name': 'Donald Cochran',
  'email': 'wendyjones@example.net',
  'age': 15,
  'address': '255 Ward Spring Suite 178\nAdamland, MI 99867',
  'device': {'browser': 'Mozilla/5.0 (iPad; CPU iPad OS 12_4_8 like Mac OS X) AppleWebKit/533.2 (KHTML, like Gecko) CriOS/60.0.839.0 Mobile/04P519 Safari/533.2'}},
 {'create_time': '2021-11-05 13:57:30',
  'name': 'Gary Lowe',
  'email': 'udean@example.net',
  'age': 25,
  'address': '60536 Kelly Green\nNorth Brittanyland, DE 80335',
  'device': {'browser': 'Mozilla/5.0 (Windows NT 6.0) AppleWebKit/536.1 (KHTML, like Gecko) Chrome/58.0.882.0 Safari/536.1'}}]
```

我们现在假设要完成以下需求：

1. 将用户相关信息`name`, `email`, `age`, `adress`包裹在一个`user_info`中
2. 将`create_time`改成时间戳格式
3. 删除`device`。

此时，我们只需要单独维护一个个需求，不考虑整个数据流而是只专注在一个数据，每个功能实现成一个函数，这样就完全实现组合式的解耦了，整个系统也变得更加可测:

```python
from datetime import datetime as dt

def drop_device(x):
    return {k:v for k, v in x.items() if k != 'device'}

def user_info(x):
    user_info_name = {"name", "email", "age", "adress"}
    return {**{
        'user_info': {
            k:v
            for k, v in x.items() if k in user_info_name
        },
    }, **{k: v for k, v in x.items() if k not in user_info_name}}

def change_create_time(x):
    res = x.copy()
    res['create_time'] = dt.strptime(
        res['create_time'],
        "%Y-%m-%d %H:%M:%S"
    ).timestamp()
    return res
```

然后我们就可以顺序调用，但是他们在计算时则是惰性执行，不是等到所有列表处理完上一步来完成任务：

```python
>>> LazyList(data_stream_gen())\
    .map(drop_device)\
    .map(user_info)\
    .map(change_create_time)\
    .take(3)\
    .collect()

[{'user_info': {'name': 'Norman Brown',
   'email': 'xmcclain@example.net',
   'age': 41},
  'create_time': 1636093122.0,
  'address': '39251 Kimberly Causeway Apt. 464\nEast Heidiburgh, TX 27435'},
 {'user_info': {'name': 'Amanda Nguyen',
   'email': 'russellpeter@example.com',
   'age': 30},
  'create_time': 1636093122.0,
  'address': '53844 Tina Divide Suite 297\nMaryburgh, IL 85323'},
 {'user_info': {'name': 'Sara Thompson',
   'email': 'zknapp@example.com',
   'age': 46},
  'create_time': 1636093122.0,
  'address': '666 Stephanie Trafficway Apt. 808\nPort Ashleyburgh, WI 50626'}]
```

我们发现，其实这种数据结构是非常好的处理流式数据，事实上，`flink`、`rdd`本质上也就是基于这种基本数据结构实现的。当然总结来说，它的主要好处是：

1. 是一个基于顺序的处理，类似于基本数据结构「队列」。
2. 每一个数据本身是一个最小单元，可以很好地实现锁（这个在之后我们会继续讨论）。
3. 基于一些函数式地规则编写，我们能让整个处理更具有可读性，函数之间的组合也是弱依赖的，实现函数级别的解耦。

下面的篇章里，我们讲稍微离开惰性列表的讨论，开始进入错误处理、测试以及抽象代数结构方面的内容。

{% include comments.html %}
