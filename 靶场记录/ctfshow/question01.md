* **注**：没有使用官方提供的虚拟机

下载文件，file命令分析

```shell
file ctfshow01
# ctfshow01: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=293e5bcd80a7a5a11005d2eb5af05a058db744ca, not stripped
# LSB 目前感觉不重要
# shared object 动态链接库 .so文件
# TODO：version 1 (SYSV) 暂未查询到相关资料，猜测为elf版本
# stripped 和 not stripped ：stripped剔除符号表的信息，使得编译出的文件更小，后者则相反，便于调试
```

这里得知了该文件的架构，得知该文件为动态链接库文件

readelf分析

```shell
readelf -h ctfshow01
# ELF Header:
#  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
#  Class:                             ELF64
#  Data:                              2's complement, little endian
#  Version:                           1 (current)
#  OS/ABI:                            UNIX - System V
#  ABI Version:                       0
#  Type:                              DYN (Shared object file)
#  Machine:                           Advanced Micro Devices X86-64
#  Version:                           0x1
#  Entry point address:               0x640
#  Start of program headers:          64 (bytes into file)
#  Start of section headers:          10720 (bytes into file)
#  Flags:                             0x0
#  Size of this header:               64 (bytes)
#  Size of program headers:           56 (bytes)
#  Number of program headers:         9
#  Size of section headers:           64 (bytes)
#  Number of section headers:         29
#  Section header string table index: 28

# 顺序与上面对应
# 与很多文件一样，elf文件的开头也有magic number来标识它是一个elf文件
# 
```

