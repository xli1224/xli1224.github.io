---
layout: post
title: 进程、线程和协程
---

# 进程、线程和协程

## 线程和协程有什么区别
想要明白协程和线程的区别，首先要搞清楚并发（Concurrency）和并行（Parallelism）的区别。
[[Difference between a “coroutine” and a “thread”] http://stackoverflow.com/questions/1934715/difference-between-a-coroutine-and-a-thread ](#)

> Concurrency is the separation of tasks to provide interleaved execution. Parallelism is the simultaneous execution of multiple pieces of work in order to increase speed.

Concurrency指的是两个或以上的任务，可以在一个重叠时间内启动、执行和结束。它并不意味着多个任务在同一时间由同一实例去运行，比如在一台单核上执行多任务，同一时间仍然只有同一个任务，但是任务之间并非顺序执行，而是在整个生命周期内有重叠。所以并发是个单独的概念，不单指协程、进程或者线程。它是对时间的切片，是一种虚拟的并行。

而Parallelism特指多个任务在同一时间运行，无论是进程、线程还是协程。

对于线程，操作系统会根据自己的调度器算法主动的切换运行中的线程。对于协程，如何切换则是由编程语言和程序员自身决定，通常在一个线程内，由任务在某些节点上主动的暂停和恢复来实现多任务。

站在程序员的角度来说，线程的调度是被动的，由操作系统控制的。协程的调度则是主动的，由程序员控制的。

凡是支持native threads的语言都可以将它的线程（user threads）放到操作系统的线程中执行（kernel threads）。每一个进程都有至少一个kernel thread。kernel thread与进程非常类似，除了同一个进程中的所有线程都共享一个内存空间。一个进程拥有所有分配给它的资源，比如说内存、文件句柄、socket、设备句柄等，这些资源共享给所有此进程的kernel thread。

操作系统的调度器会调度每个线程运行一段时间，即使线程没有在该时间片内结束，调度器也会将它中断并且切换到另一个线程上。在一个多核的机器上，多个线程可以并行的执行，每个线程都可以被调度到一个独立的核上。（对于python来说，由于GIL的存在，多线程仍然被限制在一个核上）

对于单核的机器，线程是通过切片和调度来并发的执行，而非并行执行。

**协程是cooperative functions**，它并非通过在kernel threads上执行和调度，而是在同一个线程上执行，由程序员定义好的yield或finish的时候，调度到其他的function。实现了generator的语言可以用来实现协程。Async/await是对于协程的一种抽象。

Fibers，lightweighted threads和green threads都是类似协程的东西。它们看起来很像是操作系统线程，但实际上并不会像系统线程那样并行运行，而是像协程一样。

线程并非那么高效，每个线程都有自己的stack、thread-local storage，还有线程调度，上下文切换和cpu缓存失效造成的性能损耗。这也是在高性能和高并发领域，协程变得越来越流行的一个原因。在JVM中，每个thread都有自己的stack，一般是1MB。MacOS下，只允许一个进程最多创建2000个线程，而Linux会为每个线程分配8MB的stack，所以线程总数不能超过物理内存/8的数值。

