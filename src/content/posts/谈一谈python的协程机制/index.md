---
title: 谈一谈python的协程机制
published: 2022-08-01
category: 'Python'
tags: ["Python","并发", "协程"]
description: "从Python Async函数的角度，谈一谈python的协程机制"
draft: false
---


由于GIL锁的存在，Python的多线程并不能起到利用多核处理器的作用，对于CPU密集型任务无法提升效率；然而，即使作为IO密集任务的并发机制，线程附带的上下文切换开销也导致了效率的低下。这种场景下，python使用await/async关键字提供的协程机制，提供了一种低开销的异步IO机制。

## 演进

Python尝试对协程的支持是从Python3.4开始的，其底层机制为Python中的生成器。await关键字包装了yield from

## 机制

异步IO的实现基于几个核心概念：事件循环、协程、Future、Task，以及用于支持协程的关键字await。

### await

在谈论协程的实现之前，我们需要先了解一下await这个关键字的机制。await其实是对yield from这一关键字的封装与修改，相较于yield from， await放弃了对基于生成器的协程的支持，转而提出了一个可等待对象(awaitable)的概念。可等待对象需要实现可等待协议，即实现__await__方法，并返回一个生成器。于是，我们可以应用await的场景便分为两种，一种是coroutine，另一种是可等待对象。

await关键字分为两步进行：

1. 返回coroutine(也是一种特殊的生成器，需要增加注解来标识)本身或调用可等待对象的__await__方法
2. 使用yield from，反复调用生成器的send方法，若yield出对象，则保存函数帧，将yield出的对象返回给更上一层函数；否则，将返回值或StopIteration异常的值作为返回值继续执行。

如此，await实现了协程的暂停与恢复，以及对可等待对象的支持。

### 事件循环

回想刚开始coding的时光，我们编写的程序总是从main开始，然后执行一些操作，而后很快地，以return收尾。但在现实应用场景中，提供服务的软件往往需要不停地运行，如Web服务器、嵌入式设备等。这些程序需要一种机制来处理事件，这就是事件循环。

协程这种需要调度多个任务的场景，亦需要一个事件循环。事件循环由什么元素组成呢？一般来说，需要一个任务队列——schedule_list。如果不考虑阻塞事件，显然，我们只需要按序执行完其中的所有任务，就结束了。但显然，schedule_list中的事件会触发许多与外部产生交互的事件，如网络请求、文件读写等。按照同步的方式，我们可以一一等待之，但这将不可避免地耗费许多时间，让CPU乃至线程空等。在多线程机制下，一般会让CPU调度其他线程来执行；而在协程机制下，我们就需要手动转移控制流。这一机制的实现，就依赖于协程的暂停与恢复，以及内核提供的IO与定时器库了。

### 协程

一般的同步代码中，函数调用将自己保存本地变量，然后再进入子过程栈帧，最后退出，再恢复之；而异步IO中需要营造出“假同步”的表象，便需要我们自己维护协程的状态，包括局部变量、栈帧、堆栈、指令计数器等等。

仔细想来，在python中，确乎存在这种执行到一半，暂时退出，下次再执行的机制——生成器。

仔细看下面这个函数，当执行g = generator时，并不会立即执行generator函数，而是会返回一个generator对象，这个函数有两个重要接口——next()和send()。

```python
def generator():
    yield 1
    f = yield 2
    yield 3
    return 0

g = generator()

g.next() # 1
g.next() # 2
g.send(4) # 3
try:
    next(g) # StopIteration
except StopIteration as e:
    print(e.value) # 0
```

每次调用iterator的next()方法，都会返回下一个yield表达式的值。（首次调用时常常需要先执行一次next或send(None),使其执行到第一个yeild处，便于传参）

generator机制能够提供一种lazy的iterator，即读取时才生成必要数据，减少内存占用；但在此处，却也为协程提供了一种绝好的机制，使用next/send调度协程，协程使用yeild/yeild from/await方法交出控制权，并指明等待对象——Future，以达到转移控制流的效果。

### Future

先从学术角度谈谈Future。Future是协程通信机制中的一种，其作为一个跨越于多个协程之间的对象，由等待方进行监听，由被等待方进行触发。监听的实现一般依靠调用Future对象的set_callback方法，将回调函数传入；而被等待方则调用Future对象的set_done方法，不仅结束Future对象，还对回调函数进行调用——回调内容往往是将等待方加入事件循环的调度队列中。而对回调函数的调用则是延迟调用，即将回调函数也加入事件循环，等待事件循环的调度。

