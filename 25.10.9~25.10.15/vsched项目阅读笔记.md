# vsched项目阅读笔记

## user_test修改笔记

user_test中的测例代码因为没有适应
vsched代码的重构而无法运行。因此阅读vsched代码，从而修改user_test代码使其可以运行。

### 任务数据结构

![](./vsched任务模型.png)

`TaskInner`中，除了`ext`以外的字段为vdso可见的信息。`TaskInner`的`ext`字段为`TaskInnerExt`类型，其也在base_task库中定义。`TaskInnerExt`中有用户定义的`TaskExt`类型的`task_ext`字段。

user_test中定义的`task_ext`，字段和功能如下：

|字段|是否与base_task中字段重复|
|-|-|
|base（指向base_task的指针）|否|
|name|TaskInnerExt::name|
|entry|TaskInnerExt::entry|
|in_wait_queue|TaskInner::in_wait_queue|
|timer_ticket_id（功能见代码注释）|TaskInner::timer_ticket_id|
|preempt_disable_count|TaskInner::preempt_disable_count|
|exit_code|TaskInnerExt::exit_code|
|wait_for_exit|TaskInnerExt::wait_for_exit|
|future|TaskInnerExt::future|

|功能|是否与base_task中功能重复|
|-|-|
|定义WaitQueue及相关功能|base_task中定义了WaitQueue，但没有实现任务等待相关功能|
|映射vsched|否|
|定义gc任务和idle任务|否|
|协程调度循环、栈的分配和回收|否|
|线程和协程的退出|否|
|协程的让出和阻塞|否（需要user_test和base_task中代码配合以完成协程调度）|
|线程的task_entry定义|否|
|join操作|base_task定义了join操作，但依赖于外部实现的join函数和JOIN_FUTURE|

`user_test`中所有字段在重构过程中都移动到了`base_task`中，但其所有功能均未移动到其它模块。

### 线程与协程的调度过程
