# vdso在用户态的重定位调研

时间：2025.3.31

## 背景

目前vdso共享调度器的bug在于：在OS的进程创建流程中，将vdso映射到用户空间后，对vdso的代码段做了一次重定位。但vdso在内核中的代码位于不可写的段，因此重定位会引发bug。

虽然在系统实现过程中，将vdso代码所在的单独段改为可写就能解决问题，但因为我不确定Linux中加载vdso的过程是否需要重定位（因为如果需要加载的话，应该也会遇到相同的读写权限问题），因此想学习Linux中对vdso在用户态重定位的处理，包括要不要重定位、如何重定位。

## 一般共享库的重定位过程

### 一般重定位过程

由OS执行：

- 映射（加载）主程序的各个段
- 准备执行环境
- 进入程序入口或动态链接器

由动态链接器执行：

- 动态链接器查询用到的所有共享库，加载它们的段，合并它们的符号表到主程序的符号表。
- 遍历重定位表，修正GOT/PLT中的地址。

### AsyncOS的特殊情况

重定位过程由OS而非动态链接器执行。目前还不知道AsyncOS是否支持动态链接到共享库。

### vDSO的特殊情况

- 在进入动态链接器之前，vdso代码已经加载好。（这个不同对AsyncOS来说不存在）
- vdso代码位于内核代码段，不可写。

## 对vDSO重定位过程的调研

在[RISC-V Syscall 系列 4：vDSO 实现原理分析](https://tinylab.org/riscv-syscall-part4-vdso-implementation/)中，提到了vdso需要进行重定位，从而获得`_vdso_data`等符号指向的正确地址。而重定位由`setup_vdso`函数完成，该函数位于glibc中，属于动态链接器。

文中提到的关键代码，其中重定位了vdso的首地址：

```C
// elf/setup-vdso.h
static inline void __attribute__ ((always_inline))
setup_vdso (struct link_map *main_map __attribute__ ((unused)), struct link_map ***first_preload __attribute__ ((unused)))
{
  ...
  l->l_map_start = (ElfW(Addr)) GLRO(dl_sysinfo_dso);
  ...
}
```

重定位过程中涉及了`link_map`结构，我对此不熟悉。因此查询资料后得知，它是动态链接器中用于描述一个共享库的数据结构。

在glibc仓库的动态链接器代码的`elf/setup-vdso.h`的函数`setup_vdso`中，注释介绍该函数的功能如下：

```text
Do an abridged version of the work _dl_map_object_from_fd would do to map in the object.  It's already mapped and prelinked (and better be, since it's read-only and so we couldn't relocate it).
We just want our data structures to describe it as if we had just mapped and relocated it normally.
```

其中提到，该函数为vdso代码设置了对应的`link_map`，使其看起来像已经被映射和重定位了一样。在之后的流程中，也不会对vdso对应的`link_map`再进行重定位（通过设置`l->l_relocated = 1`，在之后的`_dl_relocate_object`函数中跳过了对该`link_map`的重定位。）文中介绍的重定位代码似乎只是设置了`link_map`中，vdso代码的加载地址。

这里也提到了，vdso代码部分是只读的。

从中看出，vdso代码在映射到用户态后似乎是没有重定位的。那么，之前提到的`_vdso_data`的重定位问题是如何解决的？还是说，对`_vdso_data`的引用使用的都是相对PC寻址，因此不需重定位？

在我们实现的vdso中，读取符号表可知，`vdso_data`被赋值为ffffffffffffe000，与文章中对vdso_data的处理相同。

![alt text](image.png)

而内核采用的直接包含vdso二进制文件的方式，应该是没有对该so文件进行正常的装载和重定位的，而是将它们直接放在一个段中。（在Linux中，可能是代码段；在AsyncOS中，是一个单独的段。）

