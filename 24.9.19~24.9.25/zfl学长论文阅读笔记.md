# 学长论文阅读笔记

时间：2024/9/21

## Introduction

**研究背景**：感知I/O任务的完成，已有的中断和轮询方式各有弊端 -> 将两者的弊端总结成相同的原因：信息不对称\* -> 引出本文工作：抹平信息差距的机制

\*解释：在中断方式中，CPU只知道I/O任务完成的时间信息，不知道CPU任务与I/O任务关联的信息；轮询方式中，则相反。

**本文工作**：

- 核心思想：使中断控制器感知任务状态，抹平信息差
- 核心工作：改进的任务状态模型、硬件调度任务
- 支持工作：修改的软件任务模块、实现软件运行时
- 支持工作：结合TAIC开发的网络驱动、部署到FPGA、测试性能

## Related Work

不太清楚，是否需要介绍提到的几个大类别（Fast Context Switching Supported by Hardware、 Interrupt Coalescing、Polling、Hybrid Interrupt and Polling Modes、Hardware Intelligence、Advanced Interrupt Controller）间的逻辑关系？

## Design of TAIC

## Implementation

没有什么好说的？

## Evaluation

有条理地递进开展了一系列实验：发包时钟周期测量、跑满带宽需要的最小包大小、负载与数据包无关的收包-发包时延、负载与数据包有关的收包-发包时延，测试的均是poll、interrupt、TAIC三种方案的性能对比。之后，再测试了有优先级和没有优先级（所有任务相同优先级）情况下的对比，以及真实情况下（ArceOS、Redis/YCSB）的性能测试。

每个实验都详细解释了结果，有时会在结果中凸显本方案的优势（例如，负载与数据包无关的收包-发包时延测量中，随着负载增大，TAIC的优势在相对比例上减小了，于是补充说明了其在绝对数值上增大）

对于实验结果解释了其中的原因（例如，负载与数据包无关的收包-发包时延测量中，TAIC方法性能比interrupt更优的原因是，避免了中断计算任务造成的缓存丢失和流水线清空这样的间接开销）

## Prospect

Usage Scenarios：通过地址映射，向用户态的进程提供访问；未来可以添加多进程的支持。

Extensibility：添加对时钟中断和软中断的支持。