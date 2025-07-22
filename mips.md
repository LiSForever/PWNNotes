### 汇编MIPS

#### 指令

```assembly
la   $t9, func ; 加载func地址到$t9
jalr $t9  ; 跳转到函数

```

#### **1. `lw`（Load Word）**

- **作用**：从内存中读取一个32位字（word）到寄存器中。
- **语法**：`lw $reg, offset($base)`：从`$base + offset`地址加载数据到`$reg`。

#### **2. `sw`（Store Word）**

- **作用**：将寄存器中的一个32位字存储到内存中。
- **语法**：`sw $reg, offset($base)`：将`$reg`中的值存储到`$base + offset`地址。

#### **3. `li`（Load Immediate）**

- **作用**：将一个立即数常量加载到寄存器中。
- **语法**：`li $reg, value`

#### **4. `la`（Load Address）**

- **作用**：加载一个地址（或标签）到寄存器。
- **语法**：`la $reg, label`

#### **5. `move`**

- **作用**：将一个寄存器的值复制到另一个寄存器。
- **语法**：`move $dst, $src`

#### **6. `addiu`（Add Immediate Unsigned）**

- **作用**：向寄存器中的值添加一个立即数（无符号），结果存入目标寄存器。
- **语法**：`addiu $dst, $src, imm`

#### **7. `addu`（Add Unsigned）**

- **作用**：无符号整数相加（无溢出检测）。
- **语法**：`addu $dst, $src1, $src2`

#### **8. `jalr`（Jump and Link Register）**

- **作用**：跳转到函数地址（寄存器中），并将返回地址保存在`$ra`。
- **语法**：`jalr $reg`

#### **9. `jr`（Jump Register）**

- **作用**：跳转到`$reg`指定的地址（用于函数返回）。
- **语法**：`jr $reg`

#### **10. `beqz`（Branch if Equal to Zero）**

- **作用**：如果寄存器为0则跳转。
- **语法**：`beqz $reg, label`

#### **11. `bnez`（Branch if Not Equal to Zero）**

- **作用**：如果寄存器不为0则跳转。
- **语法**：`bnez $reg, label`

#### **12. `bne`（Branch if Not Equal）**

- **作用**：两个寄存器的值不相等时跳转。
- **语法**：`bne $reg1, $reg2, label`

#### **13. `li` / `addiu`**

- 两者都能用于加载立即数，`li` 是伪指令，底层可能转换成 `addiu` 或 `ori` 等。

#### **14. `mflo`**

- **作用**：从乘除法指令结果寄存器 LO 中获取值。
- 通常用于乘除之后获取结果。

#### **15. `divu`**

- **作用**：无符号除法，结果保存在 `LO` 和 `HI`。
- **语法**：`divu $rs, $rt` → `$LO = $rs / $rt`, `$HI = $rs % $rt`

#### **16. `break`**

- **作用**：产生断点异常，用于调试或非法操作处理。
- **语法**：`break code`

#### 参数传递