---
title: 'Xv6 Lab2' # <--- 修改这一行
date: "2026-08-10T20:40:25+08:00"
draft: false
tags: ["xv6", ""]
location: "广州"
---


## GDB 基础命令

### GDB启动：
```shell 1
make CPUS=1 qemu-gdb
```

```shell 2
gdb-multiarch
```
一旦您通过 `make qemu-gdb` 和 `gdb-multiarch` 成功连接上，您就可以在 GDB 的提示符 `(gdb)` 后面输入命令了。以下是一些最常用的命令：

*   **`b` (breakpoint)**: 设置断点。
    *   `b function_name`: 在函数 `function_name` 的入口处设置断点。例如: `b main`。
    *   `b file:line_number`: 在指定文件的指定行号设置断点。例如: `b kernel/proc.c:283`。
    *   `b *address`: 在指定的内存地址设置断点。例如: `b *0x8000000a`。

*   **`c` (continue)**: 继续执行程序，直到遇到下一个断点或程序结束。

*   **`n` (next)**: 执行下一行代码（单步执行）。如果当前行是函数调用，`n` 会执行整个函数然后停在下一行，**不会**进入函数内部。

*   **`s` (step)**: 执行下一行代码。如果当前行是函数调用，`s` 会**进入**该函数内部并停在函数的第一行。

*   **`l` (list)**: 显示源代码。
    *   `l`: 显示当前停止位置附近的源代码。
    *   `l function_name`: 显示指定函数的源代码。
    *   `l file:line_number`: 显示指定文件和行号附近的源代码。

*   **`p` (print)**: 打印变量或表达式的值。
    *   `p variable_name`: 打印变量 `variable_name` 的值。例如: `p p->pid`。
    *   `p/x variable_name`: 以十六进制格式打印变量的值。
    *   `p/t variable_name`: 以二进制格式打印变量的值。
    *   `p *pointer_variable`: 解引用并打印指针指向的内容。

*   **`info`**: 显示程序状态信息。
    *   `info breakpoints` (或 `i b`): 显示所有断点的信息。
    - `info registers` (或 `i r`): 显示所有寄存器的值。

*   **`bt` (backtrace)**: 显示当前的函数调用栈。当程序崩溃或停在断点时，这个命令非常有用，可以帮您追溯代码是如何执行到当前位置的。

*   **`q` (quit)**: 退出 GDB。

### 在 xv6 中调试：一个实例

让我们来看一个实际的例子。假设您想知道 xv6 内核启动时 `main` 函数在哪里被调用。

1.  **启动调试会话**
    *   **终端 1**: `make qemu-gdb`
    *   **终端 2**: `gdb-multiarch`

2.  **设置断点**
    在 GDB 中，设置一个断点在 `main` 函数：
    ```gdb
    (gdb) b main
    ```

3.  **开始执行**
    让内核运行起来：
    ```gdb
    (gdb) c
    ```
    程序会开始执行，然后在进入 `main` 函数时停下来。

4.  **查看调用栈**
    现在您想知道是谁调用了 `main`。使用 `bt` 命令：
    ```gdb
    (gdb) bt
    ```
    您可能会看到类似下面的输出，它显示 `main` 是被 `start` 函数调用的。
    ```
    #0  main () at kernel/main.c:23
    #1  0x00000000800000e4 in start () at kernel/start.c:21
    ```

5.  **查看代码和变量**
    您可以使用 `l` 来查看 `main` 函数周围的代码，或者用 `p` 来查看变量。例如，在 `main` 函数中，您可以打印 `cpuid()` 的值。

### 如何查看汇编代码？

您提到了 `kernel.asm` 文件，这说明您对底层执行细节感兴趣。GDB 也能很好地支持汇编级别的调试。

*   **`layout asm`**: 打开一个文本用户界面 (TUI)，并排显示汇编代码和源代码。
*   **`layout split`**: 同时显示源代码、汇编代码和命令窗口。
*   **`si` (step instruction)**: 单步执行一条汇编指令，而不是一行源代码。
*   **`ni` (next instruction)**: 类似于 `si`，但会跳过函数调用指令。
*   **`disassemble`**: 反汇编指定的函数或内存区域。例如 `disassemble main`。

当您使用 `layout asm` 或 `layout split` 时，一个箭头 `=>` 会指向下一条将要执行的汇编指令。这对于理解编译器如何将 C 代码转换为机器指令非常有帮助。

### 总结

*   从基础命令 `b`, `c`, `n`, `s`, `p`, `bt` 开始。
*   利用 `layout asm` 和 `si` 进行汇编级别的调试。
*   `.gdbinit` 文件已经帮您自动加载了内核符号，所以您可以直接按函数名和变量名进行调试。


