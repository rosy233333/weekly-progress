# 《Efficient IO with io_uring》阅读笔记

聚焦的阅读方向：

- `io_uring`相比原有IO方式的优势
- `io_uring`的高级功能
- `io_uring`的性能测试

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
