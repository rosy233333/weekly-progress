# Parallel, Hardware-Supported Interrupt Handling论文阅读

时间：2024/1/18

原文：[https://dl.acm.org/doi/10.1145/1629395.1629419](https://dl.acm.org/doi/10.1145/1629395.1629419)

## ABSTRACT

一个问题：高优先级的任务会被低优先级的中断处理程序抢占。称为单调速率优先级倒置（rate-monotonic priority inversion）

    单调速率调度（RM）：是一种实时系统的调度策略。这个策略要求预知任务的周期，按照任务的速率（周期的倒数）调度，优先调度速率快（周期短）的任务。当然还要支持抢占式调度。

    在所有静态的调度策略中，RM是最优的。

对单调的软件调度器，解决意思问题的方案有限。

本文用硬件更好地解决了这个问题。

## INTRODUCTION

实时系统中，保证任务的及时完成很重要。使用可调度性分析（schedulability analysis）判断能否完成所有ddl。系统行为越可预测，可调度性分析的结果越可靠。

###  Rate-Monotonic Priority Inversion

当前实时系统的问题：硬件掌管中断的优先级，软件掌管线程的优先级。两套优先级导致低（线程）优先级的中断处理程序可能会中断高（线程）优先级的普通线程。此即为单调速率优先级倒置（rate-monotonic priority inversion）。这种现象的发生很难预测，同样会降低系统的可预测性。

### Parallel, Hardware-Supported Interrupt Handling

本文用硬件解决这个方案。将中断用一个硬件协处理器来处理，与一般线程平行。

其在这几个方面进行了改进：

- 支持存储一个任务的多个激活

    任务激活（task activation）是AUTOSAR-OS中的一个概念，指让当前处于挂起状态的任务运行。

- 多个任务使用相同的优先级，这时它们按被激活的顺序运行。
- 不使用信号量来通知事件的到来。

![](../图片/屏幕截图%202024-01-18%20160341.png)

## DESIGN

本文实现的硬件的功能：

- 与CPU平行工作（可视为协处理器）
- 处理外设发来的中断
- 访问调度器数据结构
- 像CPU发生消息（例如通过核间中断）

在TriCore上，使用PCP（peripheral control processor，外设控制处理器）实现该硬件。

避免单调速率优先级倒置的方法：

- 在PCP上激活中断处理程序设置中断控制系统
- 将任务优先级映射到中断优先级，实现统一的优先级。

PCP需要激活中断处理程序，因此需要访问调度器数据结构。当PCP激活的中断处理程序是就绪队列中优先级最高的程序时，PCP发送AST（asynchronous system trap，异步系统陷入）通知CPU，需要调度中断处理程序。

### Design of the PCP Interrupt Handler

![](../图片/屏幕截图%202024-01-18%20185954.png)

为了解决同步问题，设置了两个临界区CS1和CS2。PCP锁：为了防止PCP在处理中断时被更高优先级的中断打断而设置。调度器锁：为了保证对调度器的互斥访问设置。使用这两个锁，连续地guard两个临界区。

    ISR（中断服务程序）：本文中，ISR只指上图中的接收中断、获取中断处理task、激活task的过程。中断处理task本身不算ISR。

    本文中，ISR在PCP中执行，task在CPU中执行。

### Advantages over Software-Based Designs

    一个我们实现的时候需要注意的点：这种中断处理方式不需要屏蔽（mask）中断，因为任务分配平衡（task leveling）是通过中断系统硬件实现的，

#### Multiple Task Activation

用屏蔽中断来处理单调速率优先级倒置很不灵活，特别是在任务可以多次激活的时候。当在一个任务的运行期间多次激活这个任务时，采取中断屏蔽的系统可能会丢失这个任务的运行次数（因为中断被屏蔽了），而本文的方法则可以不丢失运行次数（因为每一次唤醒，都会加入一次队列）。

因此，本文的方法可以处理任务多次激活的情况。

#### Multiple Tasks per Priority

和Multiple Task Activation一样的理由，如果多个低优先级任务使用同一个优先级，当在高优先级任务的运行期间激活这些低优先级任务时……

#### Stack Sharing

栈共享（stack sharing）和软件调度的配合，需要信号量等高开销的同步原语。和硬件配合就不需要。

## IMPLEMENTATION

在TriCore平台和CiAO系统上实现。

### The TriCore Platform

使用PCP协处理器，只有用于通知CPU的AST中断仍然被CPU接收，其余所有中断都由PCP接收。

使用共享内存中的一个表格维护线程和中断源的映射。

### The PCP–Interrupt-Handler Implementation

该硬件的实现分为三个部分。

（确认中断由PCP自动实现）

**开端（prolog）**：为之后修改操作系统中的数据结构做准备。包括查询当前中断对应的优先级、调度器数据结构的地址。因为都是读操作，所以可以被高优先级的中断打断。

**CS1**：将该中断对应的线程加入调度器。

**CS2**：查询就绪队列中的线程优先级，如果没有比这次加入的中断处理线程优先级更高的，则通知CPU需要进行调度。

## EXPERIMENTAL EVALUATION

实验结果：在中断响应时间上，响应时间更短且减少了抖动。但因为硬件实现中加的两个锁，开销反而高于原有的情况。

## DISCUSSION

讨论中断过载（Interrupt Overload）对本方案的影响。

### Interrupt Overload

中断过载会影响本系统，但同样会影响软件实现的系统。对于本系统，可以在CPU的执行能力基本耗尽时，降低中断的优先级或屏蔽中断，从而缓解中断过载现象。

### Executing Threads and ISRs on the PCP

把一部分线程/中断处理程序反过来放到PCP上运行？不太可行。一方面，这会增加单调速率优先级倒置现象，与设计初衷背道而驰。另一方面，PCP算力太低了。

### Applicability

虽然本方案的具体实现与具体的硬件条件绑定了，但本方案的理念可以应用到很多软硬件平台上。