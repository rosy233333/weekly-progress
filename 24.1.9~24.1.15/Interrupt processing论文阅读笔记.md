# Interrupt processing with queued content-addressable memories 阅读笔记

时间：2024/1/11

原文：[https://dl.acm.org/doi/10.1145/1478462.1478553](https://dl.acm.org/doi/10.1145/1478462.1478553)

## Introduction

讲了一下基于硬件的、带有优先级的中断处理的重要性。对优先级的确定和划分比较复杂。

讨论了固定优先级和动态优先级的优劣。动态优先级可以让计算机更能对环境变化做出响应，并且避免了高优先级的中断突然增多导致低优先级中断无法处理的情况。

## The Queued Content-Addressable Memory

硬件队列：需要固定长度，使用一个偏移寄存器（shift register）保存队尾位置（我的问题：这里不用保存队首位置吗？）

内容寻址存储器（Content-addressable memory，CAM）：文中没介绍，我自己找了介绍，CAM，通过数据的内容查找地址。每个地址只能存储唯一数据，但特定的数据可以存在多个地址中。RAM一般按字读写，但CAM一般按位读写。

[CAM（Content-Addressable Memory）介绍](https://blog.csdn.net/qq_42322644/article/details/109170011)

本文采用了队列和CAM结合的存储结构，每个CAM项均为一个队列。存入数据时，先看其key是否与现存的某个队列符合，符合则存到该队尾；否则存入一个新的队列/CAM项。

![](../图片/屏幕截图%202024-01-11%20120041.png)

## Interrupt Processor Organization

在如下设计中，预先规定其服务于32位计算机，且与CPU和内存相连。这里只考虑了单核情况，但几个参考文献（3，4，5）描述了如何应用于多核。

64个优先级，16个硬件中断源，共1024个中断。可以扩展到256个优先级和4096个中断源。

总体架构如下。主要组成部分为：优先级结构（priority structure）、随机访问便笺存储器（random access scratchpad）和队列CAM（queued CAM）。

![](../图片/屏幕截图%202024-01-11%20155447.png)

### Priority logic

通过嵌套的优先级树管理中断线，此处的优先级决定各种中断**接受和存储**的顺序，而另一个程序控制的优先级决定各种中断**处理**的顺序。

通过一个寄存器，用位图方式存储每条中断线是否启用。

活跃的最高优先级的中断线被编码为一个8bit数。将这个数送到便笺地址寄存器（Scratchpad Address Register，SAR）（应该是`level`字段？），对应的4bit中断源id存入CAM输入数据寄存器的一个字段中。

之后，设置一个锁存器，以对发出中断的设备发送确认信号，该信号同时将这一中断线从优先级树上移除，直到该设备下次reset（指的是重新发出中断吗？）该中断。

我的理解：因此，这个优先级树是为了存储已经发出的中断，并维护它们之间的优先级？

### Scratchpad 

便笺（scratchpad）是一个高速RAM，其为每一条中断线存储一个字。字长为25bit，格式如下：

![](../图片/屏幕截图%202024-01-11%20171608.png)

`level`为对该中断线设置的优先级（可变）。一个优先级可以对应多个中断源，且特定条件下一个中断源也可以对应多个优先级。

`pointer`为对应中断的处理例程的地址。

`e`字段比较特殊，既可以将单个中断线的`e`字段看作一个存储字中的字段，对其单独赋值；也可以将所有64条中断线的`e`字段看作一个存储字，对它们整体赋值（为全1或全0）。该字段代表中断线的使能。若未被使能的中断线有中断到来，其不会马上处理，而是被存储起来，直到该线被使能才会被处理。

便笺存储器通过行号来寻址。

它从优先级逻辑或队列CAM中读取请求，并在出现冲突时，向队列CAM提供优先级信息。（不确定，原文：Scratchpad read requests come from either the priority logic or the queued CAM, with preference given to the latter in the event of conflict.）

改变便笺存储器内容的三种方式：以字为单位修改；从内存中加载一块连续区域的数据；通过旁路寻址（sideways addressability）修改所有`e`字段。

从内存中加载数据时，受影响的便笺存储器区域暂时无法被优先级逻辑或队列CAM访问；若是将数据写入内存中，则不会影响另外两部件的读取。

### Queued CAM 

文中说，这是这个硬件的核心。

其包含至少8个存储字的CAM。每个CAM存储字都配备一个8个字的FIFO队列中。每个字（及其附属队列）对应一个优先级，只要这个优先级中有活跃或等待中的中断。

每个队列项字长21bit，格式如下：

![](../图片/屏幕截图%202024-01-11%20181253.png)

`level`字段从便笺中获得（即为优先级），`line`（行号）和`id`（中断源id）从优先级逻辑中获得。`f`（filled）代表本项是否填充了有意义的数据，其同时控制数据在队列中的前进。将`f`设置为0即可删除这一数据项。

CAM项字长为23bit，格式如下：

![](../图片/屏幕截图%202024-01-11%20181733.png)

`r`（running）代表这个优先级的中断处理例程是否正在CPU中运行，`e`（enable）看不懂，直接贴原文：The E (Enable) bit in the CAM is updated as each new request enters the CAM and whenever its counterpart changes in the scratchpad. 

所有对CAM的搜索/寻址都使用`level`字段（或者结合`e`和`level`字段）。因为`level`各不相同，因此搜索不会出现多个结果。CAM也可能在`line`字段上查询，从而清楚特定行产生的中断请求。这种情况下可能会出现多个结果。

## Interrupt Processor Operation

中断处理的三个阶段：辅助函数（例如改变便笺内容、给中断线使能）、将中断输入到队列CAM中、向处理器输出服务结果。

### Arming and enabling interrupts

中断通过其对应的行号来使能。其arm和disarm由优先级逻辑中的arm寄存器（arm register）来实现；enable和disable由便笺中的`e`字段来实现。便笺存储器中的`e`字段变化后，CAM中相应行的`e`字段会相应地变化。

我的问题：arm/disarm和enable/disable有什么不同？

辅助程序可以在以下方面控制中断控制器：arm/disarm、enable/disable、修改优先级、修改中断处理例程的指针。

在它们修改相应内容时，如何处理可能被影响的、正在处理的中断，有几种方案：正常处理、清除受影响的项、阻塞所有输入、阻塞受影响的输入、阻塞所有输出、阻塞受影响的输出。

### Detecting and acknowledging interrupts

当中断到达时：

1. 设置holding寄存器的对应位
2. 如果中断线armed了，将该中断加入优先级树，为树中最高优先级的行分配一个8bit便笺地址（对应`level`）
3. 将该中断源id存入队列CAM
4. 向发出中断的设备返回一个确认信号，将该中断从优先级树上删除，直到该设备下一次发出中断。

### Storing and scheduling interrupts

对于存入CAM的中断信号：

1. 如果CAM中已有该优先级的中断信号，则放入对应队列的尾部；否则，放入一个新的CAM槽位中。
2. CAM检索`e`和`level`字段合并后的最大值项（enable的最高优先级），如果其`r`为0，则对CPU发出中断，执行中断处理例程；否则，无动作。

另：软件可以考虑将CAM中溢出的数据项存入内存。

### Initiating CP service

IP让CPU启动中断处理例程的条件（满足其一即可）：

1. 一个高优先级且enable的中断存入CAM。
2. CPU enable了一个高于现在优先级且有中断正在等待的优先级。
3. 高优先级的中断处理完成后，有低优先级的中断正在等待。

IP将相关行号放入便笺的寻址寄存器（？），当获得许可后，IP通知CPU，向其发送中断优先级、中断源id、处理例程地址。

### Terminating CP service

1. 中断处理例程结束后，CPU向IP返回处理中断的优先级。
2. CAM删除其对应的项（将`f`置0），让该队列中之后的数据项前移。新的队首请求的`e`位通过查询便笺获得。
3. 再次进行`e`和`level`字段的最大值检索。