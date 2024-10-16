## 和域相关的代码

### kernel/src/domain

调用create_domain!和register_domain!进行域的创建和注册

### kernel/src/domain_helper

DOMAIN_CONTAINER、DOMAIN_CREATE、DOMAIN_INFO、DOMAIN_RESOURCE等存放所有域的相关信息的全局变量

register_domain!即是将创建好的域注册进这些全局变量中。

此外，在此处定义了共享堆和私有堆，以及由域代码使用的安全系统调用。

### kernel/src/domain_loader

实现域的创建和加载。

### kernel/src/domain_proxy

生成各个域的代理。

使用对应的宏gen_for_XXDomain!()生成代理结构的定义。

对于一些代理结构，需要通过impl实现其中的特殊方法：DevFsDomainProxy、BlkDomainProxy、SchedulerDomainProxy

DomainProxy也是一个Trait，需要impl ProxyBuilder、Basic和对应的Domain

### domain-lib/interface

以Trait形式定义了各种域类型的接口。

### domain-lib/gproxy

用于生成域代理？

### domains

存放各个域的代码。

## 如何（基于已有接口）写一个新的域

在domains目录下，可以使用现有的工具程序进行查看帮助、编译、新建域等操作：

```
# help
cargo domain --help # Display help

# build
cargo domain build-all -l "" # Build all domains
cargo domain build -n gsyscall -l "" # Build syscall domain

# create
cargo domain new --name new_domain
# 之后，根据工具程序的提示选择域类型和域接口（需要是domain-lib/interface中已有的接口）
```

创建域之后，需要手动将其放入`domains/domain-list.toml`的`members`以及【`init_members`和`disk_members`取其一】中。（猜测，`init_members`为系统启动时加载的域，`disk_members`为存放在硬盘中的域。）

如果设置为`init_members`，则还需要将新创建的域加入`kernel/src/domain/init.rs`的`INIT_DOMAIN_LIST`中。

TODO：如何使操作系统加载硬盘上的域？

使用`new`命令创建好新域对应的crate后，在其中实现新的域，并在`main`函数中返回创建的新域的对象。

在此处编写好的域，会在Alien的编译流程中编译为单独的二进制文件，并被Alien系统读取和加载。

## 系统运行进度

按照`README.md`中的步骤运行。