> 在一般的被动式（响应式）编程方案中，我们一般通过传入回调函数来实现异步IO，这种方式的缺点是：当我们的响应函数嵌套太深，回调函数调用路径过长，将导致代码难以维护，同时也使得运行时在回调处理时间过长。延迟调用能很好地避免这方面问题

从基础定义上来看，Future是一个静态对象，我们可以将其set_done方法绑定到操作系统内核中提供的异步IO接口，或自己实现的epoll事件、定时器队列上，如此，当任务所等待的外部事件完成后，便可借由future再度回到任务队列中，等待调度。

### Task

Task是对协程的一层封装，是事件循环调度任务的基本单位。除了持有一个coroutien，对外提供coroutine的接口外，Task亦继承了Future。为什么呢？我们需要考虑到，task除了需要等待外部事件外，有时还存在task对task的依赖，即taskA完成后taskB才能执行。于是，我们需要task本身也成为一个可等待对象。

## 实战——事件循环

通过前面的论述，我们已经充分了解了一个事件循环的基本构成，接下来通过实现一个简单的事件循环来加深对事件循环的理解。

```python

import time
import socket
from functools import partial
import select

def coroutine(name):
    loop = yield None
    yield None
    print(name+"A")
    yield loop.sleep(3)
    print(name+"B")
    sock = socket.socket()
    sock.connect(("www.baidu.com",80))
    sock.send("GET / HTTP/1.1\r\n".encode("utf-8"))
    sock.send(("Host: "+"www.baidu.com"+"\r\n").encode("utf-8"))
    sock.send(("\n").encode("utf-8"))
    future = Future()
    # 将自己注入回去
    loop.add_handler(sock.fileno(), lambda: future.set_done("done"), select.EPOLLIN)
    yield future
    data = sock.recv(1024)
    print(name)
    print(data)
    task = Task(coroutine2(name+"-2"),loop)
    loop.add_task(task)
    yield task
    print(name+"Over")
    return "Over"

def coroutine2(name):
    loop = yield None
    yield None
    print(name+"A")
    yield loop.sleep(3)
    print(name+"Over")
    return "Over"


class TimerEvent:
    def __init__(self, secs, callback):
        self.deadline = time.time() + secs
        self.callback = callback

    def call(self):
        self.callback()


class Future:

    def __init__(self):
        self.callbacks = []
        self.value = None

    def add_callback(self, callback):
        self.callbacks.append(callback)

    def set_done(self, value):
        self.value = value
        for callback in self.callbacks:
            callback()

    def get_result(self):
        return self.value


class Task(Future):
    def __init__(self, gen,loop):
        Future.__init__(self)
        self.gen = gen
        self.loop = loop
        self.gen.send(None)
        self.gen.send(loop)

    def step(self):
        try:
            yielded = next(self.gen)
            if isinstance(yielded, Future):
                yielded.add_callback(partial(self.loop.task_list.append, self))
            else:
                self.loop.task_list.append(self)
        except StopIteration as e:
            self.set_done(e.value)

        

class EventLoop:
    """
    事件循环
    """
    def __init__(self,*coroutines):
        self.task_list = [Task(c,self) for c in coroutines]
        self.poll = select.epoll()
        self.io_event_list = dict()
        self.timer_event_list = []

    def run(self):
        """
        运行事件循环
        """
        #默认等待时间，用于决定epoll等待时间
        default_wait_time = 10
        while True:
            #无待处理事件，结束循环
            if not self.task_list and not self.timer_event_list and not self.io_event_list:
                break
            now = time.time()                                                                                                                                                                               
            for timer_event in self.timer_event_list[:]:
                if timer_event.deadline <= now:
                    timer_event.call()
                    self.timer_event_list.remove(timer_event)

            while self.task_list:
                task = self.task_list.pop(0)
                task.step()
            # 修改一下下次的睡眠时间，防止睡眠时间过长，错过定时器
            min_left_time = default_wait_time
            for timer_event in self.timer_event_list:
                left_time = timer_event.deadline - now
                if left_time <= 0:
                    left_time = 0
                if left_time<min_left_time:
                    min_left_time = left_time
            next_wait_time = min(default_wait_time, min_left_time)

            # 等待IO事件（阻塞）
            events = self.poll.poll(next_wait_time)
            while events:
                fd,event = events.pop()
                self.io_event_list[fd]() # 调用回调函数，即Future的set_done方法
                self.poll.unregister(fd)
                self.io_event_list.pop(fd)

    def add_handler(self,fd, handler, events):
        self.io_event_list[fd] = handler
        self.poll.register(fd, events)
    
    def sleep(self, secs):
        future = Future()
        self.timer_event_list.append(TimerEvent(secs, partial(future.set_done, None)))
        return future
    
    def add_task(self,task:Task):
        self.task_list.append(task)

if __name__ == "__main__":
    loop = EventLoop(coroutine("1"),coroutine("2"),coroutine("3"))
    loop.run()

```

