# qemu-tutorial笔记

时间：2024/1/1

## 编译

还未编译通过，等通过了再来补充。

## 源代码概览

`target`目录对应指令架构，对于i386架构，`i386/tcg/translate.c`进行目标指令翻译成中间代码的核心工作。如果要添加新指令，就去这里修改。但riscv对应的文件夹里没有`translate.c`。

`hw/intc`存放中断控制器设备。我关注的是`sifive_plic.c`和`riscv_aplic.c`

## 输出调试

`#include "qemu/log.h"`，使用`qemu_log()`函数，用法和`printf()`相同。

因为加载linux后会清空输出，所以为了看到QEMU的log，需要将linux的输出重定向到其它地方。

```bash
$ ./run.sh | echo nothing
nothing
hello qemu!!
hello qemu!!
```

## 指令的翻译

我发现riscv的翻译代码和i386的有很大不同，本文内容并不适合riscv。

riscv的tcg核心函数可能是`target/riscv/translate.c/riscv_tr_translate_insn()`和`target/riscv/instmap`？这是我看名字看出来的。

