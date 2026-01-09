### 交叉编译

### 工具

### 概念

#### Thumb 指令

ARM 处理器有两种主要运行模式：

- **ARM 模式**：指令长度固定为 32 位（4 字节）。
- **Thumb 模式**：指令长度通常为 16 位（2 字节），效率更高，内存占用更小。

Thumb指令更短，一般来说可用的Gadget更多，通过如下命令搜索Thumb指令

```shell
ROPgadget --binary libc-2.21.so --thumb
```

ARM 处理器的程序状态寄存器（CPSR）中有一个 **T 位（Thumb bit）**。

- 如果 T=0，CPU 认为下一条指令是 32 位的 ARM 指令。
- 如果 T=1，CPU 认为下一条指令是 16 位的 Thumb 指令

`BLX r5` r5的值是`0x00003738 + 1`，这里的+1就表示T=1

### 汇编

#### 寄存器

| 寄存器     | 别名                                         | 用途说明                                                  |
| ---------- | -------------------------------------------- | --------------------------------------------------------- |
| `R0`–`R3`  | 无                                           | 用于函数调用时传递参数（前 4 个参数）或返回值（R0）       |
| `R4`–`R11` | 无                                           | 被称为 "callee-saved"，函数调用时通常要保存和恢复的寄存器 |
| `R12`      | `IP` (Intra-Procedure-call scratch register) | 中间过程使用的临时寄存器                                  |
| `R13`      | `SP` (Stack Pointer)                         | 指向当前栈顶                                              |
| `R14`      | `LR` (Link Register)                         | 保存函数调用返回地址                                      |
| `R15`      | `PC` (Program Counter)                       | 当前正在执行的指令地址                                    |

#### 指令

| 指令                     | 含义                                        |
| ------------------------ | ------------------------------------------- |
| `PUSH {R11, LR}`         | 保存 R11 和返回地址 LR 到栈中（函数调用前） |
| `POP {R11, PC}`          | 恢复 R11 和程序计数器（返回到调用者）       |
| `ADD Rn, Rm, #imm`       | Rn ← Rm + imm，立即数加法                   |
| `SUB Rn, Rm, #imm`       | Rn ← Rm - imm，立即数减法                   |
| `MOV Rd, Rm/#imm`        | 把寄存器或立即数复制到目的寄存器            |
| `STR Rd, [Rn, #offset]`  | 把 Rd 的值存到 Rn + offset 地址上（word）   |
| `LDR Rd, [Rn, #offset]`  | 从地址 Rn + offset 读取 word 到 Rd          |
| `STRB Rd, [Rn, #offset]` | 存储一个字节到地址 Rn + offset              |
| `LDRB Rd, [Rn, #offset]` | 读取一个字节从地址 Rn + offset 到 Rd        |
| `CMP Rn, #imm`           | 比较 Rn 和立即数，设置标志（Z/N）           |
| `BNE/BGT/BEQ/B`          | 条件跳转：不等于/大于/等于/无条件跳转       |
| `BL func`                | 调用函数（并保存返回地址到 LR）             |
| `UXTB Rd, Rm`            | 扩展低字节为无符号到 32-bit                 |
| `EOR Rd, Rm, #imm`       | 按位异或                                    |
| `LSR Rd, Rm, #imm`       | 逻辑右移                                    |
| `MOV R0, Rn`             | 通常用于设置函数参数                        |

#### 函数调用和参数传递

| 参数编号  | 使用寄存器 |
| --------- | ---------- |
| 第1个参数 | R0         |
| 第2个参数 | R1         |
| 第3个参数 | R2         |
| 第4个参数 | R3         |

超过 4 个参数的，其余参数会**从右到左**依次压栈

参数按 **word 对齐**（4 字节对齐）

1. 返回值放在 `R0` 中（基本类型）

- `int`, `char`, `pointer` 等类型直接用 `R0`
- `long long`、`double` 用 `R0:R1` 组成 64 位

2. 返回结构体（大小 ≤ 4 个字）的方式：

- 如果结构体大小 ≤ 4 字节，可以直接用 `R0` 返回
- 如果结构体大小 > 4 字节，则**调用者提供缓冲区地址**，传递给 callee 的第一个参数，callee 写入该结构体

#### 跳转指令

```assembly
# PC = target,LR = 下一条指令地址，不支持ARM / Thumb切换；类似于call
BL sub_1234

# PC = target,LR = 下一条指令地址，支持ARM / Thumb切换
# BLX支持 BLX 立即数; 和BLX 寄存器; 两种形式
BLX target

# 只跳转 不保存返回地址 不切换状态
B Rm

# 跳转到 Rm；根据 bit0 切换 ARM/Thumb；不保存 LR
BX Rm

# 从栈中加载 PC；根据 PC bit0 切换 ARM/Thumb
POP {R0, R1, PC}

# 和 pop {..., pc} 本质相同；ARM 状态常见
LDMIA SP!, {R4, R5, PC}

# 不会切换状态
MOV PC, R3

# 从内存加载 PC；根据 bit0 切换状态
LDR PC, [SP, #0x10]
```

