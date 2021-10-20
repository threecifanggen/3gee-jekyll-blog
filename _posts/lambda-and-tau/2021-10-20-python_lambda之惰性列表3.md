---
layout: post
title:  "Python函数式编程系列011：类实现的列表"
date:   2021-10-20 16:40:30 +0800
category: "lambda-and-tau"
usemathjax: true
tags: ["Python", "函数式编程", "类型", "面向对象"]
---

本文不带有新的内容，仅仅是无副作用地用类重新实现一遍列表的功能。

```python
class ConsList(Generic[S]):
    pass

@dataclass
class Empty(ConsList, Generic[S]):
    def map(self, _: Callable[[S], T]) -> Empty:
        return Empty()

    def filter(self, _: Callable[[S], T]) -> Empty:
        return Empty()
    
    def fold_left(self, _: Callable[[S], T], init: S) -> Empty:
        return init

@dataclass
class Cons(ConsList, Generic[S]):
    head: S
    tail: S

    @classmethod
    def maker(cls, *args: S) -> ConsList[S]:
        if len(args) == 0:
            return Empty()
        else:
            return Cons(args[0], cls.maker(*args[1:]))

    def map(self, f: Callable[[S], T]) -> ConsList[T]:
        return Cons(f(self.head), self.tail.map(f))

    def filter(self, f: Callable[[S], bool]) -> ConsList[S]:
        if f(self.head):
            return Cons(self.head, self.tail.filter(f))
        else:
            return self.tail.filter(f)
            
    def fold_left(self, f: Callable[[S, T], S], init: S) -> S:
            return self.tail.fold_left(f, f(init, self.head))
```

至此，我們可以使用链式地方法来完成列表操作了：

```python
>>> Cons.maker(1, 2, 3, 4).map(lambda x: x + 1).filter(lambda x: x % 2 == 0).fold_left(lambda x, y: x + y, 0)
6
```

比较原来的写法，这样就具有非常高的可读性了。

```python
>>> fold_left_cons(lambda x, y: x + y, 0, filter_cons(lambda x: x % 2 = 0, map_cons(lambda x: x + 1, cons_apply(1, 2, 3, 4))))
6
```

{% include comments.html %}