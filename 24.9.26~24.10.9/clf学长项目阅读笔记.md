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

之后出现了如下错误，还未解决：

```
          riscv64-linux-musl-gcc: error: crt1.o: No such file or directory
          riscv64-linux-musl-gcc: error: crti.o: No such file or directory
          riscv64-linux-musl-gcc: error: crtbegin.o: No such file or directory
          riscv64-linux-musl-gcc: error: crtend.o: No such file or directory
          riscv64-linux-musl-gcc: error: crtn.o: No such file or directory
```

## Alien系统上和任务调度相关的内容

目前已知，有`Scheduler`和`Task`两个域，还涉及`domain-lib/task_meta`模块和`kernel/src/task`。

还没来得及详细调研。