---
title: 'Xv6 Lab4 Trap' # <--- 修改这一行
date: "2026-08-22T11:42:25+08:00"
draft: false
tags: ["", ""]
location: ""
---

- ra 返回地址（return address）

- jal 跳转到某个函数 + 同时把“回来以后应该执行哪里”保存到 ra。

	jal ra, f   保存“返回地址” ，跳到 f 。是 `f` 执行完以后，main 应该继续执行的那条指令的地址。
	
```
	0x1000: addi a0, zero, 10
	0x1004: jal  ra, f
	0x1008: addi a0, zero, 20
	0x100c: ...
```
执行：0x1004: jal ra, f
那么：ra = 0x1008，然后：pc = f 的地址
即：
```
	执行前：
	pc = 0x1004
	ra = xxx
	执行 jal：
	
	ra = 0x1008
	pc = f
```

jalr (jump and link register)
```
jalr ra, 0(t0)
```

ra = 返回地址;    pc = t0 + 0
也就是：跳到 t0 保存的地址

直接知道目标地址 -> jal ；目标地址放在寄存器里 ->  jalr

sp addi sp, sp, -32 给当前函数申请 32 bytes 的 stack space。函数调用过程中需要保存很多东西：局部变量、返回地址、保存的寄存器、这些东西通常放在：stack栈，由：sp指向。在 RISC-V 中，stack 通常向低地址增长。函数结束：addi sp, sp, 32释放。