## xv6 exec() 函数完整解析

> exec：**加载ELF可执行程序，替换当前进程的用户地址空间，不创建新进程（PID不变）**，fork+exec 才是创建新程序的组合。 fork：复制进程；exec：把进程内容换成新程序。



### 1. 打开可执行文件（inode）

```
ip = namei(path)   // 根据文件路径找到inode
ilock(ip)
```

通过路径拿到磁盘上ELF程序的inode，加锁保证文件读写安全。

### 2. 校验ELF文件头部

```
readi(ip, ... &elf ...)
if(elf.magic != ELF_MAGIC) goto bad;
```

读文件头部，校验魔数，确认是合法 ELF 可执行文件，不是普通文本。

### 3. 创建一份**临时新页表** `pagetable = proc_pagetable(p)`

> ⚠️关键点：**不会直接修改正在运行进程的页表**，先建一份全新用户页表，全部加载成功之后，才替换；中途出错直接销毁这份临时页表，老进程不受破坏。

### 4. 遍历ELF程序段（program header），把程序从磁盘加载到新虚拟地址空间

```
for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    readi(ip,...&ph,...);   //读取一个段头
    if(ph.type != ELF_PROG_LOAD) continue; //只处理需要加载到内存的段
    
    uvmalloc(pagetable, sz, ph.vaddr + ph.memsz); //分配虚拟内存、建立页表映射
    loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz); //把磁盘文件内容拷贝到用户虚拟地址
}
```

- `ph.vaddr`：程序期望运行的虚拟地址
- `filesz`：文件中实际存在的数据；`memsz`：内存需要占用大小（bss零区，文件没有，内存要补0）
- `loadseg`：磁盘inode → 用户虚拟内存，完成代码、数据段加载。

### 5. 分配用户栈（2个页面）

```
sz = PGROUNDUP(sz);
uvmalloc(pagetable, sz, sz + 2*PGSIZE);
uvmclear(pagetable, sz-2*PGSIZE); //清空第一页（守护页，不允许访问）
sp = sz;
stackbase = sp - PGSIZE;
```

> 两个页：
> 
> - 低地址那一页：**guard page（守护页）**，不映射，栈越界访问直接崩溃
> - 高地址那一页：真正的用户栈，栈从高地址向低地址生长 `sp` 从栈顶往下压数据

### 6. 把命令行参数 `argv` 压入**用户栈**（非常核心）

exec运行时是内核态，`argv` 在内核内存，**不能直接给用户程序用**，要复制到用户虚拟栈：

1. 循环每一个参数字符串，`copyout` 把字符串复制到用户栈，向下移动栈指针`sp`，记录每个字符串的地址存入内核数组`ustack[]`
2. 在用户栈再压一层指针数组：`argv[0], argv[1], ... , NULL`
3. 设置trapframe寄存器：
    - `a0`：系统调用返回值，就是`argc`，作为main第一个参数
    - `a1 = sp`：用户栈上argv指针数组的地址，作为main第二个参数
        
        > 用户程序 `main(int argc, char *argv[])` 的参数来源就在这里。
        

### 7. 替换进程上下文，正式切换到新程序运行

```
oldpagetable = p->pagetable;
p->pagetable = pagetable;      //替换进程页表
p->sz = sz;
p->trapframe->epc = elf.entry;// epc = ELF入口地址，即用户程序第一条指令
p->trapframe->sp = sp;        // 用户栈指针
proc_freepagetable(oldpagetable, oldsz); //释放旧的用户地址空间
```

- `epc`：发生trap返回用户态时CPU从epc取指令，所以直接跳转到新程序入口。
- 释放原来进程的用户页表；**内核栈、内核页表、pid保持不变！** exec不改变PID。

```
return argc;
```

返回`argc`，这个返回值会放到`a0`寄存器，就是main函数的`argc`。

### 8. bad 错误分支

只要中间任意一步失败（文件不存在、elf非法、内存分配失败）：销毁刚刚建好的临时页表，文件解锁，返回‑1，**进程继续运行原来的程序，不会崩溃**。

### exec核心要点总结（考试重点）

1. ✅ **不创建新进程，PID不变；只替换用户态地址空间，内核栈、内核页表保留**
2. ✅ 采用“先构造完整新页表，成功才替换”策略，失败不会破坏旧进程。
3. ✅ 从磁盘读取ELF，解析program‑header，把代码段、数据段加载到用户虚拟内存。
4. ✅ 分配用户栈+guard守护页；把argv参数字符串、指针全部拷贝到**用户栈**，设置寄存器为main准备好参数。
5. ✅ 修改trapframe：`epc`设为程序入口，`sp`设用户栈顶；替换页表，释放旧用户内存。
6. ✅ trap返回之后CPU就跑新程序代码。
7. ❗exec不会复制打开文件，默认文件描述符会继承（除非设置close‑on‑exec）。

