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

### 3. 新增核间中断发送接口后，在AsyncOS上的额外适配

首先遇到了检测到上下文被覆盖的报错。发现原因是因为核心在休眠后被中断时，保存到SCHEDULER_WAIT_CONTEXT上下文，而这个上下文后续只会取出栈，不会取出寄存器上下文。因此修改了检测方式，只有检测到原有的上下文中存在栈时，才会报错上下文被覆盖。

之后，发现AsyncOS无法处理核间中断而panic。因此修改了AsyncOS中中断的分发函数，在核间中断（即软件中断）到来时不做任何操作。

解决问题的版本：[async-os 737d198](https://github.com/rosy233333/async-os/commit/737d1981759d4ef8bea5c518067ea629e2103ab4)

### 11. 在Waker中检查data为空的问题

在中断、多核、非线程api模式下出现，在prepare_to_wait函数中因为检查到waker的data为空而panic。

此外，该配置下还会出现卡死问题。先解决上一问题，之后看卡死问题是否还会出现。

在线程api模式下，出现了0xffffffc0808398c0 tried to release mutex it doesn't own, which belong to 0xffffffc080842140的问题。

通过AI辅助，定位了卡死问题的原因：在条件阻塞/唤醒中，A检测条件 -> B改变条件 -> B唤醒 -> A阻塞的时序导致了A的唤醒被丢失。

因此，在[流程图](../26.7.13~26.7.19/任务及其状态变化流程图.png)中考虑了条件阻塞/唤醒的情况，重新设计了避免丢失唤醒的时序。解决了卡死和data为空的问题。

线程模式下“tried to release mutex it doesn't own”的问题仍存在。

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

可能是因为trap_handler独特的唤醒方式导致的？

### 7. 若trap_handler在处理trap的过程中被阻塞，则当前核心上的其它trap也无法处理

修改方案见[trap_handler修改计划](../26.7.20~26.7.26/trap_handler修改计划.md)。

由xjj修改。

解决问题的版本：[vsched2 78c637c](https://github.com/rosy233333/vsched2/commit/78c637c200f29ef98a8e8f8c676f3f1f972ef9e6)

### 8. 栈复用导致的问题

1. InstructionPageFault、LoadPageFault、StorePageFault
   - InstructionPageFault只能由非硬编码的跳转引起，例如栈数据被破坏后的return，以及上下文数据被破坏后的restore。
   - 发现在出现InstructionPageFault时，有可能fp也被破坏。
   - 发现栈的使用上似乎有问题：某个栈原本作为调度器栈（游离），但后续变成了线程栈（与任务绑定），没有经过协程栈（与任务关联）的过渡
   - 其中的一次LoadPageFault，是从waker中解析到的指针为0导致的。因此已经增加了对waker的检查。
2. 在调度器中，到达了unreachable的代码（main_loop:819），但相同核心在这之后继续运行？
   - 这个unreachable的代码理论上在之前就会被其它panic拦截，但此处没有触发这些拦截的panic，可能是因为是乱跳转跳过来的。
3. 仍有cannot use thread api to do task switch，在新建trap_handler后出现
   - 这类问题是在已有线程上下文的情况下保存线程上下文导致的。线程上下文的保存只会在两个地方发生，其一为中断时，其二为线程切换时。如果在中断时发生，则推测原因为将要运行任务，已经设置了current_task，还未恢复上下文时被中断，或者任务已经保存了上下文，还未让出时被中断；若在线程切换时发生，则推测原因为同一任务同时在两个核心上运行。推测为原因1的第二种情况或原因2，因为在运行任务时已经做了关中断的检查，且这些问题只有在多核下才会发生。
  
发现一个代码中的问题：trap_handler在让出时没有正确处理设置任务状态和关中断的关系。已经在设置任务状态前和让出后分别关/开中断。

解决上述问题后，仍发生LoadPageFault、StorePageFault和InstructionPageFault，还有IllegalInstruction（pc跳到了未对齐的代码区域导致）。

- 一次StorePageFault是因为在raw_run_task中的jalr跳转到了错误的位置。
- 推测这些问题可能具有共同的原因：错误的跳转。
- 发现了原因之一：0xdeadbeef在riscv中不是非法指令，而是`jal t4,-150038`，导致乱跳。在发现此原因的例子中，从kschedule返回后，返回值没有符合两个跳转分支，因此进入了原本用于拦截的0xdeadbeef。
- 分析了一个异常的原因，应该是栈上的数据被覆盖，导致ret返回到错误的地址。
- 目前，通过编写检测代码，没有检测到一个栈同时被两个核心使用的情况。
- 那么，是否是因为使用了被释放的栈/任务？将栈和任务的释放语句注释掉后，问题依然出现。因此应该不是。
- 是否和线程上下文的恢复有关？检查了在是否开关中断、是否使用线程API下的运行情况：若开中断，就会频繁地出现异常（基本不可能将测试正常跑完）；若关中断，则不会出现异常，只会偶尔出现其它问题（如一直运行不结束）。因此，确定该问题与中断强相关。此外，在单核模式下，开中断也不会出现异常。
- 在中断上下文和线程上下文恢复时尝试检测上下文以拦截异常，未能拦截。因此，问题出在中断处理流程中？
- 每次出现异常，sepc和ra都是一样的。因此，应该是运行过程中return到了错误的地址，所以在ret回去后就立刻触发异常了。

LoadPageFault等Fault出现的原因已确认：是调度器回收当前栈后仍在使用，而被回收的栈又被其它核心复用，导致的栈争用。将空闲栈池改为per-cpu后，就解决了该问题。

### 9. 与更新后的trap_handler相关的问题

在解决问题8后，首先出现了调度器不断创建和运行trap_handler，导致正常任务无法运行的情况。

推测原因为从接收到中断到进入trap_handler，中间打开了中断，导致再次触发中断的递归情况。因此，通过修改任务的创建，保证从接收中断到进入trap_handler不会打开中断。从而解决了该问题。

之后，bug的现象变成了创建一个trap_handler后卡死的问题。

推测原因为：trap_wait_queue在取出handler后，若handler为Blocking状态，则返回空指针，从而使当前核心以为已经没有任务而休眠。因此修改了trap_wait_queue的行为，若handler为Blocking状态，则忙等到状态变为Blocked，再设置该handler的状态为Ready并运行。从而解决了该问题。

之后，bug的现象变成了不创建trap_handler就卡死。

卡死体现在：增加调试输出后，发现所有核心都在休眠并唤醒。但即使把实际的休眠（wfi）代码注释掉，仍然发现所有核心在忙等循环。因此，main任务由于某种原因未能运行。

在AI辅助下分析，发现了卡死时各核心频繁地睡眠和唤醒的问题。但该问题背后的原因AI没分析出来，因此我这么分析：

- 频繁地睡眠 <-- 从选定的事件源中未能取出任务 <-- 1. trap_wait_queue; 2. scheduler
- 频繁地唤醒 <-- 频繁地push_task <-- 1. 任务被唤醒; 2. 任务被中断（可能就是ipi）并放回

最后决定，暂时禁用就绪队列放入任务后唤醒休眠核心的代码。一方面，如果唤醒不及时，在核心已经被唤醒后继续发送IPI，就会将核心上的任务放入就绪队列，并再次触发一个发往其它核心的IPI，从而造成中断风暴。另一方面，主动唤醒核心也不是必要的。休眠的核心在收到时钟中断时也会唤醒。

由此，解决了该问题。
