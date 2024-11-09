# Embassy

时间：2023/12/10

源资料：[https://embassy.dev/book/dev/getting_started.html](https://embassy.dev/book/dev/getting_started.html)

Embassy是一个帮助你使用异步协程进行嵌入式开发，并且提供了对一些嵌入式外设的简单接口的Rust库。其可以方便我们用Rust进行嵌入式开发。

## 使用协程调用外设——协程和中断机制的配合

具体见[Executor/interrupts](https://embassy.dev/book/#_interrupts)

![](..\图片\embassy_irq.png)

## Embassy的分层结构

具体见[From bare metal to async Rust](https://embassy.dev/book/dev/layer_by_layer.html)

### PAC（Peripheral Access Crate）层

Rust接口，提供直接通过管脚操作外设的功能。

### HAL层

Rust接口，提供封装程度更高地，更简单地操作外设的功能。

### 中断层

可以控制硬件的中断。

或许可以使用这个功能，指定中断的行为？

### Rust异步层

使用Rust异步操作（例如`await`）封装了中断过程，使开发更简单。

例如：在如下代码中：

```Rust
button.wait_for_any_edge().await;
```

程序会为该IO事件设置中断处理函数。当执行到这一行时，系统进入低功耗模式等待（前提是没有别的协程可以执行）。当IO事件到来时，触发中断，再唤醒这个协程继续执行。