首先出现错误，查询后发现，因为没有安装`riscv64-linux-musl-gcc`。从[https://github.com/michaeljclark/musl-riscv-toolchain/releases/tag/v7.2.0-7](https://github.com/michaeljclark/musl-riscv-toolchain/releases/tag/v7.2.0-7)仓库中下载安装了对应工具。

之后出现了如下错误：

```
          riscv64-linux-musl-gcc: error: crt1.o: No such file or directory
          riscv64-linux-musl-gcc: error: crti.o: No such file or directory
          riscv64-linux-musl-gcc: error: crtbegin.o: No such file or directory
          riscv64-linux-musl-gcc: error: crtend.o: No such file or directory
          riscv64-linux-musl-gcc: error: crtn.o: No such file or directory
```

询问clf学长，得知可能是工具链安装不完全。目前先将出错的用户程序的编译命令注释掉，跳过了该程序的编译。

之后，又出现了读取`libc.so`出错导致的运行时错误。询问学长得知，在安装busybox的过程中，需要设置静态链接。

- 系统在第一次编译时，会下载安装busybox。
- 安装的过程中，需要维持终端窗口大小较大（例如，如果使用vscode的集成终端，则需要使终端全屏），从而使busybox的安装程序可以进入图形化界面的菜单。
- 在菜单中，进入Settings，选中Build static binary (no shared libs)。
- 保存设置并退出界面，之后就可以自动地继续编译。

进行这些修改后，成功编译运行了Alien系统。

## Alien系统上和任务调度相关的内容

目前已知，有`Scheduler`和`Task`两个域，还涉及`domain-lib/task_meta`模块和`kernel/src/task`。

### 各个模块的关系

`task_meta`提供任务的基本信息，包括状态、上下文等。被`TaskDomain`和`kernel/task`使用。

`SchedulerDomain`提供调度器的接口。被`kernel/task`使用。

`TaskDomain`提供访问进程管理的资源、进程相关系统调用的接口。它负责任务的创建和任务信息的管理。在系统调用的处理过程中使用。

`kernel/task`实现任务的调度、运行和切换。

询问学长后得知：`kernel/task`负责连接`TaskDomain`和`SchedulerDomain`。

### `SchedulerDomain`域

定义：domain-lib/interface/src/scheduler.rs

接口：

```Rust
#[proxy(SchedulerDomainProxy, RwLock)]
pub trait SchedulerDomain: Basic + DowncastSync {
    fn init(&self) -> AlienResult<()>;
    /// add one task to scheduler
    fn add_task(&self, scheduling_info: RRef<TaskSchedulingInfo>) -> AlienResult<()>;
    /// The next task to run
    fn fetch_task(&self, info: RRef<TaskSchedulingInfo>) -> AlienResult<RRef<TaskSchedulingInfo>>;
}
```

实现：domains/common/fifo_scheduler和domains/common/random_scheduler。此外，还有帮助实现`Scheduler`域的工具代码库：domains/common/common_scheduler

### `TaskDomain`域

定义：domain-lib/interface/src/task.rs

接口：

```Rust
#[proxy(TaskDomainProxy, RwLock)]
pub trait TaskDomain: Basic + DowncastSync {
    fn init(&self) -> AlienResult<()>;
    fn satp_with_trap_frame_virt_addr(&self) -> AlienResult<(usize, usize)>;
    fn trap_frame_phy_addr(&self) -> AlienResult<usize>;
    fn heap_info(&self, tmp_heap_info: RRef<TmpHeapInfo>) -> AlienResult<RRef<TmpHeapInfo>>;
    fn get_fd(&self, fd: usize) -> AlienResult<InodeID>;
    fn add_fd(&self, inode: InodeID) -> AlienResult<usize>;
    fn remove_fd(&self, fd: usize) -> AlienResult<InodeID>;
    fn fs_info(&self) -> AlienResult<(InodeID, InodeID)>;
    fn set_cwd(&self, inode: InodeID) -> AlienResult<()>;
    fn copy_to_user(&self, dst: usize, buf: &[u8]) -> AlienResult<()>;
    fn copy_from_user(&self, src: usize, buf: &mut [u8]) -> AlienResult<()>;
    fn read_string_from_user(
        &self,
        src: usize,
        buf: RRefVec<u8>,
    ) -> AlienResult<(RRefVec<u8>, usize)>;
    fn current_pid(&self) -> AlienResult<usize>;
    fn current_ppid(&self) -> AlienResult<usize>;
    fn do_brk(&self, addr: usize) -> AlienResult<isize>;
    fn do_clone(
        &self,
        flags: usize,
        stack: usize,
        ptid: usize,
        tls: usize,
        ctid: usize,
    ) -> AlienResult<isize>;
    fn do_wait4(
        &self,
        pid: isize,
        exit_code_ptr: usize,
        options: u32,
        _rusage: usize,
    ) -> AlienResult<isize>;
    fn do_execve(
        &self,
        filename_ptr: usize,
        argv_ptr: usize,
        envp_ptr: usize,
    ) -> AlienResult<isize>;
    fn do_set_tid_address(&self, tidptr: usize) -> AlienResult<isize>;
    fn do_mmap(
        &self,
        start: usize,
        len: usize,
        prot: u32,
        flags: u32,
        fd: usize,
        offset: usize,
    ) -> AlienResult<isize>;
    fn do_munmap(&self, start: usize, len: usize) -> AlienResult<isize>;
    fn do_sigaction(&self, signum: u8, act: usize, oldact: usize) -> AlienResult<isize>;
    fn do_sigprocmask(&self, how: usize, set: usize, oldset: usize) -> AlienResult<isize>;
    fn do_fcntl(&self, fd: usize, cmd: usize) -> AlienResult<(InodeID, usize)>;
    fn do_prlimit(
        &self,
        pid: usize,
        resource: usize,
        new_limit: usize,
        old_limit: usize,
    ) -> AlienResult<isize>;
    fn do_dup(&self, old_fd: usize, new_fd: Option<usize>) -> AlienResult<isize>;
    fn do_pipe2(&self, r: InodeID, w: InodeID, pipe: usize) -> AlienResult<isize>;
    fn do_exit(&self, exit_code: isize) -> AlienResult<isize>;
    fn do_mmap_device(&self, phy_addr_range: Range<usize>) -> AlienResult<isize>;
    fn do_set_priority(&self, which: i32, who: u32, priority: i32) -> AlienResult<()>;
    fn do_get_priority(&self, which: i32, who: u32) -> AlienResult<i32>;
    fn do_signal_stack(&self, ss: usize, oss: usize) -> AlienResult<isize>;
    fn do_mprotect(&self, addr: usize, len: usize, prot: u32) -> AlienResult<isize>;
    fn do_load_page_fault(&self, addr: usize) -> AlienResult<()>;
    fn do_futex(
        &self,
        uaddr: usize,
        futex_op: u32,
        val: u32,
        timeout: usize,
        uaddr2: usize,
        val3: u32,
    ) -> AlienResult<isize>;
}
```

`TaskDomain`域的接口用于对当前任务调用各种操作

实现：domains/common/task

该模块中包含了所有任务管理相关的内容，并对外实现了`TaskDomain`域的接口。

该模块使用了与Linux相似的模式，使用相同的结构`Task`表示进程和线程，将进程实现为线程组的主线程。

该模块也应用了`TaskMeta`中的任务信息。

### `domain-lib/task_meta`

包含任务的基本信息（tid、状态、上下文）和调度信息。还使用了`TaskOperation`和`OperationResult`定义了一组任务相关操作。

### `kernel/src/task`

负责任务的调度和运行。与`TaskDomain`的区别可能是，前者负责管理系统中的所有任务，后者负责对单个进程的各项操作？

此处实现了线程式的任务切换代码。