# 在Rust内联汇编中操作段

时间：2025.3.26

## 

在改共享调度器的问题中，发现Rust内联汇编和段操作的不兼容。因此编写了一个小程序进行测试：

```asm
# src/test.S
.section testsection
.balign 0x1000
.globl test_value
test_value:
    .quad 0x0123456789abcdef
    .balign 0x1000
.previous
```

```Rust
// src/main.rs
use std::arch::global_asm;

global_asm!(include_str!("test.S"));

extern "C" {
    fn test_value();
}

fn main() {
    let test_value: usize = unsafe {*(test_value as *const () as *const usize)};
    println!("test_value is {test_value:#018x}");
}
```

结果：当内联汇编中使用了段操作语句时，获得了错误的结果（0x00010102464c457f）。如果没有使用段操作语句，则可以获得正确的结果（0x0123456789abcdef）。

之后，为了研究这样的错误是否由内联引起，修改了构建过程，单独编译汇编语言文件并在链接阶段链接到Rust代码。

```makefile
# Makefile
run:
	as src/test.S -o test.o
	ar rcs libtest.a test.o
	RUSTFLAGS="-Clink-arg=-L -Clink-arg=./" cargo run
```

```Rust
// src/main.rs
// global_asm!(include_str!("test.S"));

#[link(name = "test", kind = "static")]
extern "C" {
    fn test_value();
}
```

结果：在这样的构建流程下，无论汇编代码里是否有段操作语句，均获得了错误的结果（0x00010102464c457f）