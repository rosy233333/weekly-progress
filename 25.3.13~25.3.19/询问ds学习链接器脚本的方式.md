# 如何学习编写链接脚本？

学习编写链接脚本（Linker Scripts）需要理解链接器的工作原理、目标文件的组织方式以及内存布局的规划。以下是逐步的学习路径和资源建议：

---

### **1. 基础知识准备**
- **理解链接器的作用**  
  链接器（如 GNU `ld`）负责将多个目标文件（`.o`）和库合并为可执行文件或库，处理符号解析、地址分配、段（Sections）合并等任务。
- **目标文件结构**  
  学习目标文件的段（`.text`, `.data`, `.bss` 等）、符号表（Symbol Table）、重定位表等概念，推荐阅读《程序员的自我修养——链接、装载与库》。
- **内存布局**  
  了解程序在内存中的布局（代码段、数据段、堆栈等），以及嵌入式系统中常见的 Flash 和 RAM 分区。

---

### **2. 官方文档与语法学习**
- **GNU LD 官方文档**  
  [GNU Linker (LD) Manual](https://sourceware.org/binutils/docs/ld/) 是最权威的参考，涵盖语法、命令和示例。
- **关键语法元素**  
  - **ENTRY()**：指定程序入口点。
  - **MEMORY**：定义内存区域（如 Flash、RAM）及其地址范围。
  - **SECTIONS**：控制段的分配和布局。
  - 符号（Symbols）操作：如 `PROVIDE`, `DEFINED` 等。
  - 位置计数器（`.`）和地址对齐（`ALIGN`）。

---

### **3. 从简单示例入手**
#### **示例 1：基本脚本**
```ld
ENTRY(_start)          /* 入口符号 */

MEMORY {
    ROM (rx) : ORIGIN = 0x08000000, LENGTH = 256K
    RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 64K
}

SECTIONS {
    .text : {
        *(.text*)      /* 合并所有 .text 段到 ROM */
    } > ROM

    .data : {
        *(.data*)
    } > RAM AT > ROM   /* 数据段存储在 ROM，运行时拷贝到 RAM */

    .bss : {
        *(.bss*)
    } > RAM
}
```

#### **示例 2：复杂布局**
```ld
SECTIONS {
    . = 0x8000;        /* 设置当前位置计数器 */
    .text : { *(.text) }
    .rodata : { *(.rodata) }
    .data 0x1000000 : { /* 强制 .data 段到 0x1000000 */
        _data_start = .;
        *(.data)
        _data_end = .;
    }
    .bss : {
        *(.bss)
        end = .;       /* 定义符号 'end' */
    }
}
```

---

### **4. 实践与调试方法**
- **编译时指定链接脚本**  
  ```bash
  gcc -T script.ld -o output.elf input.o
  ```
- **查看内存布局**  
  使用 `objdump -h output.elf` 或 `readelf -S output.elf` 检查段分布。
- **反汇编验证**  
  `objdump -d output.elf` 查看代码地址是否符合预期。
- **调试符号问题**  
  若出现未定义符号，检查链接脚本中的段是否遗漏了必要目标文件（如启动文件 `startup.o`）。

---

### **5. 进阶技巧**
- **条件链接**  
  使用 `PROVIDE` 定义备用符号，避免未定义错误：
  ```ld
  PROVIDE(__stack_top = 0x20008000);
  ```
- **覆盖特定文件的段**  
  将某文件的段分配到特定地址：
  ```ld
  .special_section : {
      some_file.o(.text)
  } > ROM
  ```
- **动态计算地址**  
  结合算术和符号定义复杂布局：
  ```ld
  . = ALIGN(4);        /* 4字节对齐 */
  _end_rom = .;        /* 记录当前地址 */
  ```

---

### **6. 常见问题与解决**
- **段重叠（Overlap）**  
  检查 `MEMORY` 定义的长度是否足够，使用 `LENGTH` 限制。
- **未初始化段（.bss）未清零**  
  需在启动代码中手动清零 `.bss`。
- **符号地址错误**  
  确保符号在链接脚本中正确定义，并使用 `extern` 在代码中声明。

---

### **7. 学习资源推荐**
- **书籍**  
  《程序员的自我修养》第 4 章（链接与库）。
- **开源项目参考**  
  研究嵌入式项目的链接脚本（如 [STM32 Cube](https://github.com/STMicroelectronics/STM32CubeF4) 或 [Zephyr OS](https://github.com/zephyrproject-rtos/zephyr)）。
- **在线教程**  
  [The Linker Script Guide](https://www.scoberlin.de/content/media/http/informatik/msc/embedded-systems/ldscript.pdf)（简明教程）。

---

通过以上步骤，逐步掌握链接脚本的编写，并结合实际项目实践，最终能够灵活控制程序的内存布局。
