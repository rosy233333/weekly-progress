# tokio和io_uring阅读笔记

原文：[io-uring 与 Tokio 的华丽合体：Rust 异步 IO 的零拷贝革命](https://mp.weixin.qq.com/s/FHaJ6_7alzLRezLhFBctgw?click_id=1)

## 原文阅读

原文所述，不是tokio官方提供了对io_uring的支持，而是自己实现的使用tokio和io_uring的异步I/O方式。

而文中提供的两个示例代码均有不足，分析如下：

### 示例代码1

该代码使用io_uring实现了`IouringReader`类型，并为其实现了tokio中的`AsyncRead` trait。

然而，`AsyncRead`中的`poll_read`接口在返回`Pending`时需要实现相应的协程唤醒机制，而本示例中的代码并未实现该唤醒机制。

因此，该示例代码中实现的`IouringReader`也无法直接在tokio运行时中使用。在该示例代码中，是通过在循环中不断调用`poll_read`来使用`IouringReader`的。

同时，该代码读取的内容会先存放在`IouringReader`的内部缓冲区中，之后再拷贝到调用者的缓冲区中。如此就没有达到文中宣称的零拷贝优势。

### 示例代码2

该代码在主协程提交SQE后，并创建了另一个blocking task（可以理解为，在另一个线程中创建了一个协程）不断轮询CQ获取结果。

该设计与ldh的设计类似，通过一个独立的任务（dispatcher）轮询CQ，获取结果并唤醒工作协程。

## tokio中实现`AsyncRead`的示例

tokio中的`File`实现了`AsyncRead`。`File`的内部状态如下：

```Rust
enum State {
    Idle(Option<Buf>),
    Busy(JoinHandle<(Operation, Buf)>),
}
```

在`Idle`状态下，其内部缓冲区已有读到的内容，直接拷贝到调用者缓冲区中再返回即可。

在`Busy`状态下，其会对`JoinHandle`调用`poll`，等待读操作的完成。（`JoinHandle`指向的任务在内部缓冲区被读完时创建，读取更多内容到内部缓冲区中）

## tokio与io_uring的整合情况

目前，tokio方也在进行整合io_uring的工作：

- [tokio_uring](https://github.com/tokio-rs/tokio-uring.git)库，为tokio运行时提供了io_uring支持。
- [【Rust日报】2025-05-27 tokio 准备将 io-uring 用于文件 IO](https://rustcc.cn/article?id=02a654fc-b6c1-4906-8942-5180e5bd8d73&current_page=1)

仍需进一步调研：tokio_uring的内部实现；io_uring与tokio的整合情况。
