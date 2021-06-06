---
title: 初识Python asyncio异步编程
date: 2019-09-23 15:58:52
categories: 后端
tags: [python,异步,asyncio]
---

同步代码(synchrnous code)
我们都很熟悉，就是运行完一个步骤再运行下一个。要在同步代码里面实现"同时"运行多个任务，最简单也是最直观地方式就是运行多个 threads 或者多个 processes。

<!--more-->

这个层次的『同时运行』多个任务，是操作系统协助完成的。 也就是操作系统的任务调度系统来决定什么时候运行这个任务，什么时候切换任务，你自己，作为一个应用层的程序员，是没办法进行干预的。

我相信你也已经听说了什么关于 thread 和 process 的抱怨：process 太重，thread 又要牵涉到很多头条的锁问题。尤其是对于一个 Python 开发者来说，由于GIL（全局解释器锁）的存在，多线程无法真正使用多核，如果你用多线程来运行计算型任务，速度会更慢。

异步编程与之不同的是，值使用一个进程，不使用 threads，但是也能实现"同时"运行多个任务（这里的任务其实就是函数）。

这些函数有一个非常 nice 的 feature：必要的可以暂停，把运行的权利交给其他函数。等到时机恰当，又可以恢复之前的状态继续运行。这听上去是不是有点像进程呢？可以暂停，可以恢复运行。只不过进程的调度是操作系统完成的，这些函数的调度是进程自己（或者说程序员你自己）完成的。这也就意味着这将省去了很多计算机的资源，因为进程的调度必然需要大量 syscall，而 syscall 是很昂贵的。


### 一 定义一个简单的协程：
```
import asyncio
  
async def execute(x):
    print('Number:', x)
    return x
  
coroutine = execute(1)
print('Coroutine:', coroutine)
print('After calling execute')
  
loop = asyncio.get_event_loop()
task = loop.create_task(coroutine)
print('Task:', task)
loop.run_until_complete(task)
print('Task:', task)
print('After calling loop')
 ```
 
print('Task Result:', task.result())  这样也能查看task执行的结果

```
Coroutine: <coroutine object execute at 0x10e0f7830>
After calling execute
Task: <Task pending coro=<execute() running at demo.py:4>>
Number: 1
Task: <Task finished coro=<execute() done, defined at demo.py:4> result=1>
After calling loop
```

我们使用 async 定义了一个 execute() 方法，方法接收一个数字参数，方法执行之后会打印这个数字。
随后我们直接调用了这个方法，然而这个方法并没有执行，而是返回了一个 coroutine 协程对象。

随后我们使用 get_event_loop() 方法创建了一个事件循环 loop，并调用了 loop 对象的 run_until_complete() 方法将协程注册到事件循环 loop 中，然后启动。最后我们才看到了 execute() 方法打印了输出结果。可见，async 定义的方法就会变成一个无法直接执行的 coroutine 对象，必须将其注册到事件循环中才可以执行。

我们也可以不使用task来运行，它里面相比 coroutine 对象多了运行状态，比如 running、finished 等，我们可以用这些状态来获取协程对象的执行情况。将 coroutine 对象传递给 run_until_complete() 方法的时候，实际上它进行了一个操作就是将 coroutine 封装成了 task 对象，如：

```
import asyncio

async def execute(x):
    print('Number:', x)

coroutine = execute(1)
print('Coroutine:', coroutine)
print('After calling execute')

loop = asyncio.get_event_loop()
loop.run_until_complete(coroutine)
print('After calling loop')
```
查看了源码，正好可以验证上面这一观点：

run_until_complete()这个方法位于源码中的base_events.py，函数有句注释：
Run until the Future is done.If the argument is a coroutine, it is wrapped in a Task.



### 二 发送网络请求结合aiohttp实现异步：
我们用一个网络请求作为示例，这就是一个耗时等待的操作，因为我们请求网页之后需要等待页面响应并返回结果。耗时等待的操作一般都是 IO 操作，比如文件读取、网络请求等等。协程对于处理这种操作是有很大优势的，当遇到需要等待的情况的时候，程序可以暂时挂起，转而去执行其他的操作，从而避免一直等待一个程序而耗费过多的时间，充分利用资源。为了测试，我自己先通过flask 创建一个实验环境：
```
from flask import Flask
import time
  
app = Flask(__name__)
  
@app.route('/')
def index():
    time.sleep(3)
    return 'Hello!'
  
if __name__ == '__main__':
    app.run(threaded=True)
```

开始测试...
```
import asyncio
import aiohttp
import time
  
start = time.time()
  
async def get(url):
    session = aiohttp.ClientSession()
    response = await session.get(url)
    result = await response.text()
    session.close()
    return result
  
async def request():
    url = 'http://127.0.0.1:5000'          # 访问flask搭建的服务器（睡眠3秒），模仿IO阻塞
    print('Waiting for', url)
    result = await get(url)
    print('Get response from', url, 'Result:', result)
  
tasks = [asyncio.ensure_future(request()) for _ in range(5)]
loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait(tasks))
  
end = time.time()
print('Cost time:', end - start)
```

运行结果...
```
Waiting for http://127.0.0.1:5000
Waiting for http://127.0.0.1:5000
Waiting for http://127.0.0.1:5000
Waiting for http://127.0.0.1:5000
Waiting for http://127.0.0.1:5000
Get response from http://127.0.0.1:5000 Result: Hello!
Get response from http://127.0.0.1:5000 Result: Hello!
Get response from http://127.0.0.1:5000 Result: Hello!
Get response from http://127.0.0.1:5000 Result: Hello!
Get response from http://127.0.0.1:5000 Result: Hello!
Cost time: 3.0199508666992188
```
我们发现这次请求的耗时由 15 秒变成了 3 秒，耗时直接变成了原来的 1/5。

代码里面我们使用了 await，后面跟了 get() 方法，在执行这五个协程的时候，如果遇到了 await，那么就会将当前协程挂起，转而去执行其他的协程，直到其他的协程也挂起或执行完毕，再进行下一个协程的执行。


## 二 总结
协程"同时"运行多个任务的基础是函数可以暂停（await实际就是用到了yield）。上面的代码中使用到了 asyncio的 event_loop，它做的事情，本质上来说就是当函数暂停时，切换到下一个任务，当时机恰当（这个例子中是请求完成了）恢复函数让他继续运行（这有点像操作系统了）。
