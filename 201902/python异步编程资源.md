python 异步编程包含比较多的概念，各种概念扯到一起，相当繁琐

## 基础

生成器，协程，yield, yield from

## 文章

### 流程的 python [入门]

第 16、17、18 章

### [深入 Asyncio 系列](https://www.cnblogs.com/ikct2017/p/9828978.html) [入门]

适合了解基础原理。
[深入 Asyncio（三）Asyncio 初体验](https://www.cnblogs.com/ikct2017/p/9828985.html) 对 asyncio 层级进行了描述，推荐阅读。

### Python 的异步 IO 系列 [中等]

讲述了 asyncio 库 ensure_future run_until_complete 的原理。  
实现了 http echo client，内容比较详实。  
非常值得一读。

[Python 的异步 IO：Asyncio 简介](https://segmentfault.com/a/1190000008814676)

[Python 的异步 IO：Asyncio 之 TCP Client](https://segmentfault.com/a/1190000012286062)

### [Asyncio.gather vs asyncio.wait](https://stackoverflow.com/questions/42231161/asyncio-gather-vs-asyncio-wait)

比较了 gather 和 wait 的不同之处

## 源码

[TCP echo client protocol ](https://docs.python.org/3.6/library/asyncio-protocol.html#asyncio-tcp-echo-client-protocol)

[TCP echo client using streams](https://docs.python.org/3.6/library/asyncio-stream.html#asyncio-tcp-echo-client-streams)