这段示例代码中，通过使用最为基础的生成器，以及yield与send方法，模拟出了一个带有定时器事件、IO事件、任务事件的事件循环。当然，此处还未使用到wait、__wait__等机制，仅仅是在最简单的逻辑层面进行了一个实现，读者不妨本地运行试试，帮助理解。

PS: 代码中使用了epoll，这是Linux特有的IO机制，在Windows下需要替换为select。

## 实战——实现一个Sleep

python协程的使用案例网络上的例子相当之多，此处我不打算再重复一遍，而是通过手动实现一个sleep功能，来帮助深入理解协程及其事件循环的机理。

下面是一段再寻常不过的异步代码，但是其中的sleep的实现却涉及到事件循环的Timer、以及直接yield future。

```python
async def main():
    await asyncio.sleep(1)
    print('Hello')

asyncio.run(main())
```

首先我们要知道，python原生的协程中是不能使用yield关键字的，其只能使用await(yield from)。于是乎，我们无法在async def中通过生成future再抛出的方式来实现sleep。剩下的方案便剩下求助于基于生成器的协程。asyncio中便是如此实现的。

下面为python中对sleep函数的实现。可以看到，由于使用的是基于生成器的协程，可以直接yield一个None，以及使用yield from一个future。

```python
@coroutine
def sleep(delay, result=None, *, loop=None):
    """Coroutine that completes after a given time (in seconds)."""
    
    class helper:
        
        def __await__(self):
            yield None
        
        __iter__ = __await__
    
    if delay==1:
        yield from helper()
        return result

    if loop is None:
        loop = events.get_event_loop()
    future = loop.create_future()
    h = future._loop.call_later(delay,
                                futures._set_result_unless_cancelled,
                                future, result)
    try:
        return (yield from future)
    finally:
        h.cancel()
```

> 值得注意的是，在设置h的时候，我们使用了future._loop.call_later，这个函数的作用是设置一个定时器，当定时器触发时，将future的结果设置为result。

但是，我们知道，现在在python中使用@coroutine注解是一种depreciated的行为，并且python计划在python3.11版本移除这个注解，那么届时该如何实现sleep呢？

回顾我们对await关键字的讨论，其后面可以跟三种东西: coroutines、awaitables。既然我们无法在coroutines中使用yield，那么我们只能使用awaitables。

如下，我们在协程test2中定义了一个helper类，其中实现了__await__方法，这个方法返回一个生成器，其中按照delay yield一个设置了定时器的future。

```python
import asyncio
import selectors
from asyncio import futures,events
from types import coroutine

async def test2():
    class helper:
        
        def __init__(self,delay):
            self.delay = delay
        
        def __await__(self,loop=None):
            delay = self.delay
            result = ""
            if delay==1:
                yield None
                return result

            if loop is None:
                loop = events.get_event_loop()
            future = loop.create_future()
            h = future._loop.call_later(delay,
                                        futures._set_result_unless_cancelled,
                                        future, result)
            try:
                return (yield from future)
            finally:
                h.cancel()

        #__iter__ = __await__
    await helper(2)

selector = selectors.SelectSelector()
loop = asyncio.SelectorEventLoop(selector)
asyncio.set_event_loop(loop)
loop.run_until_complete(test2())
```

如此便能够实现自己的sleep，类似的，对于epoll的事件注册也可以通过这种形式来完成。

## 总结

Python使用原有的生成器机制，实现了一种类似状态机的无栈协程，并通过Future对象在任务、外部事件间建立依赖关系，从而实现有序的控制流。

### 拓展

本文对await、Future、Task、EventLoop的机制进行了分析，并通过代码对其进行模拟实现。本文中的事件循环是基于普通生成器的，而Python3.5后就对普通生成器与协程进行了区分，并提供async与await关键字作为协程接入事件循环的接口。Python自身也提供了标准的事件循环相关库——asyncio。 此外，python还提供了许多协程相关机制，如async for/async with等。本文就暂且略过，留给读者自行探索。

### 应用场景

协程机制为受限于GIL的python提供了除多进程外的又一高效并发方式，但与多进程不同的是，协程仅在IO密集型任务中才能发挥其优势，而在CPU密集型任务中，协程的优势并不明显，还是应当使用基于Process的多进程来实现。
