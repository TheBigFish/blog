# tornado 源码之 coroutine 分析

> tornado 的协程原理分析  
> 版本：4.3.0

为支持异步，tornado 实现了一个协程库。

tornado 实现的协程框架有下面几个特点：

1. 支持 python 2.7，没有使用 yield from
   特性，纯粹使用 yield 实现
2. 使用抛出异常的方式从协程返回值
3. 采用 Future 类代理协程（保存协程的执行结果，当携程执行结束时，调用注册的回调函数）
4. 使用 IOLoop 事件循环，当事件发生时在循环中调用注册的回调，驱动协程向前执行

由此可见，这是 python 协程的一个经典的实现。

本文将实现一个类似 tornado 实现的基础协程框架，并阐述相应的原理。

## 外部库

使用 time 来实现定时器回调的时间计算。  
bisect 的 insort 方法维护一个时间有限的定时器队列。  
functools 的 partial 方法绑定函数部分参数。  
使用 backports_abc 导入 Generator 来判断函数是否是生成器。

```python
import time
import bisect
import functools
from backports_abc import Generator as GeneratorType
```

## Future

> 是一个穿梭于协程和调度器之间的信使。  
> 提供了回调函数注册(当异步事件完成后，调用注册的回调)、中间结果保存、结束结果返回等功能

add_done_callback 注册回调函数，当 Future 被解决时，改回调函数被调用。  
set_result 设置最终的状态，并且调用已注册的回调函数

协程中的每一个 yield 对应一个协程，相应的对应一个 Future 对象，譬如：

```python
@coroutine
def routine_main():
    yield routine_simple()

    yield sleep(1)
```

这里的 routine_simple() 和 sleep(1) 分别对应一个协程，同时有一个 Future 对应。

```python
class Future(object):
    def __init__(self):
        self._done = False
        self._callbacks = []
        self._result = None

    def _set_done(self):
        self._done = True
        for cb in self._callbacks:
            cb(self)
        self._callbacks = None

    def done(self):
        return self._done

    def add_done_callback(self, fn):
        if self._done:
            fn(self)
        else:
            self._callbacks.append(fn)

    def set_result(self, result):
        self._result = result
        self._set_done()

    def result(self):
        return self._result
```

## IOLoop

这里的 IOLoop 去掉了 tornado 源代码中 IO 相关部分，只保留了基本需要的功能，如果命名为 CoroutineLoop 更贴切。

这里的 IOLoop 提供基本的回调功能。它是一个线程循环，在循环中完成两件事：

1. 检测有没有注册的回调并执行
2. 检测有没有到期的定时器回调并执行

程序中注册的回调事件，最终都会在此处执行。
可以认为，协程程序本身、协程的驱动程序 都会在此处执行。
协程本身使用 wrapper 包装，并最后注册到 IOLoop 的事件回调，所以它的从预激到结束的代码全部在 IOLoop 回调中执行。
而协程预激后，会把 Runner.run() 函数注册到 IOLoop 的事件回调，以驱动协程向前运行。

理解这一点对于理解协程的运行原理至关重要。

这就是单线程异步的基本原理。因为都在一个线程循环中执行，我们可以不用处理多线程需要面对的各种繁琐的事情。

### IOLoop.start

事件循环，回调事件和定时器事件在循环中调用。

### IOLoop.run_sync

执行一个协程。

将 run 注册进全局回调，在 run 中调用 func()启动协程。  
注册协程结束回调 stop, 退出 run_sync 的 start 循环，事件循环随之结束。

```python
class IOLoop(object):，
    def __init__(self):
        self._callbacks = []
        self._timers = []
        self._running = False

    @classmethod
    def instance(cls):
        if not hasattr(cls, "_instance"):
            cls._instance = cls()
        return cls._instance

    def add_future(self, future, callback):
        future.add_done_callback(
            lambda future: self.add_callback(functools.partial(callback, future)))

    def add_timeout(self, when, callback):
        bisect.insort(self._timers, (when, callback))

    def call_later(self, delay, callback):
        return self.add_timeout(time.time() + delay, callback)

    def add_callback(self, call_back):
        self._callbacks.append(call_back)

    def start(self):
        self._running = True
        while self._running:

            # 回调任务
            callbacks = self._callbacks
            self._callbacks = []
            for call_back in callbacks:
                call_back()

            # 定时器任务
            while self._timers and self._timers[0][0] < time.time():
                task = self._timers[0][1]
                del self._timers[0]
                task()

    def stop(self):
        self._running = False

    def run_sync(self, func):
        future_cell = [None]

        def run():
            try:
                future_cell[0] = func()
            except Exception:
                pass

            self.add_future(future_cell[0], lambda future: self.stop())

        self.add_callback(run)

        self.start()
        return future_cell[0].result()
```

## coroutine

协程装饰器。  
协程由 coroutine 装饰，分为两类：

1. 含 yield 的生成器函数
2. 无 yield 语句的普通函数

装饰协程，并通过注册回调驱动协程运行。
程序中通过 yield coroutine_func() 方式调用协程。  
此时，wrapper 函数被调用：

