---
layout: post
tags: 
- python
- coroutine
title: python中的协程
---
# 协程
在之前的文章中有提到到协程和线程在实现上的主要区别之一在于，线程是由内核进行切换，用户无法控制切换时间，而协程则是由用户自己主动交出控制权。这里的用户不是特指程序员，而是代码本身。

从某种程度来说，协程和生成器（generator）的工作机制很相似，我也无法确定python中的协程是否就是基于生成器来实现的。一个典型的生成器如下：
```python
def fabbi(upper_limit):
    x = 0
    y = 1
    for i in xrange(upper_limit):
        yield x + y
        y = x + y
        x = y - x


if __name__ == '__main__':
    first_100_num = fabbi(100)
    for i in first_100_num:
        print i
```

对于生成器来说，它的应用范围初看有点类似于数据的管道，不断的生成数据，而并非是生成一批数据并存储下来。但是生成器的使用过程却很类似协程的概念，一方面，它在生成数据后主动返回，交出控制器，另一方面，下次调用，它又能继续上次的位置，stack信息仍然保留着。协程要切换的情况，一般来说是依赖于某个资源的获取。生成器如果做协程的话，该怎么获取数据呢？看个例子：

```python
def coro():
    recv = yield "in coroutine"
    yield recv

c = coro()
print (next(c))
print c.send("send to coroutine")
```

在上面的例子中，第一条yield语句除了返回值给调用者之外，还在等待外部通过send方法来传入变量。具体的步骤如下：

1. coro()创建了一个生成器实例
2. next(c)调用返回"in coroutine"并打印变量
3. c.send("send to coroutine")，进入coro并且把值赋给recv变量，第二条yield又将此变量返回。也就意味着send不仅接收了变量传入，同时也继续运行生成器，直至yield。

上面的例子已经满足了协程中的自主调度的要求。如果把数据生成，想象成IO请求，生成器在此yield返回。同时有一个更上层的对象，管理着所有资源的状态和所有生成器的调用。一旦某个资源空闲，就切换回该生成器。这个概念是不是很像我们所期望的协程？我们可以试一下：



# 使用eventlet

__init__
