# ldh论文阅读笔记

## 问题与回答

对ldh工作的已有调研见[Rel4 Book调研笔记/ldh的工作](../25.8.14~25.8.20/Rel4%20Book调研笔记.md#ldh的工作)。

在此基础上，仍有需要弄清的问题：

- 用户态中断的处理流程，及其与Sel4原有中断处理的关系。ldh的工作中，申请notification对象的作用。
- endpoint按照文档也有异步的发送接收方式，为何选择notification而非endpoint进行改造？
- 中断模式与轮询模式结合的方式，能如何在此基础上提高性能？

### 用户态中断的处理流程，及其与Sel4原有中断处理的关系。ldh的工作中，申请notification对象的作用

用户态中断的处理机制与S态中断类似，首先注册中断向量，再通过置位`uip.USIE`触发中断。可以将已委托到S态的中断委托到U态，从而将高特权级的中断放在用户态处理。详见[此处](https://blog.kuangjux.top/2021/11/14/RISC-V-N%E6%89%A9%E5%B1%95/)。

ldh的工作中，对用户态中断的使用见[此处](https://rel4team.github.io/zh/docs/async/Background/%E7%94%A8%E6%88%B7%E6%80%81%E4%B8%AD%E6%96%AD/#risc-v%e7%94%a8%e6%88%b7%e6%80%81%e4%b8%ad%e6%96%ad%e7%9a%84%e8%bd%af%e7%a1%ac%e4%bb%b6%e6%8e%a5%e5%8f%a3)描述。

## 论文阅读笔记
