# Rust异步原理中的executor实现

时间：2024/1/28

源资料：[Async/Await | Writing an OS in Rust](https://os.phil-opp.com/async-await/#futures-in-rust)，Implementation章节

## Task

`Task`是对`Future`的包装，是`Executor`调度的单位。其要求`Future`返回值均为`()`。

`poll`方法由`Executor`内部调用，不暴露给用户，因此没有声明`pub`。

```Rust
// in src/lib.rs

pub mod task;
```

```Rust
// in src/task/mod.rs

use core::{future::Future, pin::Pin};
use alloc::boxed::Box;
use core::task::{Context, Poll};

pub struct Task {
    future: Pin<Box<dyn Future<Output = ()>>>,
}

impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            future: Box::pin(future),
        }
    }

    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        self.future.as_mut().poll(context)
    }
}
```

## Dummy Waker

`Dummy Waker`是`Waker`的一种简单实现，目前没有任何实际功能。

先创建一个`RawWaker`，再通过`Waker::from_raw`将其转化为`Waker`。

```Rust
// in src/task/simple_executor.rs

use core::task::{Waker, RawWaker};

fn dummy_raw_waker() -> RawWaker {
    todo!();
}

fn dummy_waker() -> Waker {
    unsafe { Waker::from_raw(dummy_raw_waker()) }
}
```

`RawWaker`包含一个显式定义的虚函数表。表中的函数会收到一个`*const ()`类型的参数（用于指向任意类型的数据）。`RawWaker`有`data`和`vtable`两个成员，后者为虚函数表，前者为传入表中函数的参数。

`vtable`为`RawWakerVTable`类型，存储了`clone`、`wake`、`wake_by_ref`和`drop`四个函数。

- `clone`：当`RawWaker`被拷贝时（例如，当`Waker`被拷贝时），调用该函数。处理内容的复制，并且保证复制后的`RawWaker`也可以唤醒原`RawWaker`对应的`Executor`。
- `wake`：当`Waker`被`wake`时，调用该函数。它需要唤醒该`Waker`关联的任务。
- `wake_by_ref`：当`Waker`被`wake_by_ref`时，调用该函数。与`wake`相同，但它不能消耗`data`指针。
- `drop`：当`Waker`被释放时，调用该函数。它需要释放`RawWaker`和关联的任务的资源。

```Rust
impl RawWakerVTable {
    pub const fn new(
        clone: unsafe fn(_: *const ()) -> RawWaker,
        wake: unsafe fn(_: *const ()),
        wake_by_ref: unsafe fn(_: *const ()),
        drop: unsafe fn(_: *const ())
    ) -> Self
}
```

可以如此创建没有实际功能的`dummy_raw_waker`：

```Rust
// in src/task/simple_executor.rs

use core::task::RawWakerVTable;

fn dummy_raw_waker() -> RawWaker {
    fn no_op(_: *const ()) {}
    fn clone(_: *const ()) -> RawWaker {
        dummy_raw_waker()
    }

    let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
    RawWaker::new(0 as *const (), vtable)
}
```

## Simple Executor

`SimpleExecutor`是一个简单的`Executor`实现，可以通过`spawn`方法来向其添加`Task`，使其调度、执行它们。

```Rust
// in src/task/mod.rs

pub mod simple_executor;
```

```Rust
// in src/task/simple_executor.rs

use super::Task;
use alloc::collections::VecDeque;

pub struct SimpleExecutor {
    task_queue: VecDeque<Task>,
}

impl SimpleExecutor {
    pub fn new() -> SimpleExecutor {
        SimpleExecutor {
            task_queue: VecDeque::new(),
        }
    }

    pub fn spawn(&mut self, task: Task) {
        self.task_queue.push_back(task)
    }
}
```

结合之前创建的`dummy_waker`，可以为`SimpleExecutor`实现`run`方法，不断`poll`就绪队列中的`Task`。

```Rust
// in src/task/simple_executor.rs

use core::task::{Context, Poll};

impl SimpleExecutor {
    pub fn run(&mut self) {
        while let Some(mut task) = self.task_queue.pop_front() {
            let waker = dummy_waker();
            let mut context = Context::from_waker(&waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {} // task done
                Poll::Pending => self.task_queue.push_back(task),
            }
        }
    }
}
```

## 测试

```Rust
// in src/main.rs

use blog_os::task::{Task, simple_executor::SimpleExecutor};

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialization routines, including `init_heap`

    let mut executor = SimpleExecutor::new();
    executor.spawn(Task::new(example_task()));
    executor.run();

    // […] test_main, "it did not crash" message, hlt_loop
}


// Below is the example_task function again so that you don't have to scroll up

async fn async_number() -> u32 {
    42
}

async fn example_task() {
    let number = async_number().await;
    println!("async number: {}", number);
}
```

## 异步键盘输入

将中断处理程序和真正执行中断处理的功能代码分离，从而提高中断响应速度。中断功能代码可以放入`Executor`中执行。

无锁并发队列：可以使用`crossbeam` crate中的`ArrayQueue`，它支持`no-std`。

## 有实际功能的Waker

为`Task`实现`id`字段：

```Rust
// in src/task/mod.rs
use core::sync::atomic::{AtomicU64, Ordering};

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
struct TaskId(u64);

impl TaskId {
    fn new() -> Self {
        static NEXT_ID: AtomicU64 = AtomicU64::new(0);
        TaskId(NEXT_ID.fetch_add(1, Ordering::Relaxed))
    }
}
```

```Rust
// in src/task/mod.rs

pub struct Task {
    id: TaskId, // new
    future: Pin<Box<dyn Future<Output = ()>>>,
}

impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            id: TaskId::new(), // new
            future: Box::pin(future),
        }
    }
}
```

`TaskId`的`NEXT_ID`为`new`函数的静态变量，多次poll`new`只会初始化一次。使用`AtomicU64`类型及其方法，以实现并发安全。

`new`方法可以实现id的自动分配。

实现新的`Executor`：

```Rust
// in src/task/mod.rs

pub mod executor;
```

```Rust
// in src/task/executor.rs

use super::{Task, TaskId};
use alloc::{collections::BTreeMap, sync::Arc};
use core::task::Waker;
use crossbeam_queue::ArrayQueue;

pub struct Executor {
    tasks: BTreeMap<TaskId, Task>,
    task_queue: Arc<ArrayQueue<TaskId>>,
    waker_cache: BTreeMap<TaskId, Waker>,
}

impl Executor {
    pub fn new() -> Self {
        Executor {
            tasks: BTreeMap::new(),
            task_queue: Arc::new(ArrayQueue::new(100)),
            waker_cache: BTreeMap::new(),
        }
    }

    pub fn spawn(&mut self, task: Task) {
        let task_id = task.id;
        if self.tasks.insert(task.id, task).is_some() {
            panic!("task with same ID already in tasks");
        }
        self.task_queue.push(task_id).expect("queue full");
    }

    fn run_ready_tasks(&mut self) {
        // destructure `self` to avoid borrow checker errors
        let Self {
            tasks,
            task_queue,
            waker_cache,
        } = self;

        while let Ok(task_id) = task_queue.pop() {
            let task = match tasks.get_mut(&task_id) {
                Some(task) => task,
                None => continue, // task no longer exists
            };
            let waker = waker_cache
                .entry(task_id)
                .or_insert_with(|| TaskWaker::new(task_id, task_queue.clone()));
            let mut context = Context::from_waker(waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {
                    // task done -> remove it and its cached waker
                    tasks.remove(&task_id);
                    waker_cache.remove(&task_id);
                }
                Poll::Pending => {}
            }
        }
    }
}
```

几个看点：

- `task_queue`内存储`TaskId`，再根据`tasks`映射找出实际`Task`。
- 对`task_queue`使用`Arc`包装，因为它需要在`Executor`和`Waker`间共享。
- `waker_cache`保存了`TaskId`到`Waker`的映射。其可以保证对同一个`Task`的多次调用复用同一个`Waker`，且让`Waker`不会提前释放。
- `run_ready_tasks`函数开头，将`Executor`解构为三个独立变量，以通过借用检查。由于`run_ready_tasks`运行完成后就不再需要`Executor`，因此可以这么操作。

如此设计一个有实际作用的`Waker`：

```Rust
// in src/task/executor.rs

struct TaskWaker {
    task_id: TaskId,
    task_queue: Arc<ArrayQueue<TaskId>>,
}

impl TaskWaker {
    fn wake_task(&self) {
        self.task_queue.push(self.task_id).expect("task_queue full");
    }
}
```

使用`Wake` Trait来兼容不同的`Waker`：

```Rust
// in src/task/executor.rs

use alloc::task::Wake;

impl Wake for TaskWaker {
    fn wake(self: Arc<Self>) {
        self.wake_task();
    }

    fn wake_by_ref(self: &Arc<Self>) {
        self.wake_task();
    }
}
```

因为`Waker`会频繁地被共享，因此`Wake` Trait中的`self`需要`Arc`包装。

使用如下的代码，根据`TaskWaker`创建`Waker`：

```Rust
// in src/task/executor.rs

impl TaskWaker {
    fn new(task_id: TaskId, task_queue: Arc<ArrayQueue<TaskId>>) -> Waker {
        Waker::from(Arc::new(TaskWaker {
            task_id,
            task_queue,
        }))
    }
}
```

`Waker::from`可以从任何`Arc`包装的对象创建`Waker`。

最后，为`Executor`添加`run`方法，其会在空闲时halt。

```Rust
// in src/task/executor.rs

impl Executor {
    pub fn run(&mut self) -> ! {
        loop {
            self.run_ready_tasks();
            self.sleep_if_idle();   // new
        }
    }

    fn sleep_if_idle(&self) {
        if self.task_queue.is_empty() {
            x86_64::instructions::hlt();
        }
    }
}
```

不过还不知道怎么恢复运行，可能是通过中断？

## 扩展方向

- 支持其它任务调度算法
- 目前的`Executor`在调用`run`后就不再可用，可以创建额外的`Spawner`类型来解决这个问题。
- 利用线程：在不同的线程中运行多个`Executor`实例
- 任务平衡：在多个`Executor`间进行任务分配，例如任务窃取（work stealing）