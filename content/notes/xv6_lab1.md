---
title: 'Xv6 Lab1' # <--- 修改这一行
date: "2026-07-29T10:38:38+08:00"
draft: false
tags: ["xv6", ""]
location: "广州"
---
```c user/ulib.c
 int
atoi(const char *s)
{
  int n;

  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}
 ```
 功能：把数字字符串转成整数
 *s++：后置自增 先拿当前指针内容，指针再加一。
 *(++s)：指针先 + 1，再取新位置字符。

 ```c kernel/sysproc.c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;
  // 刚开始：进程：RUNNING  锁：UNLOCKED
  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);  //从用户态获取系统调用参数。执行此操作之后：tickslock变为LOCKED（已上锁），除非release（释放锁）,否则其他进程就不能直接拿到锁
  // 此时：进程：RUNNING   锁：LOCKED
  // 锁的目的就是:同一时刻只能让一个执行流进入临界区。
  ticks0 = ticks;
  while(ticks - ticks0 < n){  
    if(myproc()->killed){         // myproc()获取当前进程
      release(&tickslock);  //发现进程被杀死，就要立即释放锁
      return -1;
    }
    sleep(&ticks, &tickslock);  //执行xv6 内核内部的进程睡眠函数。叫做：阻塞等待 / 睡眠
    // 作用：让当前进程进入 Sleeping 状态（不再占用 CPU），并等待 ticks 这个等待通道被唤醒。
    // 睡眠之前需要正确释放、唤醒后需要重新获取的锁。如果睡眠的时候不释放锁，其他进程想要锁的话 就得一直等待。
    // 此时：进程：SLEEPING   锁：UNLOCKED
    // 不使用while检查的原因（  while (ticks - ticks0 < n) {    // 什么都不做}  ）：会一直检查，占用CPU。使用while也叫忙等待

  }
  release(&tickslock);
  return 0;
}
 ```
 程序流程：用户程序调用 sleep(5) -->进入 sys_sleep() -->argint(0, &n) -->n = 5 -->acquire(tickslock) --> ticks0 = ticks -->ticks0 = 100 --> ticks - ticks0 < 5 ? --yes--> 检查 killed 状态 -->  sleep(&ticks,...) 当前进程 SLEEPING --CPU 执行其他进程--> 时钟中断发生（每当一个 Tick 到来时，时钟中断会唤醒等待 &ticks 的进程） -->  ticks++ --> wakeup(&ticks) -->当前进程变成 RUNNABLE (等待调度器)-->  以后重新获得 CPU(进程变为RUNNING) --> 回到 while 判断 --> 已经到 5 Tick --> release(tickslock) --> return 0

 sys_sleep() → sleep() → sched() → scheduler() → 定时器中断 → wakeup() → 进程从 SLEEPING 变成 RUNNABLE
其中：sched()，当前进程 A 已经不能继续运行了，请把 CPU 交给其他进程。当前进程放弃 CPU，进行上下文切换
scheduler()调度器：寻找并切换到下一个可以运行的进程
不同对象有不同的状态机。锁通常有“已占用/未占用”两种核心状态；进程则有 RUNNING、RUNNABLE、SLEEPING 等多种状态。对象的状态由对应的操作和系统事件改变，例如 acquire() / release() 改变锁状态，scheduler()、sleep()、wakeup()、exit() 等改变进程状态。