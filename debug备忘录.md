在ELF共享库（.so）中，`.dynsym`段中的符号若具有`Ndx=UND`（未定义符号），通常意味着这些符号需要在动态链接时从其他共享库中解析。而`Dynamic Section`中的`NEEDED`项（`DT_NEEDED`）则声明了该库的显式依赖关系。若二者不一致（存在`UND`符号但无`NEEDED`项），可能由以下原因导致：

（DeepSeek）

---

### **1. 符号由主程序提供**
- **场景**：共享库依赖的符号由**主程序（可执行文件）直接提供**，而非其他共享库。
- **原理**：动态链接器在加载共享库时，会优先从主程序的全局符号表中解析未定义符号。若主程序已定义这些符号，共享库无需声明外部依赖（无需`NEEDED`项）。
- **验证**：检查共享库的未定义符号是否存在于主程序的符号表（如`readelf -s ./main`）。

---

### **2. 链接器选项允许未定义符号**
- **场景**：链接时通过参数**抑制了依赖项的自动添加**或**允许未解析符号存在**。
- **常见选项**：
  - **`--unresolved-symbols=ignore-all`**：忽略未定义符号，强制生成共享库，即使符号未解析。
  - **`--no-add-needed`** 或 **`--as-needed`**（误用）：抑制自动添加依赖库的`NEEDED`项（需结合其他条件触发）。
- **示例**：
  ```bash
  gcc -shared -o libfoo.so foo.c -Wl,--unresolved-symbols=ignore-all
  ```

---

### **3. 运行时动态加载（dlopen）**
- **场景**：共享库的依赖在**运行时通过`dlopen`手动加载**，而非静态声明。
- **原理**：编译时未通过`-l`指定依赖库，因此`NEEDED`项缺失，但代码中存在对动态加载库符号的引用（如`dlsym`）。
- **影响**：动态链接器在初始加载时不会检查这些符号，依赖程序在运行时正确加载库。

---

### **4. 符号可见性或版本控制**
- **场景**：符号的版本或可见性设置导致依赖未被正确记录。
- **可能性**：
  - **符号版本脚本**：通过版本脚本隐藏了某些依赖关系。
  - **`__attribute__((visibility("hidden")))`**：符号可见性设置异常，但通常不会影响`UND`符号的依赖关系。

---

### **5. 人为修改或链接错误**
- **场景**：ELF文件被手动修改（如删除`NEEDED`项）或链接流程存在缺陷。
- **工具影响**：使用`patchelf`等工具删除`NEEDED`项，但未清理`.dynsym`中的符号引用。

---

### **总结**
最常见的原因是**符号由主程序提供**或**链接器选项允许未定义符号存在**。例如，若共享库设计为与特定主程序配合使用，主程序导出符号供库使用，则无需声明外部依赖。此外，开发者可能有意忽略未定义符号（如测试或特殊场景），需结合具体构建命令和上下文分析。




-static
           Do  not link against shared libraries.  This is only meaningful on platforms for which shared libraries are supported.  The
           different variants of this option are for compatibility with various systems.  You may use this option  multiple  times  on
           the   command   line:   it  affects  library  searching  for  -l  options  which  follow  it.   This  option  also  implies
           --unresolved-symbols=report-all.  This option can be used with -shared.  Doing so means that  a  shared  library  is  being
           created but that all of the library's external references must be resolved by pulling in entries from static libraries.

--no-undefined
-z defs
           Report unresolved symbol references from regular object files.  This is done even if the linker is creating a  non-symbolic
           shared  library.   The switch --[no-]allow-shlib-undefined controls the behaviour for reporting unresolved references found
           in shared libraries being linked in.

           The effects of this option can be reverted by using "-z undefs".

--allow-shlib-undefined
--no-allow-shlib-undefined
           Allows  or  disallows  undefined  symbols  in  shared  libraries.   This switch is similar to --no-undefined except that it
           determines the behaviour when the undefined symbols are in a shared library rather than a regular object file.  It does not
           affect how undefined symbols in regular object files are handled.

           The default behaviour is to report errors for any undefined symbols referenced in shared libraries if the linker  is  being
           used to create an executable, but to allow them if the linker is being used to create a shared library.

           The reasons for allowing undefined symbol references in shared libraries specified at link time are that:

           •   A  shared  library  specified at link time may not be the same as the one that is available at load time, so the symbol
               might actually be resolvable at load time.

           •   There are some operating systems, eg BeOS and HPPA, where undefined symbols in shared libraries are normal.

               The BeOS kernel for example patches shared libraries at load time to select whichever function is most appropriate  for
               the current architecture.  This is used, for example, to dynamically select an appropriate memset function.

在设置rustc参数 -C target-feature=+crt-static 后，编译器报错target riscv64gc-unknown-linux-musl does not support crate type "cdylib"，但在不设置上述参数后，编译器不报错。原因是？

（ChatGPT）

这是由于使用了静态链接 C 运行时（crt-static）与生成动态库（cdylib）的本质要求之间存在冲突造成的。具体来说：

