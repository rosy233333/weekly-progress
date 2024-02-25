# tokio调研

时间：2024/2/1

## tokio架构

- `Runtime`：异步运行时，包含IO、计时器、文件系统、同步、调度功能
- `Hyper`：HTTP客户端、服务器库
- `Tonic`：gRPC客户端、服务器库（应该也是网络应用层面的）
- `Tower`：提供现代网络程序的机制，例如重试、负载均衡、过滤等
- `Mio`：对操作系统的事件IO（evented I/O）的封装
- `Tracing`：数据收集和日志记录
- `Bytes`：处理网络应用中的字节流

![](../图片/屏幕截图%202024-02-01%20120128.png)

我的重点应该是关注`Runtime`和`Mio`。

## tokio编程

### `#[tokio::main]`宏

标记在一个`async`函数上，将其转化为同步的`main`函数，转化后的函数会先初始化协程运行时，再在其中运行原有的函数。

```Rust
// before transform
#[tokio::main]
async fn main() {
    println!("hello");
}

// after transform
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

## Cargo features

tokio有许多feature，可以在`Cargo.toml`中选择使用哪些功能：

```Rust
tokio = { version = "1", features = ["full"] }
```

## tokio api

tokio提供了许多std api的异步版本，且使用了和std版本相同/相似的名称和参数。

## `tokio::spawn`

用于新建协程并运行。在HTTP服务器等程序中，可以将接受信息后的功能代码放在这样新建的协程中，从而不会阻塞信息的接收。

`tokio::spawn`返回`JoinHandle`类型，对其`.await`可以返回`Result`类型。

```Rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // Do some async work
        "return value"
    });

    // Do some other work

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```

## `'static`约束

向tokio运行时中加入的协程，其类型需要有`'static`生命周期。也就是说，它不能引用协程外的、非`'static`的对象。

```Rust
// this will not compile
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

解决方法是，使用`move`关键字，将使用的外部对象移入协程中。（这时就需要注意，不能多次移入）

## `Send`约束

向tokio运行时中加入的协程，其类型需要实现`Send` trait。

为了做到这一点，需要使协程内使用的，未实现`Send`的类型的对象，不能拥有跨越`.await`点的生命周期。

```Rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // The scope forces `rc` to drop before `.await`.
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` is no longer used. It is **not** persisted when
        // the task yields to the scheduler
        yield_now().await;
    });
}
```

注意：由于编译器使用作用域来判断变量的生命周期，以下的写法也是无法编译的

```Rust
// this will not compile
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");
        println!("{}", rc);
        drop(rc);

        yield_now().await;
    });
}
```

## 在异步代码中使用`Mutex`

无论是`std::sync::Mutex`还是`tokio::sync::Mutex`，都有一个问题：当获取不到锁时，会阻塞整个线程，导致这个线程无法切换到其它协程。

因此，为了避免死锁，应该确保协程不会跨越`.await`点持有锁。

如果使用`std::sync::Mutex`，则这点编译器会帮助我们确认，因为`MutexGuard`没有实现`Send`。

但若使用`tokio::sync::Mutex`，则`MutexGuard`实现了`Send`。但仍不推荐跨越`.await`点持有锁。

```Rust
use tokio::sync::Mutex; // note! This uses the Tokio mutex

// This compiles!
// (but restructuring the code would be better in this case)
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().await;
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
```

## 消息传递（message passing）和管道（channel）

在协程间共享资源，除了使用`Mutex`以外，还有一种方法：使用一个专门的协程管理这个资源。

之后，其它协程使用管道等机制与这个管理资源的协程进行通信。

感觉离题太远，所以看得不是很详细，详见[https://tokio.rs/tokio/tutorial/channels](https://tokio.rs/tokio/tutorial/channels)

## I/O

tokio提供两个trait，`AsyncRead`和`AsyncWrite`，以提供统一的读写接口。tokio的`TcpStream`、`File`、`Stdout`、`Vec<u8>`、`&[u8]`等都实现了这些trait。

一般不会直接调用这两个trait提供的方法，而是使用[`AsyncReadExt`](https://docs.rs/tokio/1.35.1/tokio/io/trait.AsyncReadExt.html)和[`AsyncWriteExt`](https://docs.rs/tokio/1.35.1/tokio/io/trait.AsyncWriteExt.html)提供的方法。常用的有`read()`、`read_to_end()`、`write()`、`write_all()`等。

除此之外，还有一些帮助函数，例如`tokio::io::copy()`等。

`tokio::io::split()`：将一个可读可写的类型拆分为一个读端和一个写端。

## 取消`Future`的执行

直接把对应的`Future` `drop`掉就行。

## `tokio::select!`

同时等待多个异步任务，且在其中一个异步任务完成时就返回，并取消掉其它异步任务。

详见[https://tokio.rs/tokio/tutorial/select](https://tokio.rs/tokio/tutorial/select)

## tokio stream

流（stream）为异步版本的迭代器，由`Stream` trait表示。

目前位于另一个crate，`tokio-stream`中。

可使用`while let`进行迭代：

```Rust
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let mut stream = tokio_stream::iter(&[1, 2, 3]);

    while let Some(v) = stream.next().await {
        println!("GOT = {:?}", v);
    }
}
```

对流调用`next()`需要将它`pin`起来。

`Stream` trait定义如下：

```Rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}
```

- `Stream::poll_next()`：和`Future::poll()`类似。根据调用时，值是否已准备好，返回`Poll::Ready`或者`Poll::Pending`。若返回`Poll::Pending`，需要处理好`Waker`，使得在值准备好后调用`Waker`。
- `Stream::size_hint()`：和`Iterator`的同名函数相同。

一般使用`Future`和其它`Stream`的结合来创建`Stream`。

目前，Rust语言还未提供`async/await`这样的，和创建`Future`类似的创建`Stream`的关键字。可以使用`async-stream` crate中的`stream!`宏来创建。

```Rust
use tokio_stream::Stream;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

struct Interval {
    rem: usize,
    delay: Delay,
}

impl Interval {
    fn new() -> Self {
        Self {
            rem: 3,
            delay: Delay { when: Instant::now() }
        }
    }
}

impl Stream for Interval {
    type Item = ();

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<()>>
    {
        if self.rem == 0 {
            // No more delays
            return Poll::Ready(None);
        }

        match Pin::new(&mut self.delay).poll(cx) {
            Poll::Ready(_) => {
                let when = self.delay.when + Duration::from_millis(10);
                self.delay = Delay { when };
                self.rem -= 1;
                Poll::Ready(Some(()))
            }
            Poll::Pending => Poll::Pending,
        }
    }
}
```

这段代码也某种程度上可以反映嵌套`Future`的`poll`和`Waker`的行为。`Executor` `poll`外层`Future`时，传入了外层`Future`的`Waker`。外层`Future`执行时会`poll`内层`Future`，此时传入内层`Future`的还是外层`Future`的`Waker`。内层`Future`若无法执行完成，将传入的`Waker`注册到其需要的资源处，然后返回`Pending`，外层`Future`再返回`Pending`。

当内层`Future`等待的资源准备好时，其调用`Waker`，`Executor`再次`poll`外层`Future`，外层`Future`再嵌套地`poll`内层`Future`，……

也就是说，`Executor`只会管理最外层的`Future`，内层的`Future`都是由它们的父`Future`管理的？

## 流适配器（stream adapter）

接收一个流，返回另一个流。

常用的流适配器有`map`、`take`、`filter`。

- `take`：只取流的前几个项
- `map`：对流的每一项做变换
- `filter`：对流的每一项做计算，根据结果为`true`或`false`来保留或丢弃这些项。