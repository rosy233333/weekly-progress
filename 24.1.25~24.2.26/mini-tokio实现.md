# mini-tokio实现

时间：2024/2/3

原文：[https://tokio.rs/tokio/tutorial/async](https://tokio.rs/tokio/tutorial/async)

## 对`Future`的深入理解

使用`async`声明的函数或代码块，其返回值不是原值，而是`Future`类型。

对`Future`使用`.await`得到原值。

当`Future`之间相互嵌套（外层`Future`使用`.await`调用内层`Future`）时，`poll`外层`Future`会引起`poll`内层`Future`。

## 简单版本的tokio：mini-tokio

[源代码](https://github.com/tokio-rs/website/blob/master/tutorial-code/mini-tokio/src/main.rs)

## Waker

如果一个`Future`返回了`Pending`，那它必须已经设置了`Waker`在之后的某个时间会调用，否则该`Future`就永远不会再被调度。

mini-tokio对计时器协程的实现：其先创建一个计时器线程，将协程自身的`Waker` `clone`之后传入线程。线程sleep一个特定的时间后，调用这个`Waker`，从而实现了`Waker`的通知机制。

mini-tokio对`Waker`的实现：其设置了一个通道，`Executor`作为通道的接收方，接收`Task`并放入队列；`Waker`与`Task`对应，作为通道的发送方。当`Waker`被调用时，其将对应的`Task`放入管道。