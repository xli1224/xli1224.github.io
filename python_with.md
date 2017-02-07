# with怎么用
with语句是一种与异常处理相关的功能。适用于对资源进行访问的场合，确保不管使用过程中是否发生异常都会执行必要的“清理”操作，比如文件使用后自动关闭、线程中锁的自动获取和释放等。

先举几个例子。比如一般的文件读取，需要这样:

```python
    try:
        f = open('./with.txt')
        print f.readlines()
    finally:
        f.close()
```

必须保证打开的文件，不论中间发生了什么异常都要close掉，才能释放文件锁，所以才会将f.close()放在final块里。同样的，对于多线程情况下，关键代码处的互斥，我们经常使用Lock。比如：

```python
import threading

lock = threading.Lock()
count = 0

class MyThread(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)

    def run(self):
        global count
        try:
            if lock.acquire():
                count += 1
                print count
        finally:
            lock.release()
```

首先判断是否拿到锁，然后在结束后要释放掉。如果发生了异常，而又没有释放锁的话，其他线程都会等在这个锁上。如果使用with，上面这两个例子就会变成这样：

```python
with open('./with.txt') as f:
    print f.readlines()
```

```python
class MyThread2(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)

    def run(self):
        global count
        with lock:
            count += 1
            print count
```

从上面的例子来看，最直观的感受就是with去掉了finally所需操心的东西。那么为什么open()和lock可以与with一起用呢？with主要做了两件事，第一件就是准备工作，比如打开文件，获取锁；第二件就是结束工作，文件关闭，释放锁。这些都被封装在一个叫做context manager的东西里，谁是context manager？open返回的对象就是，lock自身也是。

# Context Manager
关于具体的context manager定义，可以参考https://docs.python.org/release/2.5.2/lib/typecontextmanager.html

一个context manager对象，必然是实现了\_\_enter\_\_()和\_\_exit\_\_()方法。在with进入时，会去调用enter方法，而在with代码块结束时（try finally包起来），会去调用exit方法。如果我们不用with，那么lock的代码可以这样写

```python
class MyThread3(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)

    def run(self):
        global count

        lock.__enter__()
        try:
            count += 1
            print count
        finally:
            lock.__exit__()
```

手工调用\_\_enter\_\_和\_\_exit\_\_和原本想做的acquire和release是同样的效果，这就是因为lock本身在实现context manager时所定义的\_\_enter\_\_和\_\_exit\_\_方法正是如此。用了with，除了能自动调用\_\_enter\_\_和\_\_exit\_\_之外，还能自动帮我们针对with下面的代码块进行try和finally，在合适的地方调用这两个上下文方法。

同理可以推断，open()所返回的对象，也是同样的实现，在自己的__exit__方法里面实现了对file的close。

# 自己实现context mananger
从上面的例子不难理解，实现context mananger的一种途径就是定义__enter__和__exit__方法。比如：

```python
class WithAdoptable(object):
    def __init__(self):
        return

    def __enter__(self):
        print "Do something with calling with"

    def __exit__(self, exc_type, exc_val, exc_tb):
        print "Do something when with ends"
        
with withAdoptable:
print "Do something in the middle"
```

>Do something with calling with
Do something in the middle
Do something when with ends

还有一种办法，就是利用contextmanager模块，contextmananger会自动帮被装饰对象加上\_\_enter\_\_和\_\_exit\_\_方法。so上有一个很棒的例子。这个例子其实更精确的说明了流程：
1. \_\_enter\_\_的时候直接进入了working_directory方法，切换到目标目录
2. yield跳回，相当于enter方法结束
3. 外面执行完自己的事情以后，with退出调用\_\_exit\_\_，返回到上次的yield，运行finally部分

也就是说yield之前的语句相当于enter，yield之后的相当于exit。如果想返回一个值来使用with...as...的用法，那么可以直接使用yiled obj的方式返回。

```python
from contextlib import contextmanager
import os


@contextmanager
def working_directory(path):
    current_dir = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(current_dir)


with working_directory("data/stuff"):
    # do something within data/stuff
# here I am back again in the original working directory
```