### 配套调用链路

用户系统调用：`exec(path, argv)` → `sys_exec()` → 调用这个`exec()`函数。

> 搭配：`fork()`复制进程，子进程调用`exec()`加载新程序，这就是shell运行程序的完整流程。

exec()
 │
 ├── 创建新的 pagetable
 │
 ├── 加载 ELF
 │
 ├── 分配用户内存
 │
 ├── 建立用户映射
 │
 ├── 设置 stack
 │
 └── p->pagetable = new pagetable
## xv6 `sbrk()`

> 系统调用：`sbrk(n)`，**扩大/缩小进程堆内存**，修改进程的**brk断点（program break）**，也就是堆的边界。 C库：`void *sbrk(int incr)`；内核里面是 `sys_sbrk()`。

进程虚拟地址空间布局（xv6）：

```
低地址 → 代码段 → 数据段 →【堆 heap】 ↑向上增长 → brk(断点) → 用户栈(高地址向下长)
```

`p->sz` 就是这个进程的**brk值**，代表用户空间当前最大虚拟地址。

### 功能

`sbrk(incr)`：把进程的堆边界 `brk += incr`

1. `incr > 0`：**扩大堆，申请更多内存**
2. `incr < 0`：**缩小堆，释放堆内存**
3. 返回旧的brk地址（扩大前的堆末尾，就是新内存的起始地址）

> 注意：xv6的`sbrk`**不会立刻物理内存**，只是修改虚拟地址边界`p->sz`；真正分配物理页是等到用户访问该虚拟地址，触发**缺页异常(page fault)**的时候，内核才分配物理页、建立PTE映射。

### xv6 sys_sbrk内核源码逻辑（简化）

```
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  struct proc *p = myproc();
  addr = p->sz;        //保存旧brk，作为返回值
  if(n > 0){
    if(uvmalloc(p->pagetable, p->sz, p->sz + n) == 0){
      return -1;
    }
  } else if(n < 0){
    uvmdealloc(p->pagetable, p->sz, p->sz + n);
  }
  p->sz += n;          // 修改进程的size(brk断点)
  return addr;         // 返回原来的brk指针，用户拿到就可以使用这片新内存
}
```

#### 两个关键函数

1. `uvmalloc(pagetable, oldsz, newsz)`

- 增大虚拟地址范围；**只建立页表，不一定分配物理内存**（xv6懒分配lazy allocation）
- 当用户读写这个新虚拟地址，没有物理页，触发`page‑fault`，内核`trap()`里面检测到是合法堆地址，才调用`kalloc()`分配物理页，填PTE。

2. `uvmdealloc(pagetable, oldsz, newsz)`

- n为负数，堆收缩；释放对应虚拟地址的物理页，清除页表PTE。

### 和exec、malloc的关系

1. `malloc()`：用户态库函数，**在sbrk基础上实现**。
    
    ```
    malloc → sbrk(增加堆大小) → 在堆上管理空闲内存块
    ```
    
2. `exec()`：exec会重置p->sz，重新加载程序，堆会被完全重建。

### 考试高频易错点

1. 📌 **sbrk只是修改虚拟地址边界，不一定分配物理内存！懒分配！**
    
    > 调用`sbrk(1000000)`马上返回成功，但没有物理内存；访问这片内存才真正分配物理页。
    
2. 返回值：返回**修改之前的brk地址**，不是新的brk。

```
//举例：brk原来是0x400000
void *p = sbrk(4096);
// p = 0x400000（旧brk），新brk=0x401000
// p指向新获得内存的起始位置
```

3. sbrk(0)：返回当前brk值，不修改大小，可以查询堆边界。
4. 和栈区分：
    - **堆heap：sbrk管理，向上增长**
    - **用户栈：exec一次性分配2页，固定大小，向下增长，不由sbrk控制**

### uvmalloc

- `sbrk`：**系统调用，给用户程序用**，修改`proc->sz`
- `uvmalloc`：**内核内部函数**，exec、sbrk底层都会调用，负责操作页表

### 举个小例子

```
//用户态
char *p = sbrk(4096);
//此时虚拟地址p~p+4096属于本进程堆，但还没有物理页
p[0] = 'A';  
//访问，触发缺页异常，内核分配物理页，建立映射，赋值成功
```

> 拓展：xv6 lazy allocation实验就是修改sbrk/pagefault这一套逻辑。