- **动态库（cdylib）的要求**  
  cdylib 用于生成供其他语言调用的共享库，其设计前提是依赖动态链接的 C 运行时。如果不指定 crt-static，Rust 会采用目标默认的动态链接方式，从而生成正常的 cdylib。

- **启用 `+crt-static` 的影响**  
  当你使用 `-C target-feature=+crt-static` 时，Rust 会强制静态链接 C 运行时。这在静态可执行文件（或某些库）中是允许的，但对于动态库来说则不合适，因为动态库要求使用共享（动态）CRT。而在目标 `riscv64gc-unknown-linux-musl` 上，静态链接 CRT 与生成 cdylib 不兼容，因此编译器报错说该 target 不支持 cdylib。

简单来说，启用 `crt-static` 会改变链接模型，使得编译器尝试以静态方式链接 C 运行时，但动态库本质上需要动态链接环境。这两者冲突导致了错误提示。

相关讨论可以参考 GitHub 上关于 musl、crt-static 和 cdylib 的 issue（例如 rust-lang/rust#82521）https://github.com/rust-lang/rust/issues/82521。

使用Rust语言编写的程序，target为riscv64gc-unknown-linux-musl、crate-type为cdylib时，为何编译产生的so文件的dynsym段会出现符号memcpy和memmove？

deepseek:

在使用Rust编译为`riscv64gc-unknown-linux-musl`目标并生成`cdylib`类型的动态库时，生成的`.so`文件的`dynsym`段中出现`memcpy`和`memmove`符号，主要原因如下：

---

### 1. **编译器优化与内存操作**
   - Rust编译器（通过LLVM后端）在优化代码时，**可能将大块内存复制操作（如`copy_from_slice`或结构体复制）替换为`memcpy/memmove`调用**。这是LLVM的通用优化策略，旨在利用高效的C标准库函数提升性能。
   - 例如，以下Rust代码可能触发此优化：
     ```rust
     let src = [0u8; 1024];
     let mut dst = [0u8; 1024];
     dst.copy_from_slice(&src); // 可能被优化为memcpy
     ```

---

### 2. **动态库的符号依赖（`cdylib`特性）**
   - 当编译目标为`cdylib`时，Rust会**动态链接到系统的C库（如musl libc）**，而非静态链接。这意味着动态库需要声明其依赖的外部符号（如`memcpy/memmove`），以便在运行时从宿主环境的C库中解析。
   - 尽管`riscv64gc-unknown-linux-musl`目标通常与静态链接musl关联，但`cdylib`的构建过程可能默认采用动态链接行为，导致符号出现在`dynsym`段中。

---

### 3. **目标平台的LLVM后端行为**
   - RISC-V架构的LLVM后端**可能更倾向于生成对`memcpy/memmove`的调用**，而非内联复制指令（尤其是对于较大内存块）。这种选择取决于目标架构的性能特性和ABI约定。
   - 即使Rust代码未显式调用这些函数，LLVM仍可能在IR（Intermediate Representation）层插入它们。

---

### 4. **musl libc的链接方式**
   - 若目标配置为动态链接musl（即使musl通常静态链接），或存在混合链接行为，`memcpy/memmove`可能被标记为动态符号。检查编译时的链接命令（通过`RUSTFLAGS="-Clink-args=-v"`）可确认实际链接方式。
   - 若需强制静态链接，可通过Cargo配置（如`.cargo/config.toml`）指定`target-feature=+crt-static`，但需确保目标支持。

---

### 验证与解决方案

#### 验证方法
- **检查生成的汇编代码**：
  ```bash
  RUSTFLAGS="--emit asm" cargo build --release
  ```
  搜索`memcpy/memmove`调用，确认是否由编译器插入。

- **查看动态符号表**：
  ```bash
  readelf --dyn-syms target/riscv64gc-unknown-linux-musl/release/libyourlib.so
  ```

#### 解决方案
- **强制静态链接musl**：
  ```toml
  # .cargo/config.toml
  [target.riscv64gc-unknown-linux-musl]
  rustflags = ["-C", "target-feature=+crt-static"]
  ```
  需确保工具链支持静态链接。

- **使用`#![no_std]`避免依赖libc**：
  ```rust
  #![no_std]
  ```
  适用于无标准库场景，但需自行处理内存操作或依赖`compiler_builtins`。

- **手动实现内存操作**：
  避免触发编译器的`memcpy/memmove`优化，如使用逐字节复制循环（可能影响性能）。

---

### 总结
`dynsym`段中出现`memcpy/memmove`符号是Rust编译器优化与动态链接musl libc共同作用的结果。若需消除这些符号，需调整链接方式（静态链接）或修改代码生成策略。

## 2.11工作：

1. 确认了调用memmove和memcpy的来源：buddy_system_allocator库。不过，参考rCore-N的实现，对这两个函数的调用在很多地方都会出现，因此单纯替换buddy_system_allocator库可能作用不大。
2. 测试了修改target为riscv64gc-unknown-none-elf、crate-type为bin的解决方案（因为riscv64gc-unknown-none-elf不支持cdylib）：发现如果使用bin，则无法将库中的pub函数编译为global符号。可能的解决方案是像rCore-N一样，在main函数中返回各个函数的地址。此外，不清楚修改了crate-type后，还是否是位置无关代码。
3. 之后的解决方案：要么在内核引入libc运行时，并且修改elf解析过程使其可以动态链接到内核的libc；要么采用2中提到的方式；或者，再探索一下能否让代码静态链接libc。

