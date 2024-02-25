# embassy-executor笔记

时间：2024/2/19

源文档：[https://docs.embassy.dev/embassy-executor/git/cortex-m/index.html](https://docs.embassy.dev/embassy-executor/git/cortex-m/index.html)

## `Spawner`和`SendSpawner`

用于将异步任务提交到`Executor`执行。

`Spawner`只能在`Executor`所在的线程使用，可以提交没有实现`Send` trait的任务。

`SendSpawner`可以在任意线程使用，只能提交实现了`Send` trait的任务。

通过`SpawnToken`来标识任务。`SpawnToken`可以通过调用被标注了`#[embassy_executor::task]`宏的任务获得。