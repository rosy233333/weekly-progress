# vdso共享调度器debug日志

时间：2025/2/20

## 关于未定义符号的bug

共享调度器开发完成后，运行用户态测试代码，异常退出。

检查用户态映射过程，发现问题在于读取的elf中，`.dynsym`段有未定义符号（`Ndx=UND`）。

```Rust
// modules/executor/src/loader.rs
// fn: load_app
// line: 62
let relocate_pairs = get_relocate_pairs(&elf, elf_base_addr);
```

```Rust
// lib: elf_parser
// src/arch/riscv.rs
// fn: get_relocate_pairs
// line: 61
if let Some(dyn_sym_table) = elf.find_section_by_name(".dynsym") {
    let dyn_sym_table = match dyn_sym_table.get_data(elf) {
        Ok(xmas_elf::sections::SectionData::DynSymbolTable64(dyn_sym_table)) => {
            dyn_sym_table
        }
        _ => panic!("Invalid data in .dynsym section"),
    };
```

之后，了解到可以使用`readelf`检查vdso so文件的`dynsym`段，结果如下：

![](../图片/微信图片_20250220192917.png)

其中`Ndx=UND`的两个函数`memmove`和`memcpy`即是导致`get_relocate_pairs`发生painc的原因。

查询资料后发现，`.dynsym`段中的`UND`项代表依赖其它动态库的函数。而这两个函数是libc的函数。

因此，我计划将项目对libc的依赖方式改为静态链接。

尝试1：通过编译器参数`-C target-feature=+crt-static`强制静态链接。

结果：该编译器参数与`cdylib`的库类型冲突，因此无法通过编译。

尝试2：在打印中间结果以供查看时，发现在cargo调用指令前添加`RUSTFLAGS="--emit asm"`，可以使编译出的so文件不再依赖`memmove`和`memcpy`函数。该结果十分意外，因为理论上输出中间结果不应该影响编译的最终结果。但之后在[该网页](https://siliconsprawl.com/posts/rust-emit-asm/)上也发现了这个参数会影响最终结果的情况。

结果：反汇编后发现，输出的so文件有严重问题，例如`__vdso_delete_scheduler`函数被错误地优化成“直接将返回值置0后返回”了。因此该方法在产生错误的同时，并未根本解决问题。

尝试3：zfl学长向我介绍[该网页](https://os.phil-opp.com/zh-CN/minimal-rust-kernel/)，其上有将`memcpy`等函数直接内连进代码的选项。进一步了解后，通过在`config.toml`中添加以下内容，使项目构建过程中使用内置的`memcpy`实现重新编译`core`、`alloc`等库：

```toml
[unstable]
build-std-features = ["compiler-builtins-mem"]
build-std = ["core", "compiler_builtins", "alloc"]
```

结果：成功，编译出的so文件不再依赖`memmove`和`memcpy`函数，且反汇编后的代码没有明显问题。

## 关于so文件加载的bug

解决上一个bug后，继续运行（内核的）测试代码，发现触发了`StorePageFault`。

根据反汇编结果，出错的代码为一句`amoadd.d`指令。首先推测，是由于直接用不可变的全局变量模拟堆区导致的问题。因此考虑将该全局变量放在`.data`段，但问题依然存在。

之后进一步检查发现，`amoadd.d`指令的存取地址位于`.text`段。突然发现：我在内核态调用vdso代码时，并没有真正地**加载**vdso的二进制文件，而只是将二进制文件读入内存后就在该文件里查询符号。因此，vdso的数据和代码并没有放入相应的段中，而是一起放到了`.text`段里。因此接下来需要修改vdso代码的调用方式，添加加载过程。

不过，为了排除其它bug对结果的影响，决定先成功运行用户态调用vdso（其正常地加载了so文件），再修改内核态vdso的加载过程。

## 关于用户态vdso映射过程的bug

在用户态调用vdso时，依然出现了`StorePageFault`。

排查后发现，问题出在`vdso2memoryset`函数中（这个函数的作用是把`vdso`及`vvar`区域映射到用户空间），对`copy_nonoverlapping`函数的调用上。因为上面得到的`src`和`dst`都是`memory_set`中的虚拟地址（也就是用户空间的虚拟地址），但调用`copy_nonoverlapping`时，还位于内核空间中，因此会造成地址不正确。

于是我用`memory_set`的`query`方法尝试将用户空间地址转化为内核空间地址，却发现获取不到（返回了`None`）。因为新分配的两页还未建立内核到用户的映射。

于是我调整了顺序，先映射`vdso`区域，再映射`vvar`区域，不过还是会在访问`vvar`区域时出现访存异常。并且，`vdso`区域和`vvar`区域在内核空间中均会位于`text`段，而`vvar`不应位于`text`段。

此外，我也发现我对vdso和vvar的关系不是很了解，而这会影响到我对这段代码的修改。例如vdso区域中的数据应该是vdso的so文件的内容？那vvar区域的数据又是什么？vdso的代码是如何访问vvar的数据的？对这些内容不了解的话，也无法完全理解和修改`vdso2memoryset`的代码。因此，目前我正在进一步了解这部分内容。

进一步了解了vdso和vvar的关系：vvar在内核空间中保存在全局变量内，再映射到用户空间的特定地址。vdso通过该特定地址访问vvar变量。

因此，正在修改共享调度器，以匹配此数据访问逻辑。

但遇到了两个问题：

### 1. 在内核空间内，vvar不会正好位于vdso之前，因为内核虚拟地址使用平移映射，而vdso和vvar位于不同的段。而共享调度器要求在内核态调用vdso代码，即vdso代码需要在内核态也能访问vvar数据

解决方案1：在vdso_lib（与用户/内核代码一同编译，负责调用vdso代码的库）中封装传入vdso_data地址的逻辑？对vdso_data地址的处理放在vdso_lib中，对外不可见。

新问题：alloc函数也需要用到堆区地址，而该地址无法作为外部参数传入。

解决方案2：在内核空间将vdso和vvar映射到相邻的地址。

新问题：破坏内核空间的平移映射特性，可能对其它代码造成影响？

或者，不用映射，而是在内核空间分配相邻地址，将vdso和vvar复制过去？

### 2. 内核初始化时需要创建VdsoData对象，因此需要引用vdso库中的结构体和函数；但vdso库因为其被设置为cdylib，无法直接被内核代码引用

解决方案1：内核只负责提供内存页，初始化由vdso库内部负责；但这需要在特定地址上创建对象，不知Rust能否做到。（ds：可以先在别处创建对象，再拷贝过去）但在初始化完成前，vdso库内的alloc功能不可用。

解决方案2：将AllocArea和Scheduler的定义单独写成一个库。目前最可行的方案，但需要对代码做大规模重构。并且，在内核代码和vdso代码的target不同的情况下，引用同一个结构体是否会有相同的内存布局？（即使结构体被声明为repr(C)？）（ds说应该是会的，并且提供了验证的方式）
