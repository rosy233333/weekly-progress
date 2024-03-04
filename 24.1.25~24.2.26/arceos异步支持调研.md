# ArceOS异步支持调研

时间：2024/2/23

## 对ArceOS的现有异步支持的调研

### 调研过程和结果

使用`arceos`和`arceos async`关键词分别搜索Github、Gitlab、Gitee，后两个站点基本没有相关内容（只有Gitee有一个ArceOS的镜像仓库）。

Github中查找结果如下：

[ArceOS主仓库](https://github.com/rcore-os/arceos)显示`Async I/O`功能还未完成，`dev`分支里也没有相关的文件（名字带async的文件）

找到一个相关的`Discussion`：[实验小项目招新：arceos支持tokio](https://github.com/orgs/rcore-os/discussions/21)，但未找到他们的仓库。

“实现异步I/O”作为“ArceOS 相关大实验选题”出现在了[LearningOS/os-lectures/oslabs/biglabs.md](https://github.com/LearningOS/os-lectures/blob/3ce185c74def00709e0f3558d8d3220d0715204d/oslabs/biglabs.md)，其被分类在“modules 层”。

### 之后的方案

1. 选择一个简单的executor实现，移植到ArceOS中，再修改其调度部分，使得其可以使用我们的硬件调度器。
2. 自己在ArceOS中写一个最基础的executor。

## 对学长和我的项目中，异步支持的设计的回顾

### 调研结果

异步任务的单位为`Task`，通过`Arc`分配在栈上，当存放在硬件调度器中时，转化为`TaskRef`形式。

```Rust
#[repr(C)]
pub struct Task {
    /// The task state field
    pub state: AtomicU32,
    /// The priority of task
    pub priority: AtomicU32,
    /// The task type field:
    ///     1. Normal
    ///     2. AsyncSyscall
    ///     3. ......
    pub task_type: TaskType,
    /// The actual content of a task.
    /// It may be a `process`, `thread` or `coroutine`.
    pub fut: AtomicCell<Pin<Box<dyn Future<Output=isize> + 'static + Send + Sync>>>,
}

#[repr(transparent)]
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct TaskRef {
    ptr: NonNull<Task>,
}
```

（来自[design-v6](../../学长发的文档/openingreport-zfliang2023/design-v6.md)）

每个进程使用一个调度器，调度其内部的协程。

进程和协程均用`Task`表示，其区别我也不太记得了。我大致地回忆如下：进程也是异步的`Future`，是启动一个程序时运行的第一个`Future`。启动它时，与启动其它协程相比，还需分配调度器、运行栈、地址空间等。

主CPU核与其它CPU核、系统内核与用户程序、用户程序的用户态与内核态，这三组关系：仍不清楚。

### 之后方案

去看ArceOS代码，了解其任务运行和调度机制。

### 问题

为了实现该设计，是不是需要对ArceOS的任务运行和调度机制进行大改？如果需要，这个任务量与难度是否在我们的控制范围内？

## 与学长讨论的结果

ArceOS是一个unikernel系统，单地址空间，不区分用户态和内核态。

[Unikernel: 从不入门到入门](https://zhuanlan.zhihu.com/p/29053035)

因此，用类似于写用户程序的方法写一个Executor即可，不需要修改ArceOS原有的代码。

确定这个方案后，预估的任务量较大地下降了。