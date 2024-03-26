# ArceOS 网络api调用调研

时间：2024/3/22

## 概述

```Mermaid
flowchart TD
    ulib --> api --> arceos
```

## Rust API的调用

```Mermaid
flowchart TD
    ulib/axstd --> api/arceos_api --> modules/对应模块/src/api.rs --> modules/对应模块的功能代码
```

## Rust程序对Feature的启用

```Mermaid
flowchart TD
    ulib/axstd的features --> api/axfeat的features --> 各个模块的features
```

## 网络驱动调用情况调研

```Mermaid
flowchart TD
    ulib/axstd --> api/arceos_api --> axnet --实现网络协议--> smoltcp --调用网卡设备--> axdriver --> driver_virtio --> virtio-drivers
    axnet -.依赖.-> axdriver
```

### axstd

trait `ToSocketAddrs`，将各种类型转化为socket地址。

struct `TcpStream`，建立的TCP连接

struct `TcpListener`，TCP服务端

- 重点关注`TcpListener::accept`方法，其会阻塞线程直到TCP请求到来，适合进行异步改造

struct `UdpSocket`，用于UDP信息的发送和接收。其可以“建立UDP连接”，实际起到的作用为确定发送地址、限制接收地址。

### arceos_api

对`axnet`模块的`UdpSocket`和`TcpSocket`做了一层包装。

### axnet

struct `SocketSetWrapper`，存储socket列表。`Socket`具有`poll_at`函数（怎么这个函数的参数也是`Context`啊……所以Rust的异步模型和这里的poll是有渊源的吗？），有Udp、Tcp、Dns等多种socket。

struct `DeviceWrapper`，对设备的包装。具有`receive`和`transmit`两个方法，调用它们分别可以得到`AxNetRxToken`和`AxNetTxToken`。token在一个token-ring network中是唯一的，拿到token才可以调用设备发送/接收信息。

struct `InterfaceWrapper`，对接口的包装，存储了名称、以太网地址、设备和接口。可以设置接口的IP地址和网关。可以使用接口、设备、socket三者调用`poll`方法。（显然不是异步编程的`poll`而是网络的`poll`，但目前还不知道是什么意思）（也不知道接口的意思）

### smoltcp

提供Socket的相应方法和接口。

其中有`poll`方法，用于socket与网络设备交互，将socket内保存的内容发送、从设备接收发给该socket的内容。（因为socket内部具有缓冲区）。

以及`poll_at`和`poll_delay`方法提示下一次调用`poll`的建议时间。

### axdriver

从该层开始，不再需要关注TCP、UDP等传输层协议。

### driver_virtio

ArceOS中对`virtio-drivers`数据结构的一些包装。

struct `VirtIoNetDev`，virt IO网络设备，包装了`virtio-drivers` crate的`VirtIONetRaw`。内部也有缓冲区，负责发送与接收数据。

### virtio-drivers

struct `VirtIONetRaw`，表示virt IO网络设备。使用trait `Transport`作为与实际的virtio设备的通信接口。使用`MmioTransport`作为该trait的实现。该struct有控制中断的方法`ack_interrupt`、`disable_interrupts`、`enable_interrupts`，也有进行数据传输的方法`transmit_begin`、`poll_transmit`、`transmit_complete`、`receive_begin`、`poll_receive`、`receive_complete`等。

MMIO-virtIO机制介绍：

[通过MMIO的方式实现VIRTIO-BLK设备](https://zhuanlan.zhihu.com/p/389525645)

[Virtio-Net 技术分析](https://www.openeuler.org/zh/blog/xinleguo/2020-11-23-Virtio_Net_Technology.html)

在 virtio_net 的前端驱动和 qemu 中的后端驱动之间，有两个队列 virtqueue，一个用于发送，一个用于接收。

关于 virtqueue 可以看第一篇文章。

## 网络驱动异步化改造中，重点关注的函数

### 使用了线程阻塞功能的函数

`axnet` module中，部分函数可能会返回`AxError::WouldBlock`。将这些函数包装在`block_on`函数中，在返回`AxError::WouldBlock`时，自动yield之后重试，循环直到返回其它类型结果。这个函数可以作为异步化改造的切入点。

`block_on`函数在重试前会先调用一次`SOCKET_SET.poll_interfaces()`，暂且还不知道原因。

### 调用了硬件的函数

`virtio-drivers`中的`MmioTransport`类型。

但virtio网卡已经使用了轮询+DMA的方式与硬件通信，应该不会出现忙等硬件的情况了。

### 处理中断的函数

```Mermaid
flowchart TD
    axruntime::TrapHandlerImpl --> axhal::riscv64_qemu_virt::irq::dispatch_irq --非时钟中断--> axhal::irq::dispatch_irq_common --> 使用axhal::riscv64_qemu_virt::irq::register_handler注册的处理程序
    axhal::riscv64_qemu_virt::irq::dispatch_irq --时钟中断--> 使用axhal::riscv64_qemu_virt::irq::register_handler注册的TIMER_HANDLER
```

通过检查`register_handler`函数的使用情况，发现只注册了时钟中断的处理；

`axhal::riscv64_qemu_virt::irq`模块也说对plic的支持还未完成。

似乎virtio网络设备并未使用中断机制通知驱动？而是驱动使用轮询方式定期查询设备的工作结果？