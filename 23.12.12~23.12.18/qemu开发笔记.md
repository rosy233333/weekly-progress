# qemu开发笔记

时间：2023/12/15

## 内存映射

`MemoryRegion`表示一段逻辑内存区域，其可以将设备映射的内存作为后端（MMIO）。

其内部包含的`const MemoryRegionOps *`字段`ops`用于实现MMIO

[qemu内存管理——树状视图](https://blog.csdn.net/huang987246510/article/details/104012839)

[MMIO内存模拟原理](https://blog.csdn.net/huang987246510/article/details/123101595)

## 设备注册

[QEMU Device (qdev) API Reference](https://www.qemu.org/docs/master/devel/qdev-api.html)

在machine中创建设备，可以参考 [在qemu中模拟设备](https://zhuanlan.zhihu.com/p/57526565) 的“创建设备”节。

## 面向对象

[QEMU中的对象模型——QOM](https://blog.csdn.net/u011364612/article/details/53485856)