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