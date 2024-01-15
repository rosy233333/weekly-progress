# qemu-journal笔记

时间：2024/1/5

原文：[https://github.com/U-interrupt/uintr/blob/main/doc/qemu-journal.md](https://github.com/U-interrupt/uintr/blob/main/doc/qemu-journal.md)

## 指令翻译、CPU状态

原文讲得很详细，因为我暂时不需要，就不做笔记了。

CPU对中断的处理主要在`target/riscv/cpu_helper.c`中，有以下三个函数：`riscv_cpu_all_pending`、`riscv_cpu_pending_to_irq`和`riscv_cpu_local_irq_pending`。

```C
static int riscv_cpu_all_pending(CPURISCVState *env) {
    return env->mip & env->mie;
}
/* 
主要关注两个参数 extirq_def_prio 和 iprio
先判断是否开启 AIA ，如果没有开启，则直接计算尾 0， 然后返回 irq
如果开启 AIA ，先计算特权级
    默认拿到 iprio[irq] 索引，若为 0
        如果是外部中断 extirq ，就赋值为 extirq_def_prio，否则根据 default_iprio 表中的信息拿到
按以上的方法，从右向左遍历，过程中获取最合适的 irq 返回处理
*/
static int riscv_cpu_pending_to_irq(CPURISCVState *env,
                                    int extirq, unsigned int extirq_def_prio,
                                    uint64_t pending, uint8_t *iprio);
/*
先获取对应特权态的 status 中断位，确认中断是否开启
然后按特权态由高到低的顺序对 riscv_cpu_all_pending 获得的 pending 进行处理
最后调用 riscv_cpu_pending_to_irq 返回中断请求 irqs
*/
static int riscv_cpu_local_irq_pending(CPURISCVState *env);
```

## QEMU RISC-V VirtIO Board

对`virt_machine_init`的讲解：其负责初始化CPU、外设

```C
static void virt_machine_init(MachineState *machine) {
    /*
    预分配一块内存用于存储设备的地址映射
        MemoryRegion *system_memory = get_system_memory();
    根据 CPU 插槽数进行遍历，实际 hart 数量由 smp 参数指定，关于这一部分在 hw/riscv/numa.c：
    调用 riscv_aclint_swi_create 建立对 VIRT_ACLINT_SSWI 和 VIRT_CLINT 的映射，并初始化 mtimer
    根据 s->aia_type 进行判断，如果是 VIRT_AIA_TYPE_NONE 就调用 virt_create_plic 初始化 PLIC
    否则调用 virt_create_aia 初始化 AIA

    接下来调用一系列函数进行外设初始化：
    virt_create_aia 初始化中断控制器
    memory_region_add_subregion 初始化 RAM 和 Boot ROM
    sysbus_create_simple 将 VirtIO 外设接入
    gpex_pcie_init 初始化 VIRT_PCIE_ECAM 和 VIRT_PCIE_MMIO
    serial_mm_init 初始化串口 VIRT_UART0
    virt_flash_create, virt_flash_map 初始化 flash

    最后根据地址映射创建设备树交给上层应用 create_fdt
    */
}
```

## plic代码

（原文讲的是aplic代码，我发现plic也是一样的）

### `sifive_plic.c/sifive_plic_realize()`

调用`memory_region_init_io`和`sysbus_init_mmio`对MMIO地址空间进行初始化，绑定读写函数。

```C
memory_region_init_io(&s->mmio, OBJECT(dev), &sifive_plic_ops, s,
                        TYPE_SIFIVE_PLIC, s->aperture_size);
sysbus_init_mmio(SYS_BUS_DEVICE(dev), &s->mmio);
```

调用`qdev_init_gpio_in`和`qdev_init_gpio_out`函数，注册GPIO端口。

```C
qdev_init_gpio_in(dev, sifive_plic_irq_request, s->num_sources);

s->s_external_irqs = g_malloc(sizeof(qemu_irq) * s->num_harts);
qdev_init_gpio_out(dev, s->s_external_irqs, s->num_harts);

s->m_external_irqs = g_malloc(sizeof(qemu_irq) * s->num_harts);
qdev_init_gpio_out(dev, s->m_external_irqs, s->num_harts);
```

调用`riscv_cpu_claim_interrupts`函数将中断信号唯一绑定至中断控制器。

```C
/*
 * We can't allow the supervisor to control SEIP as this would allow the
 * supervisor to clear a pending external interrupt which will result in
 * lost a interrupt in the case a PLIC is attached. The SEIP bit must be
 * hardware controlled when a PLIC is attached.
 */
for (i = 0; i < s->num_harts; i++) {
    RISCVCPU *cpu = RISCV_CPU(qemu_get_cpu(s->hartid_base + i));
    if (riscv_cpu_claim_interrupts(cpu, MIP_SEIP) < 0) {
        error_setg(errp, "SEIP already claimed");
        return;
    }
}
```

### `sifive_plic.c/sifive_plic_create()`

`sifive_plic_create`函数由外部 board 进行调用，负责将 GPIO 端口与 CPU 内部`IRQ_M_EXT`、`IRQ_S_EXT`连接起来，根据外部传入的地址完成地址空间的映射。

```C
for (i = 0; i < plic->num_addrs; i++) {
    int cpu_num = plic->addr_config[i].hartid;
    CPUState *cpu = qemu_get_cpu(cpu_num);

    if (plic->addr_config[i].mode == PLICMode_M) {
        qdev_connect_gpio_out(dev, cpu_num - hartid_base + num_harts,
                                qdev_get_gpio_in(DEVICE(cpu), IRQ_M_EXT));
    }
    if (plic->addr_config[i].mode == PLICMode_S) {
        qdev_connect_gpio_out(dev, cpu_num - hartid_base,
                                qdev_get_gpio_in(DEVICE(cpu), IRQ_S_EXT));
    }
}
```

    我的理解：GPIO端口相当于硬件输入输出端口，`sifive_plic_realize`的相关代码将端口和内部处理逻辑相连，`sifive_plic_create`的相关代码建立了端口与其它硬件（应该是CPU）的连接。

### 关于 IRQ 的注册

```C
// hw/core/irq.c
struct IRQState {
    Object parent_obj;

    qemu_irq_handler handler;
    void *opaque;
    int n;
};

typedef struct IRQState *qemu_irq;

typedef void (*qemu_irq_handler)(void *opaque, int n, int level);

void qemu_set_irq(qemu_irq irq, int level)
{
    if (!irq)
        return;

    irq->handler(irq->opaque, irq->n, level);
}

static inline void qemu_irq_raise(qemu_irq irq)
{
    qemu_set_irq(irq, 1);
}

static inline void qemu_irq_lower(qemu_irq irq)
{
    qemu_set_irq(irq, 0);
}
```

定义`IRQState`包含不同设备注册的`qemu_irq_handler`，中断高电平触发。`qemu_irq_handler`已经预先在CPU中注册，GPIO的输出引脚只需要将指针指向对应`IRQState`实例即可。

```C
// target/riscv/cpu.c
static void riscv_cpu_init(Object *obj)
{
    RISCVCPU *cpu = RISCV_CPU(obj);

    cpu_set_cpustate_pointers(cpu);

#ifndef CONFIG_USER_ONLY
    qdev_init_gpio_in(DEVICE(cpu), riscv_cpu_set_irq,
                      IRQ_LOCAL_MAX + IRQ_LOCAL_GUEST_MAX);
#endif /* CONFIG_USER_ONLY */
}
```

## 其它

之后还讲了一些改bug、调试、功能测试的内容，就等用到的时候再看。