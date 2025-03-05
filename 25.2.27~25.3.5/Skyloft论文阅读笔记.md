# Skyloft: A General High-Efficient Scheduling Framework in User Space 阅读笔记

## Abstract

Skyloft是用户空间调度框架，使用用户态中断，在用户态处理计时器，从而做到微秒级抢占。

支持抢占式或非抢占式接口。

兼容Linux，可无缝集成DPDK等高性能I/O框架

## Introduction

优化尾部时延很重要。

已有研究使用user agents、live updates和Berkeley Packet Filter (BPF) 对Linux调度器进行修改。（见原文引用文献）

在内核调度线程/在内核实现新的调度策略的不足：

- 频繁的特权级切换
- 降低代码/数据局部性
- 修改被已有接口限制
- 需要对内核做大量修改
- 无法利用内核旁路的驱动和框架

现有的用户态调度框架的不足：无法同时支持微秒级调度和多应用支持。

微秒级调度的重要性：（看原文）

（用户态）多应用调度的重要性：处理延迟敏感型（latency-critical, LC）应用和尽力型（best-effort, BE）应用的关系：在CPU空闲时运行BE应用，在CPU负载高峰期快速切换到LC应用。

Skyloft的优势：

- 灵活：不同的调度策略、抢占或非抢占模式
- 高效：微秒级抢占。为了该目标，需要实现轻量级上下文切换、高效中断管理、快速资源分配和同步原语
- 兼容：兼容内核旁路框架；兼容Linux。用户程序可以在Linux原生调度器和Skyloft间选择

Skyloft的实现：

- 微秒级抢占：用户态中断。支持中心化抢占调度和per-CPU抢占调度两者方式，在per-CPU抢占调度中，支持在用户态处理时钟中断。
- 多应用调度：不使用全局控制器，而是在每个核上运行一个主循环，从而在用户态和内核态间提供清晰的接口。

Skyloft的评估：

- 在schbench上，per-CPU策略的Skyloft取得了比Linux原生调度器低几个数量级的唤醒时延
- 在人工负载上，中心化策略的Skyloft取得了比ghOSt高1.2倍的最大吞吐量
- 在现实应用（Mencached）上，采用任务窃取的Skyloft与Shenango性能相同
- 在现实应用（RocksDB）上，抢占式的Skyloft在达到了相同的尾部时延SLO的情况下，取得了比Shenango高1.2倍的性能
- Skyloft使用核间中断的抢占开销是0.6μs；处理时钟中断的开销是0.3μs，比软件定时器快了将近10倍。

文章的主要贡献；

- Skyloft使用用户态中断实现了内核旁路的抢占，且同时支持微秒级抢占和多应用。
- Skyloft提供了使用一组通用操作实现不同调度策略的范式。
- Skyloft取得了更好的性能。