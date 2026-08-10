### 概述

* FILE和`_IO_FILE_plus`的结构如下。由于C++面向对象的底层原理，编译器编译出的对象会存在vtable，vtable指针在对象的0x00偏移，这造成了vtable劫持和参数控制的死锁，而`_IO_FILE_plus`是纯 **C 语言**手动模拟出来的面向对象机制，vtable指针不在开头，结果即可以劫持vtable又可以控制传参。

```c
// 1. 核心文件结构体
struct _IO_FILE {
  int _flags;                /* [0x00] 标志位，FSOP 中通常改写为 "/bin/sh\x00" (作参数) */
  
  /* 以下为缓冲区相关指针，FSOP 绕过 _IO_flush_all_lockp 校验常需构造 write_ptr > write_base */
  char* _IO_read_ptr;        /* [0x08] Current read pointer */
  char* _IO_read_end;        /* [0x10] End of get area. */
  char* _IO_read_base;       /* [0x18] Start of putback+get area. */
  char* _IO_write_base;      /* [0x20] Start of put area. (FSOP中常设为0) */
  char* _IO_write_ptr;       /* [0x28] Current put pointer. (FSOP中常设为1) */
  char* _IO_write_end;       /* [0x30] End of put area. */
  char* _IO_buf_base;        /* [0x38] Start of reserve area. */
  char* _IO_buf_end;         /* [0x40] End of reserve area. */
  
  /* 其他内部字段 */
  char *_IO_save_base;       /* [0x48] */
  char *_IO_backup_base;     /* [0x50] */
  char *_IO_save_end;        /* [0x58] */

  struct _IO_marker *_markers; /* [0x60] */
  
  struct _IO_FILE *_chain;   /* [0x68] 关键！指向链表下一个 FILE 的指针 (用于劫持 _IO_list_all 时的链表伪造) */

  int _fileno;               /* [0x70] 文件描述符 (0, 1, 2 等) */
  int _flags2;               /* [0x74] */
  __off_t _old_offset;       /* [0x78] */

  unsigned short _cur_column; /* [0x80] */
  signed char _vtable_offset; /* [0x82] */
  char _shortbuf[1];         /* [0x83] */

  _IO_lock_t *_lock;         /* [0x88] 锁指针 (FSOP中若触发内部调用，需将其指向可写的合法内存，防止解引用 NULL) */
  
  __off64_t _offset;         /* [0x90] */
  
  struct _IO_codecvt *_codecvt; /* [0x98] */
  struct _IO_wide_data *_wide_data; /* [0xa0] (在 House of Apple 等宽字符攻击中极其关键) */
  struct _IO_FILE *_freeres_list; /* [0xa8] */
  void *_freeres_buf;        /* [0xb0] */
  size_t __pad5;             /* [0xb8] */
  
  int _mode;                 /* [0xc0] 模式，通常需要 <= 0 以绕过校验 */
  
  char _unused2[15 * sizeof (int) - 4 * sizeof (void *) - sizeof (size_t)]; /* 填充至 0xd8 边界 */
}; // 大小到此为 0xd8

// 2. 虚函数表结构体
struct _IO_jump_t {
    size_t __dummy;                  /* [0x00] */
    size_t __dummy2;                 /* [0x08] */
    void (*__finish) (void);         /* [0x10] (其实包含一个额外的参数类型，此处简写) fclose 触发 */
    void (*__overflow) (void);       /* [0x18] exit/abort 全局刷新触发，FSOP 最常用 */
    void (*__underflow) (void);      /* [0x20] fread/gets 触发 */
    void (*__uflow) (void);          /* [0x28] */
    void (*__pbackfail) (void);      /* [0x30] */
    void (*__xsputn) (void);         /* [0x38] puts/printf/fwrite 触发，极为常用 */
    void (*__xsgetn) (void);         /* [0x40] */
    void (*__seekoff) (void);        /* [0x48] */
    void (*__seekpos) (void);        /* [0x50] */
    void (*__setbuf) (void);         /* [0x58] */
    void (*__sync) (void);           /* [0x60] */
    void (*__doallocate) (void);     /* [0x68] */
    void (*__read) (void);           /* [0x70] */
    void (*__write) (void);          /* [0x78] */
    void (*__seek) (void);           /* [0x80] */
    void (*__close) (void);          /* [0x88] */
    void (*__stat) (void);           /* [0x90] */
    void (*__showmanyc) (void);      /* [0x98] */
    void (*__imbue) (void);          /* [0xa0] */
};

// 3. glibc 中实际使用的加长版 FILE 结构体
struct _IO_FILE_plus {
  struct _IO_FILE file;              /* [0x00 - 0xd7] 继承前面的所有字段 */
  const struct _IO_jump_t *vtable;   /* [0xd8] 指向虚函数表的指针 (FSOP 劫持的核心目标) */
};
```

### <=2.23 _IO_validate_vtable引入之前

#### 伪造`_IO_FILE`并挂载到`_IO_list_all`

`_IO_list_all`是是 glibc 在 `.data` 段维护的一个全局指针变量。它的作用是作为**单向链表的头指针**，记录当前进程中打开的所有文件流

```txt
_IO_list_all (全局指针)
      |
      V
[ _IO_2_1_stderr_ ] --(_chain)--> [ _IO_2_1_stdout_ ] --(_chain)--> [ _IO_2_1_stdin_ ] --(_chain)--> NULL
```

据此，我们可以通过如下方式伪造和挂载`_IO_FILE`

* 劫持`_IO_list_all`指向伪造`_IO_FILE`
* 直接篡改`_IO_list_all`链表上合法的`_IO_FILE`
* 将某个合法`_IO_FILE`的`_chain`篡改指向伪造的`_IO_FILE`

#### 触发控制流劫持

| **触发操作 / 场景**                      | **内部调用的虚函数宏**    | **对应 _IO_jump_t 中的函数名** | **虚表索引** | **相对虚表首地址的偏移 (x86_64)** |
| ---------------------------------------- | ------------------------- | ------------------------------ | ------------ | --------------------------------- |
| **`exit()` 或 `abort()` 引起的全局刷新** | `_IO_OVERFLOW(fp, EOF)`   | `__overflow`                   | 3            | **`0x18`**                        |
| **调用 `puts`, `printf`, `fwrite`**      | `_IO_XSPUTN(fp, data, n)` | `__xsputn`                     | 7            | **`0x38`**                        |
| **调用 `fread`, `scanf`, `getchar`**     | `_IO_UNDERFLOW(fp)`       | `__underflow`                  | 2            | **`0x10`**                        |
| **调用 `fclose`**                        | `_IO_FINISH(fp)`          | `__finish`                     | 1            | **`0x08`**                        |
| 调用 `fseek`                             | `_IO_SEEKOFF(...)`        | `__seekoff`                    | 11           | `0x58`                            |
