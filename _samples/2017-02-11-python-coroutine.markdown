---
layout: post
tags:
- python
- coroutine
title: python中的协程
---
在之前的文章中有提到到协程和线程在实现上的主要区别之一在于，线程是由内核进行切换，用户无法控制切换时间，而协程则是由用户自己主动交出控制权。这里的用户不是特指程序员，而是代码本身。

协程也许算是python中最难理解的概念之一，要理解协程，你必须得抛弃传统的调用过程，即调用->退出。从某种程度来说，协程和生成器（generator）的工作机制很相似。一个典型的生成器如下：

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

上面的例子已经满足了协程中的自主调度的要求。如果把数据生成，想象成IO请求，生成器在此yield返回。同时有一个更上层的对象，管理着所有资源的状态和所有生成器的调用。一旦请求资源空闲，就将其返回给该生成器。这个概念是不是很像我们所期望的协程？

当然我们现在离这个状态还有点远，我在试图写这个例子的时候从youtube上找到了一个极佳的讲解，非常深入简出。这里我就老老实实的将这位的讲解学习一遍。

# Everything starts from Generator (see?...)
关于使用yield来创建生成器的过程就不在复述了。第一个关键点在于 **调用生成器函数返回一个生成器对象，但是函数本身并没有执行。** 这意味着如果要运行起来到第一个yield，起码要调用一次生成器的next方法。比如下面这个例子：

```python
def count(n):
    print "Start counting"
    while n > 0:
        yield n
        n -= 1

c = count(10)
print c
c.next()
for x in c:
    print x
```

输出：

```
<generator object count at 0x10bcb63c0>
Start counting
10
9
8
7
6
5
4
3
2
1
```

利用生成器的特性，即产生序列数据，可以做很多有意思的操作，比如pipeline。将一个生成器产生的数据传递给第二个生成器使用，第二个生成器并不需要感知到上一个生成器，只需要将其作为序列数据使用即可。

```python
import time

def follow(file):
    file.seek(0,2)
    while True:
        line = file.readline()
        if not line:
            time.sleep(0.1)
            continue
        yield line

def grep(pattern, lines):
    print ("looking for pattern %s" % pattern)
    for line in lines:
        if pattern in line:
            yield line


logfile = open("/tmp/test.log")
loglines = follow(logfile)
results = grep("python", loglines)

for line in results:
    print line
```


### 使用协程消费数据
这里第二个grep方法，是使用了follow产生的序列数据，他们两个本质上还都是生成器。如果让follow把数据传给grep呢？

```python
import time

def follow(target, file):
    file.seek(0,2)
    while True:
        line = file.readline()
        if not line:
            time.sleep(0.1)
            continue
        target.send(line)

def grep(pattern):
    print('looking for pattern %s' % pattern)
    while True:
        line = yield
        if pattern in line:
            print line

logfile = open("/tmp/test.log")
filter = grep()
filter.next()
follow(logfile, filter)

```


使用yield获取数据，这就是协程和生成器的主要区别。

### 使用coroutine装饰器
不觉得每次新建一个协程对象的时候就需要先调用一下next来让它进入ready状态很麻烦吗？写个装饰器来解决这个问题吧，调用方法的时候直接给你返回一个准备好的协程。

```python
import functools

def coroutine(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        cr = func(*args, **kwargs)
        cr.next()
        return cr

    return wrapper
```

### 关闭协程
使用close方法，可以使该协程对象退出。

```python
g = grep()
g.close()
```

### 由生成器到协程的总结
1. 生成器负责生成序列数据
2. 协程消费数据
3. 两者是不同的概念

# 使用协程做一些更有趣的事情，比如pipeline和数据流控制
协程可以消费数据，它也可以把数据发送给别的协程。对于一个pipeline来说，它当然有起始有结束。起始往往不是一个coroutine，因为没人传数据给它。而结束点则是一个普通的coroutine，它接收数据并处理，也不转发给别人。其它所有的节点都同时接受数据，又会把处理后的数据发送给下一个节点。（像不像WSGI？）

使用生成器也可以做pipeline，上面的follow和grep既是一个例子，只是缺乏了中间节点，但本质是一样的。如果我们复写一下grep，让它可以接收一个协程参数，那么它就可以将过滤后的数据传给下一个处理节点。

```python
def grep(pattern, target):
    print('looking for pattern %s' % pattern)
    while True:
        line = yield
        if pattern in line:
            target.send(line)
```

这种做法类似与OO设计里的handler，但是使用协程有两个优势:

1. 不需要创建类，对象，直接使用函数，更简单。
2. 性能更好。


# 使用coroutine来做event dispatching

使用coroutine来做dispatching与pipeline非常相似，我并没有找到真正的不同点。使用协程来做sax xml parser的事件处理，每个协程可接受一个target，转发消息。

# 使用coroutine来实现并发编程

终于开始进入正题。回到了开篇时的设想：协程与线程和进程是如此的相似，都有自己的运行状态，都可以接收数据，而且协程的看起来还更简单一点，不像线程要通过queue，也不像进程需要通过消息队列。

首先，我们可以把coroutine打包进一个更上层的layer，比如还是叫它线程。

```python
@coroutine
def threaded(target):
    messages = Queue()
    def run_target():
        while True:
            item = messages.get()
            if item is GeneratorExit:
                target.close()
                return
            else:
                target.send(item)

    Thread(target=run_target).start()
    try:
        while True:
            item = yield
            messages.put(item)
    except GeneratorExit:
        messages.put(GeneratorExit)
```

threaded本身就是一个协程，它将收到的数据全部放进queue里。另外会新起一个线程，不停的抓取queue里面的数据并发送给封装的协程。
<<<<<<< HEAD

需要注意的是，因为数据要通过queue来发给协程，而不是直接send进去，导致这个例子比直接用协程慢了很多。

作者还给出了用subprocess来做实现的例子，但是主要的目的是想说明使用coroutine，可以将实现封装在协程内部，外部的调度系统无论是想用线程还是进程都可以。

=======
>>>>>>> b829071347f7a76a2d2e5517e4e25c10c18b3818

# 使用eventlet
