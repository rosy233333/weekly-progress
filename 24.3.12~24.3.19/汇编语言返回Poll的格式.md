# RISC-V汇编语言返回`Poll`的格式

时间：2024/3/14

对于`Poll<i32>`：

- 返回`Pending`：`a0`为1，`a1`为任意值
- 返回`Ready(n)`：`a0`为0，`a1`为n

反汇编代码：

```
0000000000011172 <_ZN8asm_test9fun_test117h4bd09dbab6012735E>:
; fn fun_test1() -> Poll<i32> {
   11172: 41 11         addi    sp, sp, -16
   11174: 05 45         li      a0, 1
;     return Poll::Pending;
   11176: 2a c4         sw      a0, 8(sp)
; }
   11178: 22 45         lw      a0, 8(sp)
   1117a: b2 45         lw      a1, 12(sp)
   1117c: 41 01         addi    sp, sp, 16
   1117e: 82 80         ret

0000000000011180 <_ZN8asm_test9fun_test217hce14321704681e18E>:
; fn fun_test2() -> Poll<i32> {
   11180: 41 11         addi    sp, sp, -16
   11182: 25 45         li      a0, 9
;     return Poll::Ready(9);
   11184: 2a c6         sw      a0, 12(sp)
   11186: 01 45         li      a0, 0
   11188: 2a c4         sw      a0, 8(sp)
; }
   1118a: 22 45         lw      a0, 8(sp)
   1118c: b2 45         lw      a1, 12(sp)
   1118e: 41 01         addi    sp, sp, 16
   11190: 82 80         ret
```