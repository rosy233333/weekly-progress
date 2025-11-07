# 《Efficient IO with io_uring》阅读笔记

聚焦的阅读方向：

- `io_uring`相比原有IO方式的优势
- `io_uring`的高级功能
- `io_uring`的性能测试

## 评价

首先，从本文中我更加了解了io_uring的几个高级功能：注册文件与缓冲区、IO轮询、SQ轮询。

在设计层面，io_uring的优势主要体现在简易和统一的接口上。对用户而言，它通过liburing库提供了易于使用的接口；对内核而言，它使用统一的接口兼容了多种类型的IO任务。

在性能层面，io_uring对IO轮询的支持使其具有较高的峰值性能。作为经过内核的框架，其性能与绕过内核的spdk相近。而在不使用IO轮询的情况下，其性能则高于aio。这些都证明框架本身引入的开销较小。

## Introduction & Improving the status quo

已有的非阻塞I/O接口：Linux实现的，POSIX的`aio`接口。[见此处。](https://cloud.tencent.com/developer/article/1810604)其局限在于：

1. 只支持`O_DIRECT`模式（不缓存，直接写入磁盘）的异步IO。若没有`O_DIRECT`标志，则进行同步IO。同时，`O_DIRECT`模式也对数据的大小和对齐有额外要求。
2. 即使在`O_DIRECT`模式下，也可能出现阻塞。（例如，在磁盘的请求槽（request slot）已满的情况下）
3. API设计问题。带来额外的数据拷贝、使用难度、安全风险。

原有的对异步IO的改进思路主要在改进`aio`的接口上。但在`aio`的接口本就复杂的前提下，对其的修改仅解决了最突出的问题（不再要求`O_DIRECT`模式？），但使接口更加复杂了。

## New interface design goals

新接口的设计目标（重要性大致从高到低）：

- 易于使用，难以误用。
- 可扩展（Extendable）。支持块式存储、网络、非块式存储等不同IO设备。
- 功能丰富。不像`aio`只能满足小部分应用程序，新的接口需要满足大部分应用的异步IO需求。
- 高效。在不同种类的IO需求下高效，例如块式存储常用的512B/4KB请求，或者某些不带数据负载的请求。
- 可伸缩（Scalability）。低负载下的低时延，以及高负载下尽可能提高性能均很重要。

## Enter io_uring

`io_uring`的设计还是围绕性能展开。通过移除不必要的拷贝，相比`aio`提高了性能和可伸缩性。

移除拷贝 => 使内核与用户共享IO提交和完成事件的数据结构 => 采用共享内存。

共享数据结构 => 无锁（因为内核和用户难以共享锁） => SPSC环形缓冲区

两个IO操作：提交与完成，各使用一条队列。

### Data structures

```C
struct io_uring_cqe { 
    __u64 user_data;
    __s32 res;
    __u32 flags;
};
```

- `user_data`：由应用定义，用于确认每一个返回值对应的提交
- `res`：原系统调用的返回值
- `flags`：存放元数据，暂未使用

```C
struct io_uring_sqe {
    __u8 opcode;
    __u8 flags;
    __u16 ioprio;
    __s32 fd;
    __u64 off;
    __u64 addr;
    __u32 len;
    union { 
        __kernel_rwf_t rw_flags; 
        __u32 fsync_flags; 
        __u16 poll_events; 
        __u32 sync_range_flags; 
        __u32 msg_flags; 
    };
    __u64 user_data; 
    union { 
        __u16 buf_index; 
        __u64 __pad2[3];
    };
};
```

- `flags`：对应IO操作中的`flags`
- `ioprio`：IO优先级，见`ioprio_set`系统调用
- `off`：IO操作在`fd`指向文件中的偏移
- `addr`、`len`：读写缓冲区
- 第一个`union`：与`opcode`相关的额外信息
- `user_data`：由应用定义，用于确认每一个返回值对应的提交。系统会将其无修改地复制到对应的`cqe`中。
- `buf_index`：用于预注册的缓冲区。
- `pad`：填充空间以使`sqe`大小对齐到64B。

### Communication channel

`cq`是直接由`cqe`构成的环形队列，但`sq`使用了间接索引：`sq`的元素为一个索引，使用该索引在`sqe`数组上查找才能获得`sqe`。

在`sq`上使用间接索引的原因是方便应用程序将请求嵌入其它数据结构中。

一旦内核消费了`sqe`，即使其对应的IO操作尚未完成，该`sqe`也可被重用。

一般来说，`sq`的大小决定了同时有多少用户发出的IO请求正在内核中处理。但由于内核消费`sqe`的时机IO请求可能未完成，实际上内核中可能存在多于`sq`大小的IO请求数量。因此，为了避免`cq`被用尽，默认`cq`的大小是`sq`的两倍。

`cqe`的顺序与`sqe`的顺序无关联。

## io_uring interface

`io_uring_setup`和`io_uring_enter`见原文。

### sqe ordering

`io_uring`最适合提交无序的情况，但有时候提交的顺序是有需求的（例如先写文件，再`fsync`）。

此时可以在sqe的`flags`中设置`IOSQE_IO_DRAIN`位，这会使内核先将之前的所有IO操作都完成后，再执行此IO操作。

### Linked sqes

更细粒度的顺序控制：在sqe的`flags`中设置`IOSQE_IO_LINK`位，则如果上一个提交也有`IOSQE_IO_LINK`，那么该提交在上一个提交成功完成后才会执行；若上一个提交失败，则该提交也会失败并返回`ECANCELED`。

### Timeout commands

两种超时操作：一种在经过特定时间后触发，一种在完成特定数量的任务后触发。

超时操作触发后，该`io_uring`上所有正在等待完成的线程都会被唤醒。

## Memory ordering

使用底层API（而非liburing库）需要重视的内存访问顺序。

`read_barrier()`：*调用它之前的写操作* 对 *调用它之后的读操作* 可见

`write_barrier()`：*调用它之前的写操作* 一定在 *调用它之后的写操作* 之前发生

发送操作：用户在填写sqe后、提交sqe（即更新SQ.tail）之前设置`write_barrier`，并在提交sqe后设置`write_barrier`（此处可能是为了避免并发冲突？）；内核在读取sq.tail之前设置`read_barrier`。

接收操作：与发送操作类似。

## liburing library

liburing和底层API可以混用。因此可以将使用底层API的程序部分改造为使用liburing。

初始化与释放：

```C
struct io_uring ring;
io_uring_queue_init(ENTRIES, &ring, 0);

io_uring_queue_exit(&ring);
```

发送与接收：

```C
struct io_uring_sqe sqe;
struct io_uring_cqe cqe;

/* get an sqe and fill in a READV operation */

sqe = io_uring_get_sqe(&ring);
io_uring_prep_readv(sqe, fd, &iovec, 1, offset);

/* tell the kernel we have an sqe ready for consumption */
io_uring_submit(&ring);

/* wait for the sqe to complete */
io_uring_wait_cqe(&ring, &cqe);

/* read and process cqe event */
app_handle_cqe(cqe);
io_uring_cqe_seen(&ring, cqe);
```

此外：不等待地查看CQ是否有完成事件：`io_uring_peek_cqe`

## Advanced use cases and features

### Fixed Files and Buffers

每次将文件描述符填充到 sqe 中并提交到内核时，内核都必须检索对所述文件的引用。IO 完成后，文件引用将再次被删除。由于此文件引用的原子性质，对于高 IOPS 工作负载来说，这可能会明显减慢速度。

使用`O_DIRECT`时，内核必须先将应用程序页面映射到内核中，然后才能对它们执行IO，然后在IO完成后取消映射这些相同的页面。

因此，使用如下系统调用提供了预注册文件与缓冲区的功能：

`int io_uring_register(unsigned int fd, unsigned int opcode, void *arg, unsigned int nr_args);`

- `fd`：io_uring的fd
- `opcode`：注册的类型
- `nr_args`：`arg`数组的大小

注册文件：`opcode = IORING_REGISTER_FILES`，`arg`为`fd`组成的数组。之后，sqe可以在设置了`IOSQE_FIXED_FILE` flag的情况下，使用下标访问预注册的文件。注册的文件会在io_uring实例被释放时一同释放，或者使用`opcode = IORING_UNREGISTER_FILES`的`io_uring_register`手动释放。

注册缓冲区：`opcode = IORING_REGISTER_BUFFERS`，`arg`为`iovec`数组。之后，sqe可以使用`IORING_OP_READ_FIXED`和`IORING_OP_WRITE_FIXED` op读写这些缓冲区，只需保证读写的地址范围在已注册的缓冲区之中即可。

### Polled IO

在内核与IO设备的通讯间使用轮询。

启用轮询：在`io_uring_setup(2)`系统调用或者`io_uring_queue_init(3)` liburing函数中指定`IORING_SETUP_IOPOLL`。

启用轮询后，可以不再从CQ接收完成事件，而是通过频繁调用`io_uring_enter(2)`系统调用并指定`IORING_ENTER_GETEVENTS`和`min_complete`来获取完成事件。如果`min_complete`设置为0，则是在内核没有完成事件时也直接返回，不会在内核自旋等待。（类似于peek和wait的区别）

但也可以继续从CQ接收完成事件。（那么，提供上文的接口的意义在哪里？可以提高实时性吗？）

只有部分`opcode`支持轮询：`IORING_OP_READV`、`IORING_OP_WRITEV`、`IORING_OP_READ_FIXED`、`IORING_OP_WRITE_FIXED`

### Kernel side polling

用户侧提交sqe后，不需要`io_uring_enter(2)`系统调用，内核也能处理提交的sqe。通过在内核创建轮询线程实现。

要使用此功能，io_uring实例需要在 `io_uring_params` `flags`中指定`IORING_SETUP_SQPOLL`，或传递给 `io_uring_queue_init(3)`。

使用`IORING_SETUP_SQ_AFF` flag和`io_uring_params` `sq_thread_cpu`，可将内核轮询协程限定在某个CPU上。

内核轮询线程会在负载不大时睡眠。此时，它会设置SQ-ring的`flags`中的`IORING_SQ_NEED_WAKEUP`。用户提交新的IO请求时，需要调用`io_uring_enter(2)`系统调用并指定`IORING_ENTER_SQ_WAKEUP`。

通过`io_uring_params` `sq_thread_idle`可设置内核轮询线程在经过多久的空闲后进入睡眠。（单位：毫秒；默认：1000）

启用该功能需要用户程序具备一定权限。

## Performance

[https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/](https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/)

- 同样使用IO轮询的io_uring和spdk在不同测试项目和不同条件下各有胜负、差距不大。
- 不使用IO轮询的io_uring领先于aio。
- 使用IO轮询的框架（io_uring（使用IO轮询）、spdk）性能远高于不使用IO轮询的（io_uring（不使用IO轮询）、aio）。

### Raw performance

从块设备/文件随机读取，峰值性能（单位：4k IOPS）：

- io_uring（启用IO轮询）：1.7M
- io_uring（未启用IO轮询）：1.2M
- aio（不支持IO轮询）：608K

使用no-op操作测量的接口原始吞吐率：20M消息/秒，影响因素：系统调用数量、内存速率、SQ及CQ大小。

### Buffered async performance

使用内核缓冲区可以提高IO性能，其将一些常用的文件页缓存在内核，当用户读取时直接提供，不再需要请求IO硬件。

io_uring可以利用这种机制。当需要进行的IO操作已在内核缓存时，其会同步地进行IO，并在`io_uring_enter(2)`系统调用返回时已经生成了完成事件。