1. 获取协程生成器
2. 如果是生成器，则
   1. 调用 next() 预激协程
   2. 实例化 Runner()，驱动协程
3. 如果是普通函数，则
   1. 调用 set_result() 结束协程

协程返回 Future 对象，供外层的协程处理。外部通过操作该 Future 控制协程的运行。  
每个 yield 对应一个协程，每个协程拥有一个 Future 对象。

外部协程获取到内部协程的 Future 对象，如果内部协程尚未结束，将 Runner.run() 方法注册到 内部协程的 Future 的结束回调。  
这样，在内部协程结束时，会调用注册的 run() 方法，从而驱动外部协程向前执行。

各个协程通过 Future 形成一个链式回调关系。

Runner 类在下面单独小节描述。

```python
def coroutine(func):
    return _make_coroutine_wrapper(func)

# 每个协程都有一个 future， 代表当前协程的运行状态
def _make_coroutine_wrapper(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        future = Future()

        try:
            result = func(*args, **kwargs)
        except (Return, StopIteration) as e:
            result = _value_from_stopiteration(e)
        except Exception:
            return future
        else:
            if isinstance(result, GeneratorType):
                try:
                    yielded = next(result)
                except (StopIteration, Return) as e:
                    future.set_result(_value_from_stopiteration(e))
                except Exception:
                    pass
                else:
                    Runner(result, future, yielded)
                try:
                    return future
                finally:
                    future = None
        future.set_result(result)
        return future
    return wrapper
```

## 协程返回值

因为没有使用 yield from，协程无法直接返回值，所以使用抛出异常的方式返回。

python 2 无法在生成器中使用 return 语句。但是生成器中抛出的异常可以在外部 send() 语句中捕获。  
所以，使用抛出异常的方式，将返回值存储在异常的 value 属性中，抛出。外部使用诸如：

```python
try:
    yielded = gen.send(value)
except Return as e:
```

这样的方式获取协程的返回值。

```python
class Return(Exception):
    def __init__(self, value=None):
        super(Return, self).__init__()
        self.value = value
        self.args = (value,)
```

## Runner

Runner 是协程的驱动器类。

self.result_future 保存当前协程的状态。  
self.future 保存 yield 子协程传递回来的协程状态。
从子协程的 future 获取协程运行结果 send 给当前协程，以驱动协程向前执行。

注意，会判断子协程返回的 future  
如果 future 已经 set_result，代表子协程运行结束，回到 while Ture 循环，继续往下执行下一个 send；  
如果 future 未 set_result，代表子协程运行未结束，将 self.run 注册到子协程结束的回调，这样，子协程结束时会调用 self.run，重新驱动协程执行。

如果本协程 send() 执行过程中，捕获到 StopIteration 或者 Return 异常，说明本协程执行结束，设置 result_future 的协程返回值，此时，注册的回调函数被执行。这里的回调函数为本协程的父协程所注册的 run()。  
相当于唤醒已经处于 yiled 状态的父协程，通过 IOLoop 回调 run 函数，再执行 send()。

```python
class Runner(object):
    def __init__(self, gen, result_future, first_yielded):
        self.gen = gen
        self.result_future = result_future
        self.io_loop = IOLoop.instance()
        self.running = False
        self.future = None

        if self.handle_yield(first_yielded):
            self.run()

    def run(self):
        try:
            self.running = True
            while True:

                try:
                    # 每一个 yield 处看做一个协程，对应一个 Future
                    # 将该协程的结果 send 出去
                    # 这样外层形如  ret = yiled coroutine_func() 能够获取到协程的返回数据
                    value = self.future.result()
                    yielded = self.gen.send(value)
                except (StopIteration, Return) as e:
                    # 协程执行完成，不再注册回调
                    self.result_future.set_result(_value_from_stopiteration(e))
                    self.result_future = None
                    return
                except Exception:
                    return
                # 协程未执行结束，继续使用 self.run() 进行驱动
                if not self.handle_yield(yielded):
                    return
        finally:
            self.running = False

    def handle_yield(self, yielded):
        self.future = yielded
        if not self.future.done():
            # 给 future 增加执行结束回调函数，这样，外部使用 future.set_result 时会调用该回调
            # 而该回调是把 self.run() 注册到 IOLoop 的事件循环
            # 所以，future.set_result 会把 self.run() 注册到 IOLoop 的事件循环，从而在下一个事件循环中调用
            self.io_loop.add_future(
                self.future, lambda f: self.run())
            return False
        return True

```

![](https://github.com/TheBigFish/blog/raw/master/assets/tornado_simple_routine_1.png)

## sleep

sleep 是一个延时协程，充分展示了协程的标准实现。

创建一个 Future，并返回给外部协程。  
在设置的延时后，会回调 set_result 结束协程。

```python
def sleep(duration):
    f = Future()
    IOLoop.instance().call_later(duration, lambda: f.set_result(None))
    return f
```

## copyright

author：bigfish  
copyright: [许可协议 知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc/4.0/)