stack frame 每个函数都会拥有自己的stack frame

 /* (fp - 8)  找到返回地址
/ *(fp - 16)  找到上一个 frame


[6.1 Trap机制 | MIT6.S081](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec06-isolation-and-system-call-entry-exit-robert/6.1-trap)
### Trap 的核心机制与架构

xv6 中的 Trap 指的是 CPU 在执行过程中暂停当前程序，将控制权移交给操作系统的机制。Trap 涵盖了三种主要类型：**系统调用（System Calls）**、**异常（Exceptions）**和**中断（Interrupts）**。

用户空间和内核空间的切换通常被称为trap

Trap 的核心目标是从低权级的用户态（User Mode, $U$ mode）安全地切换到高权级的内核态（Kernel Mode, $S$ mode），并在处理完毕后原样恢复用户程序的上下文。
#### RISC-V 关键寄存器
- 在硬件中还有一个寄存器叫做程序计数器（Program Counter Register）。

- 表明当前mode的标志位，这个标志位表明了当前是supervisor mode还是user mode。当我们在运行Shell的时候，自然是在user mode。
	在supervisor mode时，你可以：读写SATP寄存器，也就是page table的指针；STVEC，也就是处理trap的内核指令地址；SEPC，保存当发生trap时的程序计数器；SSCRATCH等等。在supervisor mode你可以读写这些寄存器，而用户代码不能做这样的操作。
	可以使用PTE_U标志位为0的PTE。当PTE_U标志位为1的时候，表明用户代码可以使用这个页表；如果这个标志位为0，则只有supervisor mode可以使用这个页表
	supervisor mode中的代码并不能读写任意物理地址。在supervisor mode中，就像普通的用户代码一样，也需要通过page table来访问内存
- 还有一堆控制CPU工作方式的寄存器，比如SATP（Supervisor Address Translation and Protection）寄存器，它包含了指向page table的物理内存地址（详见4.3）。

xv6 运行在 64 位 RISC-V 架构上，$S$ mode 下依靠以下关键寄存器控制 Trap 流程：

| **寄存器**                                                     | **作用**                                                |
| ----------------------------------------------------------- | ----------------------------------------------------- |
| **`stvec`**(`Supervisor Trap Vector Base Address Register`) | 存储 Trap 处理程序的入口地址（Kernel 或 User trap 向量）。             |
| **`sepc`** (`Supervisor Exception Program Counter`)         | 当 Trap 发生时，硬件自动保存发生 Trap 时的程序计数器（PC）值。                |
| **`sstatus`**                                               | 状态寄存器，控制中断使能（SPP 位标识 Trap 发生前是 User 还是 Supervisor 态）。 |
| **`scause`**                                                | 记录引发 Trap 的原因（如系统调用、页面错误、定时器中断）。                      |
| **`stval`**                                                 | 记录与 Trap 相关的附加信息（如引发缺页异常的具体虚拟地址）。                     |
| **`sscratch`**                                              | 专门用来临时保存内核数据结构的指针（在进入 Trap handler 前暂存上下文）。           |
#### 页表隔离与 Trampoline

用户进程的页表和内核页表是完全隔离的。为确保发生 Trap 时能正确执行内核代码，xv6 将同一个名为 **`trampoline`** 的物理页映射到了：

1. 每个用户页表的最高虚拟地址处 (`MAXVA - PGSIZE`)。

2. 内核页表的最高虚拟地址处。

因为映射地址和权限相同，在切换 `satp`（页表基址寄存器）时，执行流不会打断。


操作系统的一些high-level的目标能帮我们过滤一些实现选项
- 安全和隔离。不想让用户代码介入到这里的user/kernel切换。有可能会破坏安全性。trap中涉及到的硬件和内核机制不能依赖任何来自用户空间东西（如32个用户寄存器，可能是恶意数据）XV6的trap机制不会查看这些寄存器，而只是将它们保存起来
- 想要让trap机制对用户代码是透明的。执行trap，然后在内核中执行代码，同时用户代码并不用察觉到任何

## Traps流程
### 用户态准备与触发(`user/usys.S`)
在用户空间，系统调用通过汇编存入系统调用号，并执行 `ecall` 指令：
```
# 以 fork 为例
.globl fork
fork:
 li a7, SYS_fork    # 将系统调用号放入 a7 寄存器
 ecall              # 触发 Trap，CPU 进入 S mode
 ret
```
### 执行 `ecall` 前一刻的状态 (User Mode / U Mode)（fork为例）
**`PC` (程序计数器)**: `0x0000000000000e10` （指向用户代码段中的 `ecall` 指令）
**`mode` (CPU 当前运行模式)**: **U Mode** (用户态)
**`a7` (系统调用号)**: `1` (`SYS_fork`)
**`sp` (栈指针)**: `0x000000000003f000` (指向用户栈)
**`stvec` (内核 Trap 入口地址)**: `0x3fffffe000` (内核提前写入的 `trampoline` 页中的 `uservec` 地址)
**`sscratch`**: `0x000000008001d000` (内核提前存好的该进程的 `trapframe` 物理/虚拟映射地址)
**`sstatus`**: `0x0000000000000020` (`SPP` 位为 0，表示前一个状态是 U Mode)
### write通过执行ECALL指令来执行系统调用
这个阶段完全由 **CPU 硬件电路自动完成**。ECALL指令会切换到具有supervisor mode的内核中。
### 执行 `ecall` 的那一瞬间（硬件自动化处理）

当 CPU 译码并执行 `ecall` 时，内部的逻辑电路在**一个时钟周期内**自动完成以下几件事：

1. **保存现场 PC 到 `sepc`**：

	- 硬件把 `ecall` 本身的地址写入 `sepc`。

	- **`sepc` 变为**: `0x0000000000000e10`

2. **记录控制权切换前的工作模式到 `sstatus`**：

	- 将 `sstatus` 中的 `SPP` (Supervisor Previous Privilege) 位置为 `0`（代表发生 Trap 之前是 U mode）。

	- 将 `sstatus` 中的 `SPIE` 位置为当前的 `SIE` 值（保存以前的中断使能状态），然后把 `SIE` 清零（自动**关闭全局中断**，防止 Trap 刚发生就被打断）。

3. **设置 Trap 原因到 `scause`**：

	- 因为是 U Mode 下的 `ecall`，系统填入相应的 Exception Code（值为 `8`）。

	- **`scause` 变为**: `8` (`Environment call from U-mode`)

4. **强制切换 CPU 权限模式**：

	- **`mode` 变为**: **S Mode** (Supervisor Mode/内核态)。

5. **跳转到 Trap 处理入口**：

	- CPU 将 `stvec` 寄存器里的值强制赋给 `PC`。

	- **`PC` 变为**: `0x3fffffe000` (即 `uservec` 的第一条指令地址)。

到达 `uservec` 第一条指令时的关键状态对比
在 `uservec` 的第一条汇编指令 `csrrw a0, sscratch, a0` 刚准备执行、但**尚未执行**的那一瞬间，系统状态如下：
|**寄存器 / 状态**|**ecall 执行前 (U Mode)**|**刚到达 uservec 时 (S Mode)**|**变化原因**|
|---|---|---|---|
|**CPU Mode**|**U Mode**|**S Mode**|硬件自动提升权级|
|**`PC`**|`0x0000000000000e10`|**`0x3fffffe000`**|硬件强行装载了 `stvec` 中的地址|
|**`sepc`**|(旧值)|**`0x0000000000000e10`**|硬件自动备份当前 `ecall` 的地址|
|**`scause`**|(旧值)|**`8`**|硬件记录 Trap 原因（来自用户态的系统调用）|
|**`satp`**|用户页表地址|**用户页表地址**|**硬件没有切换页表！** 依然运行在用户页表下|
|**通用寄存器 (`a0`-`a7`, `sp` 等)**|用户代码设置的值|**完全没有改变！**|硬件**不负责保存**通用寄存器|

### 内核执行汇编函数uservec（内核代码trampoline.s文件的一部分）
`uservec` 的核心使命是：**在不破坏用户寄存器的前提下保存它们，并切入内核上下文**。

  - 利用 `csrrw` 将 `a0` 与 `sscratch` 交换。此时 `a0` 指向当前进程的 `p->trapframe` 结构体。
	- csrrw a0, sscratch, a0 这条指令会**原子性地交换** `a0` 和 `sscratch` 的值。
	- 交换前：`a0` 里面是用户参数，`sscratch` 里面是 `trapframe` 的地址。
	- 交换后：`a0` 变成了 `trapframe` 的地址（内核终于有了可用的基址指针），而 `sscratch` 保存了原本的用户 `a0` 值。
	- 随后，`uservec` 才能利用 `a0` 作为基准地址，安全地将所有用户通用寄存器（`ra`, `sp`, `a1`~`a7` 等）一次性备份到 `trapframe` 中。
     
- 将用户寄存器（`ra`, `sp`, `gp`, `tp`, `t0-t6`, `a1-a7`, `s0-s11`）全部保存到 `trapframe` 内存区域。
    
- 从 `trapframe` 读取内核栈指针 `sp`、内核页表 `satp` 地址以及 `usertrap` 的函数指针。
    
- 切换 `satp` 到内核页表，刷新 TLB（`sfence.vma`）。

- 跳转到 `usertrap()`。
### 跳转到了由C语言实现的函数usertrap中（trap.c中）

`usertrap()` 根据 `scause` 确定 Trap 类型：  

- 更改 `stvec` 为 `kernelvec`（防止内核执行期间发生 Trap 时错走 `uservec`）。 
    
- 保存 `sepc` 到 `p->trapframe->epc`（因为内核执行中可能会切换进程/再次中断）。
    
- 如果是因为 `ecall` 触发（`scause == 8`）：
    1. 检查进程是否已被杀死。
        
    2. `p->trapframe->epc += 4`（跳过 `ecall` 指令本身，防止返回后重复执行 `ecall`）。
        
    3. 开启硬件中断 `intr_on()`。
        
    4. 调用 `syscall()`。
### 系统调用派发 (`kernel/syscall.c` -> `syscall`)
这个函数会在一个表单中，根据传入的代表系统调用的数字进行查找，并在内核中执行具体实现了系统调用功能的函数。对于我们来说，这个函数就是sys_write。
sys_write会将要显示数据输出到console上，当它完成了之后，它会返回给syscall函数。
- 从 `p->trapframe->a7` 读取系统调用号。
    
- 检查调用号合法性，根据数组索引调用对应的内核函数（如 `sys_fork`）。
    
- 系统调用的返回值存入 `p->trapframe->a0`。
### 系统调用内核实现 (`kernel/sysproc.c`, `kernel/sysfile.c` 等)

- 内核需要读取用户传入的参数时，利用 helper 函数从 `trapframe->a0` ~ `a5` 读取。
    
- 若参数包含用户空间指针（如 `write(fd, buf, n)` 中的 `buf`），必须使用 `copyin()` 或 `copyout()` 穿透页表拷贝数据，绝对不能直接访问用户指针。
### 调用一个函数叫做usertrapret，它也位于trap.c中
因为我们现在相当于在ECALL之后中断了用户代码的执行，为了用户空间的代码恢复执行。这个函数完成了部分方便在C代码中实现的返回到用户空间的工作。
1. 关闭中断。
	
2. 重置 `stvec` 为 `uservec`。
	
3. 填写 `trapframe` 中下次进入内核所需的信息（`kernel_satp`, `kernel_sp`, `kernel_trap`）。
	
4. 设置 `sstatus`：清除 `SPP`（标明返回 $U$ mode），使能 `SPIE`（恢复中断）。
	
5. 将 `p->trapframe->epc` 写入 `sepc`。

6. 准备参数，调用 `userret`。
	
### trampoline.s文件中的userret函数
还有一些工作只能在汇编语言中完成。这部分工作通过汇编语言实现，并且存在于trampoline.s文件中的userret函数中。
在这个汇编函数中会调用机器指令返回到用户空间，并且恢复ECALL之后的用户程序的执行.
    1. 切换页表 `satp` 为用户页表。
        
    2. 从 `trapframe` 恢复用户通用寄存器（包括含返回值的 `a0`）。
        
    3. 执行 `sret` 指令。
        
### **`sret` 硬件动作**
将 `sepc` 的值复制到 `PC`，恢复 $U$ mode 运行状态，重新开启中断。
![](Pasted%20image%2020260827105719.png)

### 





### 系统调用的完整生命流程

以用户程序调用 `write()` 为例，整个执行链条如下：

  #### 1. 用户态准备与触发 (`user/usys.S`)

在用户空间，系统调用通过汇编存入系统调用号，并执行 `ecall` 指令：

代码段
```
# 以 fork 为例
.globl fork
fork:
 li a7, SYS_fork    # 将系统调用号放入 a7 寄存器
 ecall              # 触发 Trap，CPU 进入 S mode
 ret
```
#### 2. 硬件行为 (CPU 自动完成)
当 `ecall` 被执行时，RISC-V 硬件自动完成以下动作：


- 将中断使能位（SIE）清零，禁用中断。   
    
- 将当前的 `PC` 保存到 `sepc`。  
    
- 将当前模式（$U$ mode）保存到 `sstatus` 的 `SPP` 位。
    
- 将 `PC` 设置为 `stvec` 中的地址（即 `uservec` 的入口）。
    
- 切换 CPU 到 Supervisor mode。
    

#### 3. 用户态 Trap 入口 (`kernel/trampoline.S` -> `uservec`)

`uservec` 的核心使命是：**在不破坏用户寄存器的前提下保存它们，并切入内核上下文**。

  - 利用 `csrrw` 将 `a0` 与 `sscratch` 交换。此时 `a0` 指向当前进程的 `p->trapframe` 结构体。
     
- 将用户寄存器（`ra`, `sp`, `gp`, `tp`, `t0-t6`, `a1-a7`, `s0-s11`）全部保存到 `trapframe` 内存区域。
    
- 从 `trapframe` 读取内核栈指针 `sp`、内核页表 `satp` 地址以及 `usertrap` 的函数指针。
    
- 切换 `satp` 到内核页表，刷新 TLB（`sfence.vma`）。

- 跳转到 `usertrap()`。
    
#### 4. 内核 C 语言处理 (`kernel/trap.c` -> `usertrap`)

`usertrap()` 根据 `scause` 确定 Trap 类型：  

- 更改 `stvec` 为 `kernelvec`（防止内核执行期间发生 Trap 时错走 `uservec`）。 
    
- 保存 `sepc` 到 `p->trapframe->epc`（因为内核执行中可能会切换进程/再次中断）。
    
- 如果是因为 `ecall` 触发（`scause == 8`）：
    1. 检查进程是否已被杀死。
        
    2. `p->trapframe->epc += 4`（跳过 `ecall` 指令本身，防止返回后重复执行 `ecall`）。
        
    3. 开启硬件中断 `intr_on()`。
        
    4. 调用 `syscall()`。
        

#### 5. 系统调用派发 (`kernel/syscall.c` -> `syscall`)

- 从 `p->trapframe->a7` 读取系统调用号。
    
- 检查调用号合法性，根据数组索引调用对应的内核函数（如 `sys_fork`）。
    
- 系统调用的返回值存入 `p->trapframe->a0`。
    

#### 6. 系统调用内核实现 (`kernel/sysproc.c`, `kernel/sysfile.c` 等)

- 内核需要读取用户传入的参数时，利用 helper 函数从 `trapframe->a0` ~ `a5` 读取。
    
- 若参数包含用户空间指针（如 `write(fd, buf, n)` 中的 `buf`），必须使用 `copyin()` 或 `copyout()` 穿透页表拷贝数据，绝对不能直接访问用户指针。
    

#### 7. 返回用户态 (`kernel/trap.c` -> `usertrapret` & `trampoline.S` -> `userret`)

- **`usertrapret`**:
      
    1. 关闭中断。
        
    2. 重置 `stvec` 为 `uservec`。
        
    3. 填写 `trapframe` 中下次进入内核所需的信息（`kernel_satp`, `kernel_sp`, `kernel_trap`）。
        
    4. 设置 `sstatus`：清除 `SPP`（标明返回 $U$ mode），使能 `SPIE`（恢复中断）。
        
    5. 将 `p->trapframe->epc` 写入 `sepc`。

    6. 准备参数，调用 `userret`。
        
- **`userret`**:
 
    1. 切换页表 `satp` 为用户页表。
        
    2. 从 `trapframe` 恢复用户通用寄存器（包括含返回值的 `a0`）。
        
    3. 执行 `sret` 指令。
        
- **`sret` 硬件动作**：将 `sepc` 的值复制到 `PC`，恢复 $U$ mode 运行状态，重新开启中断。
    

### 内核态 Trap (Kernel Traps)

当 CPU 已经在 $S$ mode 执行内核代码时，依然可能发生 Trap（如设备中断或内核异常）。

- 内核 Trap 的入口由 `kernelvec`（`kernel/kernelvec.S`）处理。
    
- 内核 Trap 直接使用当前进程的**内核栈**保存上下文，不需要切换页表，也不使用 `trampoline`。
    
- 保存上下文后调用 `kerneltrap()`（`kernel/trap.c`）。

- 处理完成后直接通过 `sret` 恢复现场并继续执行内核代码。
    

### 完成 Lab: Traps 所需的知识储备

在进行 Lab: Traps 实验前，需要清楚以下关键点：

1. **RISC-V Calling Convention（调用栈与寄存器约定）**

    - **函数调用参数**：放置在 `a0` - `a7` 寄存器中；**返回值**在 `a0`（及 `a1`）。
        
    - **栈帧指针（Frame Pointer）**：在 RISC-V 中，`s0` (即 `fp`) 保存栈帧基址，`ra` 保存返回地址。RISC-V 的栈向下增长（从高地址向低地址）。
        
    - **栈帧回溯原理**：前一个栈帧的指针保存在当前 `fp - 8` 的位置，返回地址保存在当前 `fp - 16` 的位置。
        
2. **寄存器与 Trapframe 的映射关系**
      
    - 系统调用参数的获取方式是访问 `p->trapframe->a0`, `p->trapframe->a1` 等。
        
    - 修改返回值即修改 `p->trapframe->a0`。
        
3. **时钟中断与抢占**

    - RISC-V 的 Timer Interrupt 会由 Machine mode 转化为 $S$ mode 的软中断。
        
    - `usertrap` / `kerneltrap` 捕获时钟中断后，会调用 `yield()` 让出 CPU，从而实现多任务抢占。


|**寄存器 / 状态**|**ecall 执行前 (U Mode)**|**刚到达 uservec 时 (S Mode)**|**变化原因**|
|---|---|---|---|
|**CPU Mode**|**U Mode**|**S Mode**|硬件自动提升权级|
|**`PC`**|`0x0000000000000e10`|**`0x3fffffe000`**|硬件强行装载了 `stvec` 中的地址|
|**`sepc`**|(旧值)|**`0x0000000000000e10`**|硬件自动备份当前 `ecall` 的地址|
|**`scause`**|(旧值)|**`8`**|硬件记录 Trap 原因（来自用户态的系统调用）|
|**`satp`**|用户页表地址|**用户页表地址**|**硬件没有切换页表！** 依然运行在用户页表下|
|**通用寄存器 (`a0`-`a7`, `sp` 等)**|用户代码设置的值|**完全没有改变！**|硬件**不负责保存**通用寄存器|
这道 Backtrace 的本质是：手动实现一个极简版的 GDB `bt`，沿 RISC-V 的 frame pointer 链，从当前函数一路找到调用者。

题目要求的调用链是：

```
用户程序 bttest
    ↓ sleep() + ecall
usertrap()
    ↓
syscall()
    ↓
sys_sleep()
    ↓
backtrace()
```

最终打印的三个地址分别落在：

```
sys_sleep
syscall
usertrap
```

MIT 2020 实验题面明确要求修改 `kernel/riscv.h`、`kernel/defs.h`、`kernel/printf.c`、`kernel/sysproc.c`，并在最后把 `backtrace()` 接入 `panic()`。[MIT 6.S081 Backtrace 题面](https://pdos.csail.mit.edu/6.S081/2020/labs/traps.html)

> 当前工作目录只有一个尚无提交的空 `.git`，所以以下步骤需要在你实际的 `xv6-labs-2020` 仓库中执行。建议使用 Linux/WSL 环境。

# 一、先理解栈帧

## 1. 函数调用时保存什么

RISC-V 中：

- `sp`：stack pointer，当前栈顶
- `ra`：return address，函数返回后要执行的地址
- `s0/fp`：frame pointer，当前栈帧的固定参考点

RISC-V 的栈向低地址增长：

```
高地址
         调用者的栈帧
              ↑
              │ 沿 saved s0 回溯时，地址逐渐变大
              │
s0 ────────→  当前函数进入前的 s0
s0 - 8      保存的 ra
s0 - 16     保存的调用者 s0
              │
              ↓
         当前函数的局部变量
sp ────────→ 当前栈顶
低地址
```

RV64 中一个指针是 8 字节，因此 RISC-V ABI 规定：

```
返回地址       = *(uint64 *)(fp - 8)
调用者的 fp    = *(uint64 *)(fp - 16)
```

这是正式 RISC-V ABI 定义的 frame record 布局：`s0` 作为 frame pointer，前一个 frame pointer 和 return address 存在它之前。[RISC-V psABI](https://riscv-non-isa.github.io/riscv-elf-psabi-doc/)

典型函数序言类似：

```
addi sp, sp, -32    # 分配 32 字节栈帧
sd   ra, 24(sp)     # 保存 ra
sd   s0, 16(sp)     # 保存旧 s0
addi s0, sp, 32     # 新 fp = 函数进入前的 sp
```

因为：

```
s0 = sp + 32
24(sp) = s0 - 8
16(sp) = s0 - 16
```

所以回溯代码不需要知道每个栈帧有多大，只需要知道两个固定偏移。

## 2. 为什么不能直接反复读 `ra`

当 `backtrace()` 正在执行时，寄存器 `ra` 只表示：

```
backtrace() 应该返回到 sys_sleep() 的哪里
```

它不知道 `sys_sleep()` 的调用者是谁。

但 `sys_sleep()` 进入时把自己的 `ra` 保存到了它的栈帧。只要沿着 saved fp 找到 `sys_sleep()` 的栈帧，就能读取那个 saved `ra`，继续找到 `syscall()`。

因此 frame pointer 把栈帧组成了一条链表：

```
当前 fp
  │
  ├── fp - 8  → 当前函数的返回地址
  └── fp - 16 → 调用者的 fp
                    │
                    ├── fp - 8  → 调用者的返回地址
                    └── fp - 16 → 更上层调用者的 fp
```

## 3. 为什么这种方法依赖编译选项

编译器有时会把 `s0` 当普通寄存器使用，不维护 frame pointer 链。xv6 的 Makefile 使用：

```
-fno-omit-frame-pointer
```

它要求编译器保留 frame pointer，否则这个实验无法可靠工作。

此外，尾调用优化、栈损坏、手写汇编等也可能让简单的 frame-pointer unwinder 无法继续。这个实验能工作，是因为 xv6 的编译环境和内核调用路径足够受控。

# 二、实现代码

总共修改四处；最后再接入 `panic()`。

## 第一步：实现读取 `s0` 的函数

在 `kernel/riscv.h` 中，放到仅供 C 使用的部分，也就是匹配的 `#ifndef __ASSEMBLER__` 内：

```
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x));
  return x;
}
```

逐部分解释：

```
asm volatile(...)
```

表示嵌入一条汇编指令，并告诉编译器不要认为它无用而删除。

```
mv %0, s0
```

把 `s0` 的值复制到输出操作数 `%0`。`mv` 是伪指令，本质类似：

```
addi 目标寄存器, s0, 0
```

```
"=r" (x)
```

含义是：

- `=`：这是一个只写输出
- `r`：请编译器分配一个通用寄存器
- 最终把这个输出视为 C 变量 `x`

所以整个函数等价于：

```
读取当前 s0 → 放进 x → 返回 x
```

## 第二步：声明函数

在 `kernel/defs.h` 的 `printf.c` 函数声明附近加入：

```
void backtrace(void);
```

原因是 `backtrace()` 定义在 `printf.c`，但要从 `sysproc.c` 调用。C 编译器需要提前知道它的函数签名。

不要写成：

```
void backtrace();
```

旧式声明在 C 中表示参数情况未明确。这里写 `(void)` 更准确。

## 第三步：实现 `backtrace()`

在 `kernel/printf.c` 中加入：

```
void
backtrace(void)
{
  printf("backtrace:\n");

  uint64 fp = r_fp();
  uint64 stack_bottom = PGROUNDDOWN(fp);
  uint64 stack_top = PGROUNDUP(fp);

  while (fp >= stack_bottom + 16 && fp < stack_top) {
    uint64 ra = *(uint64 *)(fp - 8);
    printf("%p\n", ra);
    fp = *(uint64 *)(fp - 16);
  }
}
```

也可以把栈顶写得更稳健：

```
uint64 stack_top = stack_bottom + PGSIZE;
```

因为当前 `fp` 一般不是页对齐的，两种写法在本实验中等价。

### `fp - 8`

```
uint64 ra = *(uint64 *)(fp - 8);
```

执行过程：

1. `fp - 8`：得到保存 `ra` 的内存地址
2. `(uint64 *)`：告诉 C 这是一个指向 64 位整数的指针
3. `*`：读取该地址中的 64 位返回地址

注意下面两种写法不相同：

```
*(uint64 *)(fp - 8)   // 正确：地址减去 8 字节
*((uint64 *)fp - 8)   // 错误：指针减去 8 个 uint64，即 64 字节
```

### `fp - 16`

```
fp = *(uint64 *)(fp - 16);
```

读取保存的调用者 frame pointer，于是循环进入上一层栈帧。

### 为什么以一个内核栈页为边界

xv6 给每个进程分配一页 4 KiB 的内核栈，而且栈页按页对齐：

```
stack_top
+----------------------+  页边界
| usertrap 栈帧        |
| syscall 栈帧         |
| sys_sleep 栈帧       |
| backtrace 栈帧       |
|                      |
+----------------------+  页边界
stack_bottom
```

因此：

```
PGROUNDDOWN(fp)
```

得到当前栈页底部；

```
PGROUNDUP(fp)
```

得到当前栈页顶部。

调用者的栈帧位于更高地址，所以遍历过程中 `fp` 通常不断变大。一旦到达页顶，就不能继续解引用，否则可能访问 guard page 或不属于当前内核栈的内存。

边界条件：

```
fp >= stack_bottom + 16
```

确保 `fp - 16` 没有越过页底。

```
fp < stack_top
```

确保没有跨出内核栈页。

这里使用 `< stack_top` 而不是 `<= stack_top` 也解释了为什么输出通常正好三项：

```
backtrace 的 ra → sys_sleep
sys_sleep 的 ra → syscall
syscall 的 ra   → usertrap
```

继续向上时，`usertrap` 的 frame pointer 通常已经到栈页顶部，所以循环终止，不进入 trampoline 的手写汇编环境。

### 一个容易写错的循环

不要写：

```
while (PGROUNDDOWN(fp) < PGROUNDUP(fp)) {
  ...
}
```

它实际上主要在测试 `fp` 是否恰好页对齐。如果每次都根据新的 `fp` 重新计算所属页面，即使损坏的 `fp` 跳到另一个页面，循环也可能继续。

正确思想是：

> 根据初始 `fp` 只计算一次当前内核栈的上下界，之后所有 frame pointer 都必须留在这个原始范围内。

## 第四步：从 `sys_sleep()` 调用

在 2020 版本的 `kernel/sysproc.c` 中找到：

```
uint64
sys_sleep(void)
{
```

在函数开头加入：

```
uint64
sys_sleep(void)
{
  backtrace();

  int n;
  uint ticks0;
  ...
}
```

为什么选这里？

用户态的 `bttest` 会调用 `sleep()`：

```
bttest
  → 用户 sleep 系统调用桩
  → ecall
  → usertrap
  → syscall
  → sys_sleep
  → backtrace
```

这样能稳定构造出题目期待的调用链。

# 三、编译和功能验证

先确认处于实验分支：

```
git fetch
git checkout traps
make clean
```

编译并运行：

```
make qemu
```

进入 xv6 shell 后：

```
$ bttest
```

预期类似：

```
backtrace:
0x0000000080002cda
0x0000000080002bb6
0x0000000080002898
```

地址不必与题目完全相同。编译器版本、源码改动和链接布局都会改变地址。

退出 QEMU：

```
Ctrl-A
X
```

注意依次按 `Ctrl-A`，松开后再按 `x`。

## 将地址转换成源码位置

可以交互式输入：

```
riscv64-unknown-elf-addr2line -f -p -e kernel/kernel
```

然后逐行粘贴：

```
0x0000000080002cda
0x0000000080002bb6
0x0000000080002898
```

最后按 `Ctrl-D`。

也可以一次传入：

```
riscv64-unknown-elf-addr2line \
  -f -p -e kernel/kernel \
  0x0000000080002cda \
  0x0000000080002bb6 \
  0x0000000080002898
```

结果应分别落在：

```
kernel/sysproc.c
kernel/syscall.c
kernel/trap.c
```

返回地址指向的是 `call` 之后将继续执行的位置，不是被调用函数的入口。因此源码行有时显示为函数调用的下一行，这是正常现象。

# 四、用 GDB 手工验证栈帧链

建议使用一个 CPU，避免多核让断点输出难以跟踪。MIT 的 GDB 指南也推荐 `qemu-gdb` 配合 RISC-V GDB。[MIT GDB Guidance](https://pdos.csail.mit.edu/6.S081/2022/labs/gdb.html)

## 1. 终端 A：启动 QEMU GDB Server

在 xv6 仓库根目录：

```
make CPUS=1 qemu-gdb
```

QEMU 会暂停，等待 GDB 连接。留意输出的端口，通常类似：

```
tcp::26000
```

不要关闭这个终端。

## 2. 终端 B：启动对应架构的 GDB

仍然在 xv6 仓库根目录：

```
riscv64-unknown-elf-gdb kernel/kernel
```

如果没有这个命令，可以尝试：

```
gdb-multiarch kernel/kernel
```

若仓库的 `.gdbinit` 正常加载，它一般会自动连接 QEMU。

如果没有自动连接，手动执行：

```
(gdb) set architecture riscv:rv64
(gdb) target remote localhost:26000
```

端口以终端 A 输出为准。

如果出现 `.gdbinit auto-loading has been declined`，按照 GDB 警告给出的路径执行类似：

```
add-auto-load-safe-path /你的/xv6-labs-2020/.gdbinit
```

最好把它写进用户级 `~/.gdbinit`，然后重启 GDB。

## 3. 设置断点

在 GDB 中：

```
(gdb) set pagination off
(gdb) set disassemble-next-line on
(gdb) break sys_sleep
(gdb) break backtrace
(gdb) continue
```

此时 QEMU 开始运行。

在终端 A 的 xv6 shell 中输入：

```
$ bttest
```

GDB 首先应该停在 `sys_sleep()`。

## 4. 观察完整调用链

在 `sys_sleep()` 断点处：

```
(gdb) backtrace
```

或者简写：

```
(gdb) bt
```

预期类似：

```
#0  sys_sleep ()
#1  syscall ()
#2  usertrap ()
#3  ...
```

这就是 GDB 根据调试信息完成的回溯。接下来我们用自己的方式手动重复它。

查看关键寄存器：

```
(gdb) info registers sp s0 ra
```

解释：

- `sp`：当前实际栈顶
- `s0`：`sys_sleep` 当前栈帧的 frame pointer
- `ra`：`sys_sleep` 返回到 `syscall` 的地址

查看函数汇编：

```
(gdb) disassemble /r sys_sleep
```

寻找类似：

```
addi sp,sp,-32
sd   ra,24(sp)
sd   s0,16(sp)
addi s0,sp,32
```

亲自验证：

```
s0 - 8  = sp + 24 = 保存 ra 的位置
s0 - 16 = sp + 16 = 保存旧 s0 的位置
```

## 5. 进入 `backtrace()`

继续执行：

```
(gdb) continue
```

现在会命中 `backtrace` 断点。

先让 GDB 自己回溯：

```
(gdb) bt
```

预期：

```
#0 backtrace
#1 sys_sleep
#2 syscall
#3 usertrap
```

查看当前寄存器：

```
(gdb) info registers sp s0 ra
```

将当前 frame pointer 保存为 GDB 临时变量：

```
(gdb) set $fp0 = (unsigned long long)$s0
(gdb) p/x $fp0
```

## 6. 查看第一个 frame record

一次查看 `fp-16` 和 `fp-8`：

```
(gdb) x/2gx $fp0-16
```

内存布局是：

```
第一个 8 字节：saved fp，地址 fp0-16
第二个 8 字节：saved ra，地址 fp0-8
```

分别读取：

```
(gdb) set $fp1 = *(unsigned long long *)($fp0 - 16)
(gdb) set $ret0 = *(unsigned long long *)($fp0 - 8)

(gdb) p/x $fp1
(gdb) p/x $ret0
```

识别第一个返回地址：

```
(gdb) info symbol $ret0
(gdb) info line *$ret0
(gdb) x/i $ret0
```

结果应显示它位于 `sys_sleep()` 中，因为这是 `backtrace()` 返回后继续执行的位置。

## 7. 手动进入调用者栈帧

现在 $fp1 是 `sys_sleep()` 的 frame pointer：

```
(gdb) x/2gx $fp1-16

(gdb) set $fp2 = *(unsigned long long *)($fp1 - 16)
(gdb) set $ret1 = *(unsigned long long *)($fp1 - 8)

(gdb) p/x $ret1
(gdb) info symbol $ret1
(gdb) info line *$ret1
```

$ret1 应位于：

```
syscall()
```

再向上一层：

```
(gdb) x/2gx $fp2-16

(gdb) set $fp3 = *(unsigned long long *)($fp2 - 16)
(gdb) set $ret2 = *(unsigned long long *)($fp2 - 8)

(gdb) p/x $ret2
(gdb) info symbol $ret2
(gdb) info line *$ret2
```

$ret2 应位于：

```
usertrap()
```

至此，你已经完全手工完成了 `backtrace()` 循环做的事情：

```
$fp0 --saved fp--> $fp1 --saved fp--> $fp2 --saved fp--> $fp3

$fp0-8 → sys_sleep 内的返回点
$fp1-8 → syscall 内的返回点
$fp2-8 → usertrap 内的返回点
```

## 8. 验证栈页边界

计算当前栈页底部和顶部：

```
(gdb) set $bottom = $fp0 & ~0xfff
(gdb) set $top = $bottom + 0x1000

(gdb) p/x $bottom
(gdb) p/x $top
(gdb) p/x $fp0
(gdb) p/x $fp1
(gdb) p/x $fp2
(gdb) p/x $fp3
```

你应该观察到：

```
bottom < fp0 < fp1 < fp2 < top
```

而 $fp3 通常已经到达栈页顶部附近或等于顶部，因此循环：

```
while (fp < stack_top)
```

在这里停止。

这就是循环终止条件的实际依据，不是一个随意选择的限制。

## 9. 单步观察 C 循环

重新运行一次：

```
(gdb) continue
```

让第一次 `bttest` 完成，然后可以重新启动 QEMU，或者再次运行 `bttest`。

命中 `backtrace()` 后：

```
(gdb) list backtrace
```

当执行到初始化 `fp` 后，设置显示：

```
(gdb) display/x fp
(gdb) display/x *(unsigned long long *)(fp - 8)
(gdb) display/x *(unsigned long long *)(fp - 16)
```

然后逐行执行：

```
(gdb) next
```

不要用 `step` 经过 `printf()`，否则会进入打印实现的大量内部代码；这里使用 `next`。

你会看到每次循环：

```
fp 变大
fp-8 的返回地址改变
fp-16 给出下一次 fp
```

由于 xv6 使用优化编译，GDB 偶尔可能显示变量被优化或源码行跳跃。这时以 $s0 和直接读取内存的手工方法为准。

# 五、接入 `panic()`

功能验证正确后，在 `kernel/printf.c` 的 `panic()` 中，放在 panic 信息打印完成之后、最终死循环之前：

```
void
panic(char *s)
{
  // 原有代码……

  printf("panic: ");
  printf(s);
  printf("\n");

  backtrace();

  // 原有的 panicked 设置和死循环……
}
```

不要重复改写整个 `panic()`；只在原实现合适位置插入：

```
backtrace();
```

这样以后内核 panic 时，不仅显示错误信息，还会显示导致 panic 的内核调用路径。

# 六、最后验证

运行：

```
make clean
make
make grade
```

重点检查：

- `bttest` 能打印 `backtrace:`
- 打印大约三个内核地址
- `addr2line` 分别解析到 `sysproc.c`、`syscall.c`、`trap.c`
- 不发生 page fault
- 不出现无限循环
- `make grade` 通过

最容易犯的错误是：

- 使用 `sp` 代替 `s0`
- 打印当前寄存器 `ra`，没有读取栈中的 saved `ra`
- 写成 `((uint64 *)fp - 8)`，错误地减了 64 字节
- 每次循环重新计算新 `fp` 所在页面
- 把 `fp-8` 与 `fp-16` 写反
- 使用 `fp <= stack_top`，意外继续进入 trampoline
- 忘记在 `defs.h` 声明
- 把 `r_fp()` 放到 `riscv.h` 的汇编可见区域
- 用普通 x86 GDB 调试 RISC-V 内核
- 在错误目录启动 GDB，导致找不到 `kernel/kernel` 或调试符号

最终应该形成这个心智模型：

```
每个函数进入时：
    保存调用者 fp
    保存返回地址
    建立自己的 fp

backtrace：
    从 s0 开始
    读取 fp-8，打印返回地址
    读取 fp-16，进入调用者栈帧
    在当前 4 KiB 内核栈页顶部停止
```

这不是通过搜索函数名字实现回溯，而是在读取编译器按照 ABI 留在栈中的“调用链链表”。