# AsyncOS和调度器中的多核同步问题记录

## AsyncOS中的问题

### 1. 在join的实现中，在获取另一任务的状态、将任务的Waker放入另一任务的等待队列期间，退出的任务先唤醒了等待队列中的所有任务，再有另一任务放入其等待队列中，导致放入的任务无法唤醒

修改：通过状态锁保证其它核心不会在此时修改任务状态，阻止了上述情况出现。

解决问题的版本：[async-os fc8a594](https://github.com/rosy233333/async-os/commit/fc8a594f033441bd89fe6bb7d98ab85064ee73e4)

### 2. 测试程序卡死，增加调试输出后发现所有任务在等待一个持有锁的任务，而该任务在被时钟中断后进入了Blocking

首先，修改了AsyncOS中的实现和调度器中的约定，使不仅是外部中断，而是所有中断均将任务状态改为Ready。

之后发现状态改为Ready后，问题已消除。（只是会大量拖长运行时间，因为被放回就绪队列的任务需要等待之前的大量任务全都执行一遍并被阻塞后才会继续运行。）

AsyncOS中的Mutex似乎不会在获取锁时关中断。在一般的中断处理方式中可能导致问题，但在我们的实现（中断只会将任务改为Ready并放回就绪队列）中理论上不会有问题。因为原来就是这么设计的，因此不做修改。

此外，还在测试输出中发现了进入调度器时即为Blocked状态的任务。似乎不应出现，任务要么使用Blocking状态代表阻塞，要么不设置任务状态，通过协程返回Poll::Pending代表阻塞。

解决问题的版本：[async-os 5883459](https://github.com/rosy233333/async-os/commit/5883459a6159579eda3fe96d4cee4f9b13f64e1d)

## 调度器中的问题

### 1. 调度器多次获取任务状态，而任务状态在期间改变，导致了与预期不一致的行为

修改：将x_entry中设置任务状态的步骤和xschedule中根据任务状态将任务放入就绪队列的步骤合并，放在x_entry中。

解决问题的版本：[vsched2 dd0085f](https://github.com/rosy233333/vsched2/commit/dd0085fa8f11f9d3fc98c2dd4a288814e7de7ffa)

### 2. 调度器复用线程栈，在多核环境下调度器使用该栈时线程被调度到另一个核心上运行并使用该栈，造成同步问题

修改：在多核模式下，thread_entry中会切换到空栈。单核模式下仍复用线程的栈。

解决问题的版本：[vsched2 9d2e688](https://github.com/rosy233333/vsched2/commit/9d2e688f52f20599d7796d34843c982e449b2ea6)

### 3. 之前未考虑到，需要操作各种队列的api在进入时也需要关中断（还未解决）

因为进入调度器后会获取锁，因此需要关中断。

在只有该问题时，运行很少出现bug。

目前打算暂不解决。在实现用户态调度的过程中，将调度器内部的共享数据实现为无锁，就能自然解决该问题。

### 4. 发生中断时的错误，进入中断向量时scause正确，但在处理该中断时提示scause非法

并且，在测试过程中还出现了非常难以置信的现象：在我修改了代码中的某个测试输出后，打印的结果居然一部分是修改前的格式，一部分是修改后的格式？？

![alt text](image-1.png)

![alt text](image.png)

（热升级了属于是）

出现问题的版本：[async-os 5883459](https://github.com/rosy233333/async-os/commit/5883459a6159579eda3fe96d4cee4f9b13f64e1d)、[vsched2 06a876b](https://github.com/rosy233333/vsched2/commit/06a876bf61a172a8e740eceeb1b1a95c7ae7da1d)

上述神奇的bug可以通过清理编译产物再重新编译解决。

清理后仍会报错，且报错常发生在多个核同时发生中断的时机。该错误还有另一种表现形式，是在取出任务trapframe时因没有trapframe而报错。

之后查明了原因：在调度器的中断向量中存在三个操作：set_state、push_prev_task和push_trap。原本的设计中，三个操作的顺序为set_state --> push_prev_task --> push_trap。

实际上，在异常/系统调用处理时，应沿用上述的顺序。而中断处理时，顺序应改为push_trap --> set_state --> push_prev_task。修改顺序的目的是始终保证将“执行后任务即可能运行”的操作放在最后，从而避免其它操作修改任务状态/上下文时，因为任务已经重新运行了而导致状态不正确。详见[此处注释](https://github.com/rosy233333/vsched2/blob/56be66983f977b5d19c1cdbd81befc56c1c3522f/src/main_loop.rs#L82)。

解决问题的版本：[async-os 29b45c6](https://github.com/rosy233333/async-os/commit/29b45c655ad462b126758b590f6f6ed381c776cd)、[vsched2 56be669](https://github.com/rosy233333/vsched2/commit/56be66983f977b5d19c1cdbd81befc56c1c3522f)

### 5. state=Blocking相关的同步问题

调度器从thread_entry进入后，会检测若任务状态为Blocking，则修改为Blocked。但该过程不是原子操作，因此可能其它核心唤醒该任务时，状态检测和修改在调度器检测Blocking和调度器修改Blocked之间，导致唤醒时设置的Ready被覆盖为Blocked，任务无法被唤醒。

因此，修改了任务状态相关接口，新增了match_set_state接口，原子地根据任务当前的状态，修改任务状态为参数中的对应值，并返回任务的旧状态。这样可以把上述的检测-修改过程变为原子操作，从而解决该问题。

此外，还在async-os中调整了关中断与设置任务状态的时机，保证设置任务状态为Ready/Exited/Blocking时都处于关中断环境下，避免设置任务状态后任务被中断，恢复后任务状态即重设为Running导致的问题。

解决问题的版本：[vsched2 09ce92b](https://github.com/rosy233333/vsched2/commit/09ce92b36640cd9b4989fcd2ef812e3025b4606f)、[async-os 6169ecb](https://github.com/rosy233333/async-os/commit/6169ecbd580ab321832158734ddca6b9c8898d22)

### 6. trap_handler cannot use thread api to do task switch（还未解决）

根据之前的经验，出现这样的上下文问题多是出现异常导致。的确是这样，出现了sepc=0的InstructionPageFault。

可能是因为trap_handler独特的唤醒方式导致的？

### 7. 若trap_handler在处理trap的过程中被阻塞，则当前核心上的其它trap也无法处理

修改方案见[trap_handler修改计划](../26.7.20~26.7.26/trap_handler修改计划.md)。

由xjj修改。

解决问题的版本：[vsched2 78c637c](https://github.com/rosy233333/vsched2/commit/78c637c200f29ef98a8e8f8c676f3f1f972ef9e6)
