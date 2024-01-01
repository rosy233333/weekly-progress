# 阅读sifive_plic的实现

时间：2023/12/15

参考资料：

[RISC-V SiFive U54内核——PLIC平台级中断控制器](https://zhuanlan.zhihu.com/p/588756601)提供了对这一硬件的介绍。

qemu源代码中的`hw/riscv/sifive_plic.c`和`include/hw/riscv/sifive_plic.h`实现了这个硬件。

## 杂项

在risc-v中，hart指硬件线程。

## 硬件介绍

sifive_plic使用内存映射来向CPU提供访问。CPU访问对应地址的虚拟内存即可读写plic寄存器。

映射区域的结构如图所示。

![](..\图片\v2-1f3c25d56b23533fbd68a54b19ad5c6a_720w.webp)

plic可提供132个中断源，ID分别为1~132，保留了ID=0未使用。

### 中断优先级（表中的priority）

每个ID占用32位存储优先级，但只使用其中的最低3位。

优先级取值为0~7，0表示禁用中断，1最低，7最高。

### 中断挂起位（表中的pending array）

每个ID占用1位存储挂起状态。

### 中断使能（表中的interrupt enables）

每个ID占用1位。

### 时钟门控（表中的clock gating disable）

看不懂作用。

### 优先级阈值（表中的priority threshold）

长度为32位，只使用最低3位。

只有优先级高于阈值的中断才会通知CPU。

### 中断声明与完成（表中的claim/complete）

中断声明：CPU读取该寄存器时，返回最高优先级挂起中断的ID，如果没有挂起中断则返回0。成功的声明还会自动清除中断源上相应的挂起位。

声明操作不受优先级阈值寄存器设置的影响。

中断完成：CPU执行完中断处理后，将中断ID写回该寄存器，以向plic表示它已完成。

## QEMU的实现

其内部状态由`SiFivePLICState`结构体表示。

### `SiFivePLICState`结构体

其中有`source_priority`、`target_priority`、`pending`、`claimed`、`enable`等与硬件介绍相符的字段。

`bitfield_words`：位图数据结构占了多少个words，应该大致相当于ceil(中断源数量/32)？

其还有一部分字段被注释为`config`，包括`hart_config`、`num_sources`等。其作用似乎是传入`sifive_plic_create`中，以创建一个`DeviceState`结构体。

### `sifive_plic_update`函数

似乎是根据`SiFivePLICState`，对CPU进行了一些操作（`riscv_cpu_update_mip`）。

### `sifive_plic_claim`函数

相当于读取claim/complete寄存器，返回值为最高挂起中断的ID。

### `sifive_plic_read`函数

根据plic的基地址，读取plic内部数据，plic根据地址所在区间返回相应的值。

应该是实现内存映射机制的核心函数之一了。

### `sifive_plic_write`

与`sifive_plic_read`相似，实现内存映射。

### `sifive_plic_ops`对象

是`MemoryRegionOps`的实例，封装了内存映射操作。

### `sifive_plic_properties`数组

`Property`数组，不知道用处，里面用到的宏需要注意一下。

```C
static Property sifive_plic_properties[] = {
    DEFINE_PROP_STRING("hart-config", SiFivePLICState, hart_config),
    // 省略相似行
    DEFINE_PROP_UINT32("aperture-size", SiFivePLICState, aperture_size, 0),
    DEFINE_PROP_END_OF_LIST(),
};
```

### `sifive_plic_realize`函数

应该是将`DeviceState`对象转化成`SiFivePLICState`对象？

并且使用了一个`qdev_init_gpio_in`进行注册。

### `sifive_plic_class_init`, `sifive_plic_info`, `sifive_plic_register_types`

应该是用来实现面向对象机制的东西？`SiFivePLICState`可能是`DeviceState`的子类？

怎么会有人用C写面向对象啊啊啊啊啊（）

### `sifive_plic_create`函数

根据`SiFivePLICState`的config字段，创建`DeviceState`对象。

