# COPS及AsyncOS中vDSO的实现方式

## COPS

需要放入vDSO的代码为`lib_so/src/bin/sharedscheduler.rs`。该文件不属于`lib_so`模块，而是一个单独的Rust模块。

编译时，根据`lib_so/makefile`的配置，将`sharedscheduler`编译出的elf复制到`user/target`下。

内核启动时，在`os/src/lkm.rs`中，将`user/target`下vDSO的elf信息加载并放入内核空间，并调用`lib_so::init_spawn`等函数，设置`lib_so`模块中的代表各个vDSO函数指针的全局变量。因此，**内核可以通过`lib_so/src/lib.rs`中的`spawn`等相应函数调用vDSO函数**。

`os/src/mm/memory_set.rs`的`MemorySet::from_elf`中，在加载用户程序的elf时，先将vDSO的elf加载进用户空间，再通过访问用户程序elf中的符号表，将其引用的`lib_so`模块中的代表各个vDSO函数指针的全局变量的值改为vDSO函数的实际地址。因此，**用户也可以通过`lib_so/src/lib.rs`中的`spawn`等相应函数调用vDSO函数**。

## AsyncOS

需要放入vDSO的代码为`vdso/cops`模块。模块中包含配置链接行为的`vdso/cops/cops.lds`。

在`vdso/src/vdso.S`中，包含了vDSO代码编译后的so文件，并在其开始和结束使用`vdso_start`和`vdso_end`标记。

在`vdso/src/lib.rs`中，根据`vdso_start`和`vdso_end`两个全局符号初始化内核中的`VDSO_INFO`全局变量。**内核可以通过`VDSO_INFO`获取vDSO对象，并通过读取elf的方式调用其中的函数。**

加载用户程序时，在`modules/executor/src/loader.rs`中，使用`vdso`模块的`vdso2memoryset`函数，将`VDSO_INFO`映射到用户空间，并将其基址传入用户程序的`auxval`的`AT_SYSINFO_EHDR`项中。因此，**用户可以通过获取`auxval`中的相应项以获取vDSO对象，并通过读取elf的方式调用其中的函数。**