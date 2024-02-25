Rust RFC book关于async/await部分的阅读笔记

时间：2024/2/18

源资料：[https://rust-lang.github.io/rfcs/2394-async_await.html](https://rust-lang.github.io/rfcs/2394-async_await.html)

## `async`的闭包和语句块

`async`关键字可用于函数、闭包和语句块。

`async`声明的语句块，等价于`async`声明无参数闭包后立即调用它。

```Rust
async { /* body */ }

// is equivalent to

(async || { /* body */ })()
```

## `await!`编译器内置宏（compiler built-in）

`await!`接收一个`Future`，将其不断地`poll`。若`Future`返回`Pending`则让出控制权（yield control）（此时可以将控制权交还给`Future`的调用者），返回`Ready`则将结果作为宏的返回值。

```Rust
// future: impl Future<Output = usize>
let n = await!(future);
```

`await!`只能在`async`代码中使用。

对`await!()`的展开类似下方代码：

```Rust
let mut future = IntoFuture::into_future($expression);
let mut pin = unsafe { Pin::new_unchecked(&mut future) };
loop {
    match Future::poll(Pin::borrow(&mut pin), &mut ctx) {
          Poll::Ready(item) => break item,
          Poll::Pending     => yield,
    }
}
```

此处提到的`await!`可能就是`await`关键字的前身。

## `async`返回的`Future`的生命周期

`Future`会捕获输入参数的生命周期。

## 带有初始化的`Future`

有些初始化步骤需要在`Future`创建时进行。包含这样初始化代码的`Future`可以这样创建：

```Rust
// only arg1's lifetime is captured in the returned future
fn foo<'a>(arg1: &'a str, arg2: &str) -> impl Future<Output = usize> + 'a {
    // do some initialization using arg2

    // closure which is evaluated immediately
    async move {
         // asynchronous portion of the function
    }
}
```