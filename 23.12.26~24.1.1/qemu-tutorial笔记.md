# qemu-tutorial笔记

时间：2024/1/1

原文：[https://github.com/OS-F-4/qemu-tutorial/blob/master/qemu-tutorial.md](https://github.com/OS-F-4/qemu-tutorial/blob/master/qemu-tutorial.md)

## 编译运行

一般在QEMU源码之外的文件夹进行编译。

```bash
$ mkdir qemu-build
$ cd qemu-build
$ ../qemu/configure --target-list="riscv64-softmmu"
$ make -j
```

运行：

```bash
# qemu-build目录下
$  ./qemu-system-riscv64 -machine virt # 运行
$  ./qemu-system-riscv64 -h # 查看帮助
```

重新下载源码，也是解决编译错误的手段之一。如果文件损坏，或者之前的编译错误产生了不一致性，可以重新下载源码。

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

