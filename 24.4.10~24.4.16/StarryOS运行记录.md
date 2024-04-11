# StarryOS运行记录

时间：2024/4/11

## QEMU上运行riscv版本

### 运行脚本

参照项目的`readme.md`，但发现并改正了一些问题。

### 运行 Unikernel 架构内核

由于一个问题，运行 Unikernel 架构内核前需要修改一处代码，也就是`modules/axhal/src/arch/riscv/mod.rs` 的第6行：

```Rust
// 修改前
// pub use trap::first_into_user;

// 修改后
#[cfg(feature = "monolithic")]
pub use trap::first_into_user;
```

此后，无论运行宏内核还是 Unikernel 都不需再次修改此处。

```Bash
# 以 Unikernel 形式启动
make A=[要启动应用的相对路径] ARCH=riscv64 run
```

使用 `A` 选项指定启动的应用。应用都位于 `apps` 目录中。这些应用中，除了 `monolithic_userboot` 用于以宏内核模式启动外，其它应用均可在 Unikernel 模式下启动。

### 运行宏内核

```Bash
# 构建镜像
./build_img.sh sdcard

# 以宏内核形式启动(当前仅支持 riscv 架构)
make ARCH=riscv64 run

# 使用 ramdisk 加载测例并且运行内核，可以显著提高文件 IO 速度
make ARCH=riscv64 FEATURES=img run
```

以上启动命令实际上是使用了 `A=apps/monolithic_userboot` 的默认值（见 `Makefile` ），从而默认以宏内核模式启动。

若需要修改启动应用，则不应修改运行命令的 `A` 参数，而是：

1. （可选）使用 `APP_FEATURES=batch` 开启批量模式（应该就是批量运行多个应用？）（我现在在该模式下运行的时候，退出 shell 之后会报Protection fault错误）
2. 若开启了批量模式，则修改 `apps/monolithic_userboot/src/batch.rs` 中的 `SDCARD_TESTCASES` 常量以改变启动应用。
3. 若未开启批量模式，则修改 `apps/monolithic_userboot/src/main.rs` 中 `main` 函数中的 `init_args` 变量以改变启动应用。