# RISC-V 裸机启动排错（Avatar OS lesson97/98）

> 目的：记录「编译成功但 QEMU 不输出 Hello World」的完整排查，提炼裸机启动的两个经典坑和一套排错方法。下次遇到「裸机程序编译过了跑不起来」，直接套用。

## 结论（TL;DR）

裸机程序「编译通过 ≠ 能跑」，本次失败是两个坑叠加：

1. `la` 伪指令默认走 GOT（PIC），裸机没有 ELF 加载器填充 GOT → 读到垃圾地址 → bss 清零 `sd` 越界、循环不收敛（死循环）。
2. `sp` 从未初始化，`main` 调子函数要压栈存返回地址 → 栈操作写非法地址崩溃。

## 现象

`make all` 无任何报错，QEMU 运行后不输出 Hello World，程序卡死（只剩 timeout 的消息）。

## 排错流程（三件套，层层缩小范围）

1. **先看历史线索**：`git log` 发现 commit 已写着「lesson98存在BSS sd越界死循环bug」，直接把怀疑范围锁到 bss 清零循环。排错第一步：先看有没有人（或过去的自己）已经踩过。
2. **指令轨迹定位卡点**：`qemu-system-riscv64 ... -d in_asm -D log`，逐条看指令执行轨迹，发现 PC 在 bss 清零循环里反复跳，把「卡死」从抽象感觉变成精确的指令地址。
3. **反汇编对照预期**：`riscv64-linux-gnu-objdump -d kernel.elf`，发现 `la a0, sbss` 被翻成 `ld`（从 GOT 加载）而非 `auipc+addi`（直接取地址）。拿「实际编译产物」对照「本该长什么样」，根因暴露。

## 根因

| Bug | 编译器假设 | 裸机现实 |
|---|---|---|
| `la` 走 GOT | 默认 PIC，`la` 从 `_GLOBAL_OFFSET_TABLE_` 加载地址，靠 ELF 加载器填表 | 无加载器，GOT 表项是垃圾值 → bss 边界错误 |
| `sp` 未初始化 | `main` 调子函数要压栈，编译器默认 libc/OS 已备好栈 | sp=0，`sd ra,8(sp)` 写非法地址崩溃 |

## 修改对照

| 文件 | 错误 | 修正 |
|---|---|---|
| boot.S | `la` 默认 PIC → 走 GOT | 加 `.option nopic`，`la` 展开成 `auipc+addi` |
| boot.S | 从不初始化 `sp` | 加 `la sp, stack_top` |
| linker.ld | 没留栈空间 | bss 后 `. += 0x1000` 并导出 `stack_top` |

## 额外坑：`%hi/%lo` 配 `auipc` 是错误组合（lesson97）

lesson97 手写 `auipc + %hi/%lo` 做「手动寻址」，但 `auipc` 是 **PC-relative** 指令，必须配 `%pcrel_hi/%pcrel_lo`；`%hi/%lo` 是**绝对寻址**重定位（`R_RISCV_HI20`），混用直接报 `relocation truncated to fit`。另外 `%lo` 的符号还写反了（`a0` 配了 `ebss`、`a1` 配了 `sbss`）。

结论：**别手写 `%hi/%lo`，统一用 `.option nopic + la`**，交给汇编器自动生成正确的 PC-relative 寻址。

## 可复用方法论

- **裸机排错三件套**：`git log`（历史线索）→ `qemu -d in_asm`（指令轨迹）→ `objdump -d`（编译产物对照预期）。三者覆盖「历史 / 运行时 / 静态」三个维度。
- **两个心智模型**，遇到「编译过但跑不起来」先查这两处：
  1. 有没有依赖 loader 的东西（GOT / PIE / 全局变量初始化）；
  2. 运行时环境（`sp`、异常向量表）是不是自己搭的。
- **「能编译」是假阳性信号**：编译器只保证语法和链接正确，不保证运行时环境存在，这是裸机编程最危险的地方。
- **统一启动模板**（已落地 lesson97/98，之后每课复用）：

```asm
    .option nopic
    .section .text
    .global _start
_start:
    la a0, sbss          # bss 起始
    la a1, ebss          # bss 结束
bss_loop:
    bgeu a0, a1, bss_done
    sd zero, 0(a0)
    addi a0, a0, 8
    j bss_loop
bss_done:
    la sp, stack_top     # 设栈
    jal ra, main
    j .
```

```ld
SECTIONS {
    . = 0x80000000;      /* qemu virt DRAM 基址 */
    .text   : { *(.text.boot) *(.text) }
    .rodata : { *(.rodata) }
    .data   : { *(.data) }
    sbss = .;
    .bss    : { *(.bss) }
    ebss = .;
    . = ALIGN(16);
    . += 0x1000;         /* 4KB 栈 */
    stack_top = .;
}
```

> 环境：qemu-system-riscv64 `-machine virt -nographic -bios none -kernel kernel.bin`；UART 地址 `0x10000000`（NS16550）；交叉工具链 `riscv64-linux-gnu-gcc`（`-march=rv64g -mabi=lp64 -ffreestanding -nostdlib`）。
