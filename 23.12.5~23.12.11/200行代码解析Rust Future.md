# 200行代码解析绿色线程原理

时间：2023/12/7

源资料：[200行代码讲透RUST FUTURES](https://stevenbai.top/rust/futures_explained_in_200_lines_of_rust/)

## 各种异步编程方式的比较

### （操作系统级）线程：

优点：简单易用，大部分工作由操作系统处理

缺点：开销大，包括保存堆栈等的空间开销和执行系统调用的时间开销。

如果并发请求的数量非常多，不适合为每一个请求都开一个线程。

### 绿色线程

在用户态而非内核态管理更加轻量级的线程。

优缺点与操作系统级线程类似。

注意，本文将“绿色线程”和“Rust目前的，基于Future的协程”做了区分。它们不是同一个东西。

### 基于回调的方法

基于回调的方法背后的整个思想就是保存一个指向一组指令的指针，这些指令我们希望以后在以后需要的时候运行。 针对Rust，这将是一个闭包。 

优点: 

1. 大多数语言中易于实现
2. 没有上下文切换
3. 相对较低的内存开销(在大多数情况下)

缺点:

1. 每个任务都必须保存它以后需要的状态，内存使用量将随着一系列计算中回调的数量线性增长
2. 很难理解，很多人已经知道这就是“回调地狱”
3. 这是一种非常不同的编写程序的方式，需要大量的重写才能从“正常”的程序流转变为使用“基于回调”的程序流
4. 在 Rust 使用这种方法时，任务之间的状态共享是一个难题，因为它的所有权模型

### 延迟计算

例如Rust里的`Future`和Javascript里的`Promise`

## future介绍

本文中对future的定义；在未来完成的操作。

### leaf future和non-leaf future

#### leaf future

看不懂 :\(
    
开摆，直接复制原文吧

    由运行时创建leaf futures，它就像套接字一样,代表着一种资源.

```Rust
// stream is a **leaf-future**
let mut stream = tokio::net::TcpStream::connect("127.0.0.1:3000");
```

    对这些资源的操作，比如套接字上的 Read 操作，将是非阻塞的，并返回一个我们称之为leaf-future的Future.之所以称之为leaf-future,是因为这是我们实际上正在等待的Future.

    除非你正在编写一个运行时，否则你不太可能自己实现一个leaf-future，但是我们将在本书中详细介绍它们是如何构造的。

    您也不太可能将 leaf-future 传递给运行时，然后单独运行它直到完成，这一点您可以通过阅读下一段来理解。

#### non-leaf future

Non-leaf future指的是那些我们用async关键字创建的Future。

通常，Non-leaf future由await一系列leaf future组成.

```Rust
// Non-leaf-future
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap();// <- yield
    println!("connected!");
    let result = stream.write(b"hello world\n").await; // <- yield
    println!("message sent!");
    ...
};
```

non-leaf future可以在等待时切换到其它任务，可以被调度器调度。

#### 我的理解：

leaf future是需要一段时间才能获得的资源本身

non-leaf future是使用了这些资源的代码，需要一段时间才能执行完成。

因为它们都需要一段时间才能完成，所以都属于future。

## 异步运行时

异步运行时可以分为两部分: executor和reactor。

Rust的标准库提供的：

1. 一个公共接口，`Future` trait
2. 一个符合人体工程学的方法创建任务, 可以通过`async`和`await`关键字进行暂停和恢复`Future`
3. `Waker`接口, 可以唤醒暂停的`Future`

标准库不提供调度器，需要使用第三方调度器（如Tokio）

## Waker和Context

Waker：一种类型，其可以在调用`Future::poll()`时传给Future对象。当这个Future对象等待的东西已经准备好时，其可以通过传入的Waker通知executor，自己准备就绪，可以执行。

建立Waker时，需要使用vtable，其机制和Rust的`&dyn Trait`引用、运行时多态相似。

## Pinning的一些实用规则

针对`T:UnPin`(这是默认值),`Pin<'a,T>`完全定价与`&'a mut T`. 换句话说: UnPin意味着这个类型即使在固定时也可以移动，所以Pin对这个类型没有影响。

针对`T:!UnPin`,从`Pin< T>`获取到`&mut T`,则必须使用unsafe. 换句话说,`!Unpin`能够阻止API的使用者移动`T`,除非他写出unsafe的代码.

Pinning对于内存分配没有什么特别的作用，比如将其放入某个“只读”内存或任何奇特的内存中。 它只使用类型系统来防止对该值进行某些操作。
大多数标准库类型实现 `Unpin`。 这同样适用于你在 Rust 中遇到的大多数“正常”类型。 `Future`和`Generators`是两个例外。

Pin的主要用途就是自引用类型,Rust语言的所有这些调整就是为了允许这个. 这个API中仍然有一些问题需要探讨.

`!UnPin`这些类型的实现很有可能是不安全的. 在这种类型被钉住后移动它可能会导致程序崩溃。 在撰写本书时，创建和读取自引用结构的字段仍然需要不安全的方法(唯一的方法是创建一个包含指向自身的原始指针的结构)。

当使用nightly版本时,你可以在一个使用特性标记在一个类型上添加`!UnPin`. 当使用stable版本时,可以将`std::marker::PhantomPinned` 添加到类型上。

你既可以固定一个栈上的对象也可以固定一个堆上的对象.

将一个`!UnPin`的指向栈上的指针固定需要unsafe.

将一个`!UnPin`的指向堆上的指针固定,不需要unsafe,可以直接使用`Box::Pin`.

## 实践过程

### reactor

reactor用于处理leaf future（例如io操作），其接收waker，在leaf future完成时唤醒waker。

waker因此可以告知non-leaf future，其等待的任务已经完成。

### waker，reactor，executor，(non-leaf) future的关系

future在leaf future上等待。

reactor管理leaf future，当leaf future完成，则通过waker唤醒executor。

executor被唤醒后，在同一线程对future调用poll。

future通过reactor查询leaf future完成进度，若完成，则继续执行下去；若未完成，返回Pending结果。

若poll返回值为Pending，则executor睡眠等待。（这样的话，可以避免忙等，提高性能。）