链接器输出：

LC_ALL="C" PATH="/home/rosy/.rustup/toolchains/nightly-2024-11-05-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/bin:/home/rosy/.vscode-server/cli/servers/Stable-33fc5a94a3f99ebe7087e8fe79fbe1d37a251016/server/bin/remote-cli:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/home/rosy/桌面/qemu-7.0.0/build:/home/rosy/.cargo/bin:/opt/riscv/musl-riscv-toolchain-8.2.0-1/bin:/home/rosy/桌面/qemu-7.0.0/build:/home/rosy/.cargo/bin:/opt/riscv/musl-riscv-toolchain-8.2.0-1/bin" VSLANG="1033" "riscv64-linux-musl-ld" "--version-script=/tmp/rustctkMhBC/list" "--no-undefined-version" "/tmp/rustctkMhBC/symbols.o" "/home/rosy/\xe6\xa1\x8c\xe9\x9d\xa2/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/debug/deps/cops.buddy_system_allocator-e82c14ef57639414.buddy_system_allocator.ae1c67a15fd69d0e-cgu.0.rcgu.o.rcgu.o" "--as-needed" "-Bstatic" "/home/rosy/.rustup/toolchains/nightly-2024-11-05-x86_64-unknown-linux-gnu/lib/rustlib/riscv64gc-unknown-linux-musl/lib/libcompiler_builtins-3b2eaf9b992f974e.rlib" "-Bdynamic" "--eh-frame-hdr" "-z" "noexecstack" "-L" "/home/rosy/.rustup/toolchains/nightly-2024-11-05-x86_64-unknown-linux-gnu/lib/rustlib/riscv64gc-unknown-linux-musl/lib" "-o" "/home/rosy/\xe6\xa1\x8c\xe9\x9d\xa2/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/debug/deps/libcops.so" "--gc-sections" "-shared" "-z" "relro" "-z" "now" "-fuse-ld=lld" "-T/home/rosy/\xe6\xa1\x8c\xe9\x9d\xa2/async-os/vdso/cops/cops.lds" "-soname=libcops.so" "-fPIC" "--no-undefined"

有`--emit asm`下的rustc调用命令：

Running `/home/rosy/.rustup/toolchains/nightly-2024-11-05-x86_64-unknown-linux-gnu/bin/rustc --crate-name cops --edition=2021 vdso/cops/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts,future-incompat --diagnostic-width=138 --crate-type cdylib --emit=dep-info,link -C opt-level=3 -C lto --check-cfg 'cfg(docsrs)' --check-cfg 'cfg(feature, values())' -C metadata=1271e2ed9383d1d6 --out-dir '/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps' --target riscv64gc-unknown-linux-musl -C linker=riscv64-linux-musl-ld -C strip=debuginfo -L 'dependency=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps' -L 'dependency=/home/rosy/桌面/async-os/vdso/cops/../target/release/deps' --extern 'buddy_system_allocator=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps/libbuddy_system_allocator-bbd8d22146bd9e65.rlib' --extern 'log=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps/liblog-37b59ca86958296b.rlib' --extern 'spinlock=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps/libspinlock-c84884ecfa4d061d.rlib' --emit asm -C link-arg=-fuse-ld=lld -C 'link-arg=-T/home/rosy/桌面/async-os/vdso/cops/cops.lds' -C link-arg=-soname=libcops.so -C link-arg=-fPIC -C link-arg=--no-undefined`

无`--emit asm`下的rustc调用命令：

Running `/home/rosy/.rustup/toolchains/nightly-2024-11-05-x86_64-unknown-linux-gnu/bin/rustc --crate-name cops --edition=2021 vdso/cops/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts,future-incompat --diagnostic-width=138 --crate-type cdylib --emit=dep-info,link -C opt-level=3 -C lto --check-cfg 'cfg(docsrs)' --check-cfg 'cfg(feature, values())' -C metadata=1271e2ed9383d1d6 --out-dir '/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps' --target riscv64gc-unknown-linux-musl -C linker=riscv64-linux-musl-ld -C strip=debuginfo -L 'dependency=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps' -L 'dependency=/home/rosy/桌面/async-os/vdso/cops/../target/release/deps' --extern 'buddy_system_allocator=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps/libbuddy_system_allocator-bbd8d22146bd9e65.rlib' --extern 'log=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps/liblog-37b59ca86958296b.rlib' --extern 'spinlock=/home/rosy/桌面/async-os/vdso/cops/../target/riscv64gc-unknown-linux-musl/release/deps/libspinlock-c84884ecfa4d061d.rlib' -C link-arg=-fuse-ld=lld -C 'link-arg=-T/home/rosy/桌面/async-os/vdso/cops/cops.lds' -C link-arg=-soname=libcops.so -C link-arg=-fPIC -C link-arg=--no-undefined`