# trap_handler修改计划

修改在`vsched`调度器中进行。

## 原有实现

每个CPU绑定一个trap_handler，在调度器取出任务时，如果trap队列中有未处理的trap，则取出该trap_handler并运行。

该设计导致若trap_handler在运行过程中阻塞在某个内核资源上，则当前核心的其它trap都无法及时处理（即使再次取出了trap_handler，也会在获取到资源前继续阻塞。）

## 修改后的实现

每个CPU从绑定一个trap_handler到绑定一个阻塞队列。

trap_handler如果处理完了所有的trap，则阻塞在trap_wait_queue的当前cpu的阻塞队列上（在调度器内部代码中实现）。如果在处理trap的过程中等待内核资源，则阻塞在那个内核资源对应的阻塞队列中（在TrapInfo::handle接口中，由OS实现）。

调度器取出任务（take_task方法）时，如果trap队列中有未处理的trap，则首先从trap_wait_queue的当前CPU的阻塞队列中取出trap_handler；如果阻塞队列中没有任务，则创建一个trap_handler。

是否需要保证，阻塞在内核资源上的trap_handler在唤醒后仍运行在相同的CPU上？若需要保证，则目前暂时没有很好的方式保证（无法使用任务的CPU亲和字段，因为目前的调度器不支持CPU亲和）。若不需保证，则可能带来更复杂的同步问题。

### 同步问题分析

多个trap_handler对当前CPU的trap队列的访问可以通过加锁来同步。

“访问trap队列”过程和“阻塞”过程之间不会有同步问题。

对于trap_handler阻塞在当前cpu的阻塞队列的过程，参考流程图中的“线程式阻塞”流程，也就是在任务内部设置Blocking状态，在进入调度器后再改为Blocked。

对于从trap_wait_queue的阻塞队列中取出trap_handler的过程，参考流程图中的“线程/协程唤醒”流程。只是修改任务状态（Blocked-->Ready）后，不需放入就绪队列，而是直接把任务传出`take_task`函数中；如果修改任务状态为（Blocking-->Ready），则被唤醒的trap_handler将由调度器继续运行，本次`take_task`返回None即可。

![流程图](../26.7.13~26.7.19/任务及其状态变化流程图.png)

## 修改清单

- 在调度器中增加阻塞队列的支持。
- trap_wait_queue的数据结构修改：从每个CPU从绑定一个trap_handler到绑定一个阻塞队列。
- trap_wait_queue的take_task方法修改：如果trap队列中有未处理的trap，则首先从trap_wait_queue的当前CPU的阻塞队列中取出trap_handler；如果阻塞队列中没有任务，则创建一个trap_handler。
- trap_handler修改：trap处理完成后，增加将自己阻塞在对应CPU的阻塞队列上的代码。

在修改过程中，应该可以保证不动interface和api？
