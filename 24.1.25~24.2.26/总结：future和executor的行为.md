# 总结：`Future`、`Executor`和`Reactor`的行为

时间：2024/2/19

## `Future`、`Executor`和`Reactor`

`Future`为Rust语言提供的数据结构，表示异步任务，或者说“未来才会需要和计算的值”。*（一些`Executor`会对`Future`进行进一步的包装，此处不考虑，并认为异步任务与`Future`等同）*。

`Executor`，连同配套的`Waker`、`Spawner`等，是`Future`的运行时环境。由它们支持`Future`的运行、唤醒和调度。

`Reactor`是`Future`和I/O硬件等异步资源之间的接口，接收`Future`的任务，提交给对应的异步资源。**`Reactor`不是`Future`，不需要`poll`，可以自主运行**。每个`Reactor`与一种I/O硬件或一种异步功能对应。多线程情况下，不同的`Future`访问`Reactor`需要保证互斥。

## `Future`的嵌套和分类

`Future`中可以持有其它`Future`对象，并通过`.await`执行它们。因此，`Future`可以嵌套并形成树形结构。树中有三种`Future`：

- 根`Future`：最顶层的`Future`，由`Executor`直接管理和执行。其与`Executor`的关系详见 **根`Future`和`Executor`** 章节。
- 叶`Future`：最底层的`Future`，不会再`.await`其它`Future`，而是直接和`Reactor`通信。其与`Reactor`的关系详见 **叶`Future`和`Reactor`** 章节。
- 不为根也不为叶的`Future`（之后称为枝`Future`）：它由上级的`Future`通过`.await`调用，同时也会`.await`下级的`Future`。

## 根`Future`和`Executor`

**`Executor`管理和调度的单位为根`Future`。枝`Future`和叶`Future`对`Executor`都是不可见的。**

在此前提下，`Executor`和根`Future`交互的行为如下：

### 1. `Future`的提交

通过`Spawner`等途径，可以向`Executor`中提交`Future`。这些`Future`即是根`Future`，会进入`Executor`的就绪队列。

### 2. `Executor`的工作

当`Executor`开始运行，它会不断地根据某种调度算法，从就绪队列中取出根`Future`，并且调用它们的`poll`方法。由此，`Executor`所在线程执行的任务由`Executor`变为根`Future`。

**同时，`Executor`会为每个根`Future`分配一个`Waker`，并在调用`poll`的时候传入根`Future`。调用这个`Waker`，会将对应的根`Future`重新加入就绪队列。**

### 3. `Future`的运行

由`async/await`创建的根`Future`执行一段时间后，返回`Pending`或`Ready`并退出，由此，对应线程的执行的任务由`Future`又回到了`Executor`，`Executor`回到执行第2步的内容。

**由于此时，该根`Future`已经被取出就绪队列，因此，若后续没有调用该根`Future`的`Waker`，它就不会再次执行。**

## 根`Future`、枝`Future`和`async/await`

我们假定根`Future`和枝`Future`均由`async/await`关键字创建：

- `async`将函数/闭包/语句块转化为`Future`。
- 外层`Future`通过`.await`执行内层的`Future`，其实质为：`poll`一次内层`Future`，若返回`Ready`则拿到结果并继续执行外层`Future`；若返回`Pending`则外层`Future`也立即返回`Pending`。**注意：这次`poll`传入内层`Future`的`Waker`，就是外层`Future`被`poll`时从中获得的`Waker`。**

通过`async/await`创建的`Future`，其要么在终止点返回`Ready`，要么在`.await`点返回`Pending`。

## 叶`Future`和`Reactor`

叶`Future`不会`.await`其它`Future`，而是调用`Reactor`实现最基本的异步功能，例如计时或者控制I/O硬件。

`Reactor`有一个任务表，其中的每个任务都记录了执行状态（完成情况、结果），**且每个任务带有一个`Waker`**。`Reactor`调用其管理的资源（例如I/O硬件），不断接收`Future`提交的任务，将其加入任务表并完成。**当任务完成时，`Reactor`会调用它对应的`Waker`。**

对叶`Future`调用`poll`时，它会检查对应的`Reactor`中，自己任务的状态，采取对应的动作：

- 若自己的任务已经完成（可以在`Reactor`的任务表中找到，状态为完成），则返回`Ready`和执行结果。
- 如果自己的任务已提交未完成（可以在`Reactor`的任务表中找到，状态为未完成），则直接返回`Pending`。
- 若自己的任务还未提交（无法在`Reactor`的任务表中找到），则向`Reactor`提交一个任务，**同时传入自己通过`poll`获得的`Waker`**，然后，返回`Pending`。

## `Future`的嵌套执行与`Waker`的传递

综上所述，嵌套`Future`的执行过程和`Waker`的传递过程如下：

1. 根`Future`被`Executor` `poll`时，它获得了`Executor`为它分配的，与它自身对应的`Waker`。
2. 随着根`Future`使用`.await`逐级调用下层`Future`，逐层向下传递的`Waker`也是与根`Future`对应的`Waker`。
3. `poll`到叶`Future`时（*假定是第一次`poll`，即，叶`Future`还未创建任务*），它在`Reactor`处创建任务，同时将根`Future`对应的`Waker`传入`Reactor`。
4. 叶`Future`返回`Pending`。由于`.await`的行为，其所有上级`Future`，包括根`Future`，都立即返回`Pending`。
5. `Executor`接收到根`Future`返回的`Pending`结果，转而调度下一个根`Future`。
6. 和**4、5**同时，`Reactor`执行任务，完成后，调用根`Future`对应的`Waker`，将根`Future`重新加入`Executor`的就绪队列。