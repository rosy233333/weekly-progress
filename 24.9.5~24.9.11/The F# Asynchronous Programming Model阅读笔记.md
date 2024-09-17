# The F# Asynchronous Programming Model阅读笔记

时间：2024/9/10

在查找`async/await`语法历史时找到的文章。本文在F#语言中提出了异步语法，并之后在C#语言中改进为我们熟知的`async/await`语法。

## Introduction

**背景**：在**事件响应型**的程序中，程序的执行流程由事件的到来决定。这些程序使用异步编程方式来编写。

**异步（非阻塞/重叠）编程的定义**：同时为多个内部或外部事件设置了正在等待的响应动作。

**异步编程的困境**：

- （若使用线程处理事件）OS线程开销太大，而仅使用轻量级线程又会引入额外的开销
- （若使用回调函数处理事件）在没有语言级支持的情况下，难用且受限。

**本文的解决方案：异步编程模式**：

- 使用和源语言的控制流语法相似的语法，但在编译和运行时做不同的解释。
- 使用不干扰原有编程习惯的方式编写轻量级异步任务。
- 本文旨在给出为一门编程语言添加异步编程模式的方法。

## An Overview of F# Asynchronous Programming

`表达式（expr）`：语言中原有的表达式；`异步表达式（aexpr）`：为了实现异步编程模式而新增的一种语法结构。

`Async<T>`类型：由`async{}`块包裹`T`类型的`aexpr`得到，一个`Async<T>`对象也属于`expr`。一个`Async<T>`类型的对象可以视为一个任务生成器（task generator），它们在创建时不会自动运行，而是在将它们传入特定函数后，就会创建一个异步任务。

`aexpr`的定义：

- `expr`属于`aexpr`；
- `aexpr`与流程控制语句（如`return`、`;`、`if`、`while`）的结合属于`aexpr`；
- 特殊语法：
  - `do! expr`、
  - `let! pat = expr in aexpr`
  - `use! val = expr in aexpr`
  - `return! expr`
这些带`!`的语法可以类比为`async/await`中的`await`，其在`aexpr`环境中执行`Async<T>`并获取返回值`T`。

使用`Async.StartImmediate`可以将不返回结果的`Async<T>`作为协程启动，并立刻切换到该协程运行。如果该协程阻塞，则可能切回原任务。协程的阻塞和切换由回调函数实现。

例如，以下代码：

```F#
Let printThenSleepThenPrint = async {
    printfn "before sleep"
    do! Async.Sleep 5000
    printfn "wake up"
}  
Async.StartImmediate printThenSleepThenPrint
printfn "continuing"
```

会打印以下结果：

```Text
;切换到协程
before sleep
;协程阻塞，切换回主任务
continuing
;协程唤醒
wake up
```

`Async.StartImmediate`使用了.NET的`同步上下文`机制，将协程与原任务在同一个线程中运行。`Async.StartInThreadPool`则将协程放入另一个线程中，与原任务并发执行。

异步函数：返回`Async<T>`的函数。可以使用`async{}`块来构造。支持尾调用和尾递归。（而Rust中的`Future`因为使用状态机实现，因此不支持递归。）

异常处理：可以像同步函数一样使用`try ... with ...`等进行异常处理

资源回收：通过状态对象保存异步执行过程中使用的资源，并在异步函数结束后将它们回收。与同步函数有相同的行为。同时，如果异步函数出错或者中途取消，资源也可以正确回收。

异步任务的取消：使用.NET的`CancellationToken`机制，调用`CancellationTokenSource`的`Cancel`方法以取消由其`CancellationToken`管理的异步任务。异步任务会在几个特定的语法结构处检查是否应该取消任务，并取消自己的执行。

## Semantics

提供语义（semantics）的**目的**：

- 给出与具体语言无关的理想实现
- 区分任务的阻塞（在I/O上）、就绪（等待运行）和运行状态
- 提供单线程和多线程（线程池）两种实现

## Patterns for Concurrent and Reactive Programming

使用该异步编程模型可以达成的模式：

### 并行构造

**fork-join并行**：使用`Async.Parallel`函数，将一组异步任务组合为一个异步任务，该任务会推动组合的所有任务的执行。

**基于promise的并行**：使用`Async.StartChild`函数，创建返回一个异步任务的异步任务（`Async.StartChild : Async<'T> → Async<Async<'T>>`）。不过目前看不出这个双层Async有什么作用，也看不出它和promise的关系

### 用状态机实现的反应式客户端

一个编程案例，与引入了异步机制的`MailboxProcessor<T>`类型有关。

### 反应式UI编程

使用`Async.AwaitObservable`响应UI事件

## Implementation

使用计算表达式实现`async`语法，使用一个传入三个后继（分别代表成功、异常和取消）的continuation-passing函数实现`Async<T>`类型。

### C#线程、C#使用回调的异步、F#线程、F#异步在开发时间、代码量、程序的扩展性的比较

![](../图片/屏幕截图%202024-09-15%20153352.png)

使用F#的异步机制，代码量与线程版本接近，开发时间小于线程版本，而C#的基于回调的异步测会带来更大的代码量和编码时间。

### 与其它实现agent models的系统对比

![](../图片/屏幕截图%202024-09-15%20153909.png)

F#的每个agent的开销稍大一些，但消息处理速度更快。（我的理解：pingpong10^5, 1msg是用于比较每个agent的开销的，而pingpong1, 10^7msg是用于比较消息处理速度的）。但这个测试都是理想情况，现实中还会遇到等待I/O等降低性能的事件。

### Summary

本文机制的三个作用：

- 可表示性：通过顺序、递归、模式匹配等语法元素可以构造组合式的异步反应，例如状态机、反应式UI和agent。
- 语义分离：使用不同的语法表示网络I/O与本地I/O（如访问磁盘、访问控制台），从而进行分别处理。
- 可扩展性：异步实现比线程实现开销更低，因此可以同时运行更多的agent。

## 我的总结

该异步机制（后来发展成`async-await`机制）提出时的**主要应用领域**是在网络服务器、GUI程序等事件驱动程序中编写**响应事件**的代码。

该模式的主要**对比对象**是相同应用场景下的**线程**和**回调函数**。在事件驱动场景中，线程开销过大，而回调函数编写难度大。该异步机制可以做到在和线程类似的编写难度（用类似同步的语法写异步代码）下达到和回调函数类似的效率（因为该机制底层使用回调函数实现）。

我猜测，这也是为什么`async-await`协程与异步关系紧密的原因——其本身就是为了实现异步I/O（非阻塞I/O）而设计的。虽然使用`async/await`协程也可以达成和线程类似的其它功能，但异步I/O之外的场景较难体现协程的优势。
