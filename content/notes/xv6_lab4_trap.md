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