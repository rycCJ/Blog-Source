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