## 练习1：实现 hello syscall

我们想增加：
```c
hello()
```
执行：
```shell
$ hello
kernel: hello world
```
### 第一步：定义 syscall 编号

系统调用需要编号

```c  kernel/syscall.h

#define SYS_fork 1
#define SYS_exit 2
...
#define SYS_hello 22
```


a7=22,内核才知道调用 hello。

### 第二步：用户声明

```c user/user.h

int hello(void);

```

### 第三步：生成用户入口
```c user/usys.pl
entry("hello");
```
make 后自动生成：user/usys.S  里面：
```asm
hello:
    li a7,SYS_hello
    ecall
    ret
```
### 第四步：内核注册



kernel/syscall.c ```extern uint64 sys_hello(void);```
```c
static uint64 (*syscalls[])()
{

[SYS_hello]
    sys_hello,

};
```
建立映射： 22 -> sys_hello()

### 第五步：实现内核函数

```c kernel/sysproc.c
uint64
sys_hello(void)
{
    printf("kernel: hello world\n");

    return 0;
}
```
第六步：用户程序
```c user/hello.c
#include "kernel/types.h"
#include "user/user.h"

int main()
{
    hello();

    exit(0);
}
```
运行：$ hello
流程：hello.c -> hello() -> usys.S ->  ecall -> usertrap() -> syscall() -> sys_hello() -> printf() -> 返回
这里sys_hello()是被 syscall() 调用的普通 C 函数。它执行完以后，要“返回”到调用它的 syscall().
而 p 是你 syscall() 函数里的局部变量，不是 sys_hello() 的局部变量，所以在 sys_hello() 中 GDB 不认识 p。

## trace作业
让后面的程序运行时，把指定的 syscall 打印出来。
例如：
```shell
$ trace 32 grep hello README
```
运行 grep 的过程中，如果发生：
```shell
read
write
open
close
```
之类的系统调用，就打印：
```shell
3: syscall read
3: syscall write
...
```
比如 执行 ls ，要经历 open() read() write() close() 等,而每个函数（open(),read()）,都要经过syscall(), 进入各自函数（如：sys_read()，sys_write()，sys_open()）
syscall的设计是：             
用户程序 ->  ecall ->  usertrap ->  syscall() ->  sys_open
用户程序 ->  ecall ->  usertrap ->  syscall() ->  sys_read
用户程序 ->  ecall ->  usertrap ->  syscall() ->  sys_write
(每一次 syscall 都是一次完整的“用户态 → 内核态 → 用户态”往返。)

即所有的所有syscall都要经过syscall(),所有系统调用进入内核后的统一分发入口。
num = p->trapframe->a7;
```c proc.c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
这里：    ```num = p->trapframe->a7;```
假设：```a7 = 22```  ; 那么：```num = 22```,  然后：```syscalls[22]() ``` 找到：```sys_hello```
现在trace想做的是：
```  num = p->trapframe->a7``` -> 这个 syscall 是不是我要追踪的？ -> 如果是  -> 打印 syscall 名字  -> 真正执行 syscall

```mask =(1 << SYS_fork) | (1 << SYS_read)``` 其中：```SYS_fork  = 1  SYS_read  = 5```,则为：00100010 这样一个整数，就可以同时记录：

哪些 syscall 要追踪。

如何判断某个 syscall 是否需要 trace？

```p->tracemask``` 保存当前进程的 mask; 得到：```num = p->trapframe->a7```, 使用 ```p->tracemask & (1 << num) ```判断即可.
eg: ```tracemask = 00101000  num = 3```, 则有： ```1 << 3 = 00001000```,做:```00101000 & 00001000 = 00001000```结果非 0,syscall 3 被要求 trace
struct proc 中要新增tracemask,这样，每个进程都有：自己的 trace 配置

trace 设置后，exec 出来的程序还能继承，exec 不会创建一个全新的进程，是同一个proc

fork产生两个进程，所以有 struct proc A 和 struct proc B;
exec 之后还是当前进程，只是：用户程序，地址空间，代码，数据被替换。trace 的 mask 可以继续存在。
要求：trace 设置应该被子进程继承 所以fork()创建子进程时，需要把：parent.tracemask  复制给： child.tracemask

trace(32); 的 32 是通过寄存器传给内核的。进入内核以后：p->trapframe->a0 = 32；p->trapframe->a7 = 23。然后：syscall()，然后syscalls[num]()（num为23的话，就调用函数sys_trace）在函数sys_trace中通过：argint(0, &mask);把把 a0 中的 32 ()取出来放入mask，把mask传给结构体p中的tracemask