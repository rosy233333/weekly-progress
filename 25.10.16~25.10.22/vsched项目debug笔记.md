# vsched项目debug笔记

## 修改user_test阶段

本阶段用于做最小的修改使user_test重新可运行，未对项目结构进行较大修改。

### 数据结构命名问题

修改了`user_test`的引用中，已被更名或移动的数据结构：

- `BaseTask` -> `AxTask`
- `BaseTaskRef` -> `TaskRef`

### `TaskExt`与`TaskInnerExt`

在[此commit](https://github.com/AsyncModules/vsched/commit/0351acfb128ab0399e21f09fc6d2eb506ac97fa5)中，原本位于`user_test::TaskExt`中的大部分字段都被迁移到了`base_task::TaskInnerExt`中。因此对`user_test`中的代码做了修改以适应该重构。

### so加载问题

`map_vsched`函数中原本的so加载流程是直接将so文件的内容拷贝到指定内存区域。更改为了以segment为单位拷贝到指定位置。

### 引用计数与释放问题

原本的`TaskInner`会传入`task_clone`、`task_weak_clone`、`task_drop`、`task_strong_count`函数指针以辅助进行引用计数管理，而在新版的base_task中取消了这些接口。不过，传入`TaskInner`的指针仍由`Arc`转化而来，因此需要进行引用计数管理。

分析后发现，将任务放入调度器，直至运行完成被释放的过程，固定消耗1个引用计数，与创建任务时产生的1个引用计数相抵。只有在`join`的情况下，需要增加任务的引用计数。因此实现了增减任务引用计数的`Task::clone_increase_sc`和`Task::drop_decrease_sc`函数，并在涉及`join`操作时，在合适时机调用它们。

### 协程状态切换问题

在`coroutine_schedule`函数中，有以下代码，检查`poll`之后的协程需要已经将`state`从`Running`切换到其它状态（`Ready`, `Blocked`, `Exited`）。

```Rust
assert!(
    !curr.is_running(),
    "{} is not running",
    curr.task_ext().id_name()
);
```

因此我检查了各个辅助`Future`对协程状态的维护，在`YieldFuture`中补充了缺失的状态切换代码；且在`Task::new_f`函数中，在创建的协程上做了一层包装，在调用`future.await`后还会调用`exit_f(0).await`，避免了用户创建的协程结束后没有修改运行状态导致的问题。

### 适应更改后的vsched接口

修改了一些调用参数适应更改后的vsched接口。例如，在`vsched_api::spawn`中，增加了获取并传入cpu_id的代码。

### `JUMP_SLOT`类型的重定位问题

见[此处](https://github.com/rosy233333/vdso_crate_template/blob/main/doc/%E5%BC%80%E5%8F%91%E6%97%A5%E5%BF%97/vdso%E6%A8%A1%E6%9D%BF%E9%87%8D%E6%9E%84-debug%E7%AC%94%E8%AE%B0.md#%E8%BF%90%E8%A1%8C%E6%97%B6%E9%87%8D%E5%AE%9A%E4%BD%8Dbug)。

### `PerCPU`结构的初始化问题

编译成功后，发现运行时出现了段错误。分析log文件、增加测试代码后得知，该错误由`PerCPU`结构未正确初始化引起。

首先发现，user_test自己定义了`PerCPU`，与base_task中的定义不同。于是将user_test中改为使用base_task中的`PerCPU`结构，但仍有bug。

最后发现，这是由于Makefile中定义的编译流程中，只在编译libvsched.so时传入了`RQ_CAP`和`SMP`环境变量，而没有在编译user_test时传入，导致了两者的`PerCPU`结构产生了不同。在user_test的编译流程中传入`RQ_CAP`和`SMP`环境变量，解决了该问题。

### 新增调试输出

为了更好地定位问题，在user_test中使用`env_logger`增加了对log调试输出的支持。

同时，在`vsched_api`调用`vsched`中的函数时，加入了调试输出，输出调用的函数和地址。该调试输出等级为trace。
