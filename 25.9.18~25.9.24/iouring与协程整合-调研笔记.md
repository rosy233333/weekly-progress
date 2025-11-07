# iouring与协程整合-调研笔记

## C++20协程

因为搜到的许多实现都基于C++，因此了解一些C++协程的基本知识。

参考资料：[C++20协程入门篇](https://lqxhub.github.io/posts/541b707d/)

C++中的协程支持也和Rust一样，没有提供语言级的运行时支持。

**协程函数**：返回协程句柄的函数。应该能类比于Rust的async函数。协程函数中可以使用`co_await`、`co_yield`和`co_return`关键字。

**协程句柄**：代表一个协程。其类型为包含了`promise`类型的模板类型。

**promise**：用于保存协程状态的状态机对象。该类型需要实现以下虚方法：

- `get_return_object()`：返回协程的句柄
- `initial_suspend()`：协程开始时是否挂起
- `final_suspend()`：协程结束时是否挂起
- `yield_value()`：协程中使用 `co_yield` 返回值时的处理
- `return_void()`：协程返回时是否返回值
- `unhandled_exception()`：处理未捕获的异常

**awaitable对象**：需要实现以下虚方法：

- `await_ready()`：用于检查异步操作是否已经完成。
- `await_suspend()`：用于挂起协程并等待异步操作的完成。
- `await_resume()`：用于恢复协程并返回异步操作的结果。

协程await一个awaitable对象时的行为：

```
--> co_await --> await_ready() --false-> await_suspend() --> （IO操作完成） --> await_resume() --> 协程恢复执行


```

awaitable对象类似于Rust协程概念中的Reactor和Leaf Future。
