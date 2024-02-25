# Rust异步原理

时间：2023/12/6

源资料：[Async/Await | Writing an OS in Rust](https://os.phil-opp.com/async-await/#futures-in-rust)

## `Future`

### `Future` trait 和 `Poll` enum

代表一个通过异步方法获得的值，其可能已准备好，也可能还未准备好。

通过`poll`方法来推进该值的获取

```Rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

### future combinators

将一种类型的`Feature`转化为另一种类型的`Feature`。

如下，将`Future<Output = String>`转化为`Future<Output = usize>`。
```Rust
struct StringLen<F> {
    inner_future: F,
}

impl<F> Future for StringLen<F> where F: Future<Output = String> {
    type Output = usize;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        match self.inner_future.poll(cx) {
            Poll::Ready(s) => Poll::Ready(s.len()),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

## `async` 和 `await`

直接使用future combinators编程很复杂，所以加入了简化编程的`async`/`await`关键字。

```Rust
async fn example(min_len: usize) -> String {
    let content = async_read_file("foo.txt").await;
    if content.len() < min_len {
        content + &async_read_file("bar.txt").await
    } else {
        content
    }
}
```

编译器自动将代码转化为状态机

使用`async`声明的函数或者代码块，不会返回值本身，而会返回`Future`。

对`Future`调用`.await`，获取其值。

注意对`await`和流程控制（`if`）的处理

![](..\图片\async-state-machine-basic.svg)

为状态生成结构体，来保存执行过程中的信息

```Rust
enum ExampleStateMachine {
    Start(StartState),
    WaitingOnFooTxt(WaitingOnFooTxtState),
    WaitingOnBarTxt(WaitingOnBarTxtState),
    End(EndState),
}

struct StartState {
    min_len: usize,
}

struct WaitingOnFooTxtState {
    min_len: usize,
    foo_txt_future: impl Future<Output = String>,
}

struct WaitingOnBarTxtState {
    content: String,
    bar_txt_future: impl Future<Output = String>,
}

struct EndState {}
```

将`async`函数变为`Future`

```Rust
impl Future for ExampleStateMachine {
    type Output = String; // return type of `example`

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        loop {
            match self { // TODO: handle pinning
                ExampleStateMachine::Start(state) => {…}
                ExampleStateMachine::WaitingOnFooTxt(state) => {…}
                ExampleStateMachine::WaitingOnBarTxt(state) => {…}
                ExampleStateMachine::End(state) => {…}
            }
        }
    }
}

// 原本的函数
async fn example(min_len: usize) -> String

// 转化后的函数
fn example(min_len: usize) -> ExampleStateMachine {
    ExampleStateMachine::Start(StartState {
        min_len,
    })
}
```

## Pinning

之前展示的代码没有处理Pinning，因此只能作为伪代码，无法运行。

当函数内的变量涉及到引用时，状态储存会变得复杂。

```Rust
async fn pin_example() -> i32 {
    let array = [1, 2, 3];
    let element = &array[2];
    async_write_file("foo.txt", element.to_string()).await;
    *element
}

struct WaitingOnWriteState {
    array: [1, 2, 3],
    element: 0x1001c, // address of the last array element
}
```

结构体引用了自身成员的地址。

导致，如果结构体在内存中移动，则对自身成员的引用失效。

因此，Rust选择将这类结构体固定在内存中（禁止其移动）。

### `Unpin` trait

这是一个auto trait，会为所有类型自动实现。代表该类型可以在内存中移动。

通过取消`Unpin`的实现，我们可以固定一个类型。（同时也会使其无法进行可变引用，因为某些对可变引用的操作，例如`mem::replace`，可以移动这个类型。）

### 堆上的固定数据：`Pin<Box<T>>`

为了取消`Unpin`的实现，需要满足：

1. 在结构体中使用`PhantomPinned`。
2. 将其放置在堆中
3. 使用`*::pin`来初始化。

```Rust
use core::marker::PhantomPinned;

struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned,
}

let mut heap_value: Pin<Box<SelfReferential>> = Box::pin(SelfReferential {
    self_ptr: 0 as *const _,
    _pin: PhantomPinned,
});
```

这样，创建的数据类型即为Pin<Box<T>>。

如果需要修改它的内容，需要使用unsafe的`get_unchecked_mut`方法。

`get_unchecked_mut`接收`Pin<&mut T>`参数，因此需要先写`as_mut`那句。

```Rust
// safe because modifying a field doesn't move the whole struct
unsafe {
    let mut_ref = Pin::as_mut(&mut heap_value);
    Pin::get_unchecked_mut(mut_ref).self_ptr = ptr;
}
```

### 栈上的固定数据：`Pin<&mut T>`

不推荐直接创建。

最好还是创建`Pin<Box<T>>`，需要时再转化为`Pin<&mut T>`

### `Pin`与`Future`

`Future::poll`方法中，使用`Pin<&mut Self>`类型的`self`。

在第一次`poll`以前，`Future`内只存储函数的输入参数，此时可以移动。

但调用`poll`方法之后，`Future`的状态存储可能包含自引用，因此就不能移动了。

通过对`poll`方法的声明，`Future`可以自由地创建和移动，但需要在调用第一次`poll`之前，放入一个`Pin`中。

## 协程调度器executor

将所有协程集中于一处，控制和调度协程的运行。可以通过线程池来高效利用多核CPU资源。

### waker

waker是一个由executor实现的特性。

某个协程等待其它协程的结果，这一现象十分常见。

`Executor`在调用`Future`的`poll`方法时，将自己的`waker`传入。如果协程在等待某些异步任务而返回`Pending`，`Waker`会注入这些异步任务中。一旦任务完成，其就会通过`Waker`通知`Executor`，使`Executor`知道再次poll这个协程可以成功推进。从而减少了主动poll的次数。

