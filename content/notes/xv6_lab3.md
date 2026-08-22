下面给的是 **MIT 6.S081 2020 `pgtbl` 分支**中“每进程内核页表 + 简化 `copyin/copyinstr`”的一套完整实现思路与参考代码。它覆盖你列出的全部要求。

核心目标是让每个进程同时拥有：

```
p->pagetable   用户页表：用户页带 PTE_U，只能用户态正常访问
p->kpagetable  内核页表：内核映射 + 本进程内核栈 + 用户页镜像（清除 PTE_U）
```

这也是官方实验的第二、第三部分。[实验说明](https://pdos.csail.mit.edu/6.S081/2020/labs/pgtbl.html)

---
# Lab3_1:Print a page table ([easy](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))
**中间节点（目录页）**：
- 作用：存**下一级页表的物理地址**，MMU 拿到 PPN，去内存读取下一级页表，继续地址查找。
- `PTE_V = 1`，**R、W、X 全部等于 0**。
**叶子 PTE（真正的数据/代码页，如 L0 层）**
- 作用：完成虚拟地址 → 物理地址翻译，MMU 到此停止，不再继续查下级页表。
- `PTE_V = 1`，为了能被程序读取或执行，它的 `PTE_R`、`PTE_W` 或 `PTE_X` 必有至少一个是 `1`。

# 1. 先理解最终结构

每个进程的 `kpagetable` 大致是：

```
低地址  0 ────────────────────────────── PLIC
        用户 text/data/heap/stack 的镜像
        但每一页都清除了 PTE_U
        因而：内核能直接解引用用户虚拟地址

高地址  PLIC ───────────────────────────
        UART / VIRTIO / PLIC
        kernel text
        kernel data / RAM 直接映射
        本进程专属 kernel stack
        trampoline
```

为什么要清除 `PTE_U`？

RISC-V 的 xv6 内核运行在 S-mode，默认不允许 S-mode 直接访问带 `PTE_U` 的页面。因此：

```
*kpte = *upte & ~PTE_U;
```

用户页表保留 `PTE_U`；内核页表中的镜像去掉 `PTE_U`。二者指向同一物理页，但权限不同。

---

# 2. `kernel/proc.h`：为进程增加内核页表

在 `struct proc` 中，紧挨着原有 `pagetable` 增加：

```
// User page table.
pagetable_t pagetable;

// Kernel page table used while this process runs in the kernel.
pagetable_t kpagetable;
```

最终局部结构类似：

```
struct proc {
  struct spinlock lock;

  enum procstate state;
  void *chan;
  int killed;
  int xstate;
  int pid;

  struct proc *parent;

  uint64 kstack;
  uint64 sz;
  pagetable_t pagetable;
  pagetable_t kpagetable;
  struct trapframe *trapframe;
  struct context context;
  ...
};
```

---

# 3. `kernel/defs.h`：增加函数声明

找到 VM 相关声明，增加：

```
void            kvmmap(pagetable_t, uint64, uint64, uint64, int);
pagetable_t     kvmmake(void);
void            kvmfree(pagetable_t);
int             uvm2kvmcopy(pagetable_t, pagetable_t, uint64, uint64);
```

`pgtbl` 分支通常已经提供 `vmcopyin.c`，若 `defs.h` 中还没有，也加入：

```
int             copyin_new(pagetable_t, char *, uint64, uint64);
int             copyinstr_new(pagetable_t, char *, uint64, uint64);
```

---

# 4. `kernel/vm.c`：创建每进程内核页表

## 4.1 改造 `kvmmap`

原先的 `kvmmap()` 默认操作全局 `kernel_pagetable`。改为显式接收目标页表：

```
void
kvmmap(pagetable_t kpgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(kpgtbl, va, sz, pa, perm) != 0)
    panic("kvmmap");
}
```

---

## 4.2 新增 `kvmmake()`

它创建一个“没有内核栈”的标准内核页表。内核栈要在 `allocproc()` 中为每个进程单独加入。

```
pagetable_t
kvmmake(void)
{
  pagetable_t kpgtbl;

  kpgtbl = (pagetable_t)kalloc();
  if(kpgtbl == 0)
    return 0;

  memset(kpgtbl, 0, PGSIZE);

  // UART registers.
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // VirtIO disk interface.
  kvmmap(kpgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // Platform-level interrupt controller.
  kvmmap(kpgtbl, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // Kernel text: read + execute, not writable.
  kvmmap(kpgtbl, KERNBASE, KERNBASE,
         (uint64)etext - KERNBASE, PTE_R | PTE_X);

  // Kernel data and usable physical RAM: read + write.
  kvmmap(kpgtbl, (uint64)etext, (uint64)etext,
         PHYSTOP - (uint64)etext, PTE_R | PTE_W);

  // Trampoline code.
  kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline,
         PGSIZE, PTE_R | PTE_X);

  return kpgtbl;
}
```

然后把原 `kvminit()` 改为：

```
void
kvminit(void)
{
  kernel_pagetable = kvmmake();
}
```

这时：

- `kernel_pagetable`：调度器空闲时使用的全局页表；
- `p->kpagetable`：某个进程在内核态运行时使用的页表。

---

## 4.3 新增 `kvmfree()`

`freewalk()` 要求传入的页表中已经没有任何叶子 PTE；否则会触发：

```
panic: freewalk: leaf
```

所以先取消所有内核叶子映射，再释放页表层级本身。

```
void
kvmfree(pagetable_t kpgtbl)
{
  uvmunmap(kpgtbl, UART0, 1, 0);
  uvmunmap(kpgtbl, VIRTIO0, 1, 0);
  uvmunmap(kpgtbl, PLIC, 0x400000 / PGSIZE, 0);

  // kernel text + kernel data/RAM 是连续的直接映射区
  uvmunmap(kpgtbl, KERNBASE,
           (PHYSTOP - KERNBASE) / PGSIZE, 0);

  uvmunmap(kpgtbl, TRAMPOLINE, 1, 0);

  // 此时所有叶子 PTE 都应清空
  freewalk(kpgtbl);
}
```

注意：这个函数**不负责**取消：

- 本进程内核栈；
- 本进程用户页镜像。

这两类映射属于进程自身，需要在 `proc_freekpagetable()` 中先处理。

---

# 5. `kernel/proc.c`：创建、切换、释放进程内核页表

## 5.1 `procinit()`：不要再在全局页表中分配内核栈

原始代码一般在 `procinit()` 中有：

```
char *pa = kalloc();
if(pa == 0)
  panic("kalloc");

uint64 va = KSTACK((int)(p - proc));
kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
p->kstack = va;
```

把这段删除或注释掉。

原因：以前所有进程共享 `kernel_pagetable`，所以内核栈映射建在全局页表中。现在每个进程有自己的 `kpagetable`，它的栈必须映射到自己的页表中。

如果你的 `procinit()` 末尾有第二次 `kvminithart()`，也可以删除；它原本主要用于刷新刚加入全局内核栈映射后的 TLB。

---

## 5.2 `allocproc()`：创建页表并映射专属内核栈

在 `p->trapframe` 分配成功后、创建用户页表前加入：

```
  // Create the process's private kernel page table.
  p->kpagetable = kvmmake();
  if(p->kpagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Allocate one physical page for this process's kernel stack.
  char *pa = kalloc();
  if(pa == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  p->kstack = KSTACK((int)(p - proc));

  // Map this physical stack page only into this process's kernel page table.
  kvmmap(p->kpagetable, p->kstack, (uint64)pa,
         PGSIZE, PTE_R | PTE_W);
```

之后保留原本代码：

```
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
```

为什么内核栈虚拟地址仍使用 `KSTACK(index)`？

```
每个进程的 kstack 虚拟地址保持固定
但它只映射在自己的 kpagetable 中
```

这样 `context.sp` 仍然稳定；调度器切换 `satp` 后再切换上下文，`sp` 就能在新页表中正确工作。

---

## 5.3 释放本进程的内核页表

在 `proc.c` 中新增：

```
static void
proc_freekpagetable(struct proc *p)
{
  // 用户页镜像：物理页由 p->pagetable 管理，不能重复释放。
  if(p->sz > 0)
    uvmunmap(p->kpagetable, 0, PGROUNDUP(p->sz) / PGSIZE, 0);

  // 内核栈物理页只属于这个 kpagetable，因此 do_free = 1。
  if(p->kstack)
    uvmunmap(p->kpagetable, p->kstack, 1, 1);

  // 取消其他内核映射并释放页表页。
  kvmfree(p->kpagetable);
}
```

然后在 `freeproc()` 中、把 `p->sz` 清零之前加入：

```
  if(p->kpagetable){
    proc_freekpagetable(p);
    p->kpagetable = 0;
  }

  p->kstack = 0;
```

完整释放顺序建议如下：

```
void
freeproc(struct proc *p)
{
  if(p->kpagetable){
    proc_freekpagetable(p);
    p->kpagetable = 0;
  }

  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;

  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;

  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
  p->kstack = 0;
}
```

最关键的是：

|映射|`uvmunmap(..., do_free)`|
|---|---|
|用户页在 `kpagetable` 中的镜像|`0`|
|内核栈|`1`|
|UART、PLIC、内核代码、RAM、trampoline|`0`|

用户页物理内存只能由原有的：

```
proc_freepagetable(p->pagetable, p->sz);
```

释放。否则会出现 double free。

---

# 6. 调度时切换 `satp`

找到 `scheduler()`。在调用 `swtch()` 前，切换到进程自己的内核页表：

```
if(p->state == RUNNABLE) {
  p->state = RUNNING;
  c->proc = p;

  // Switch to this process's kernel page table.
  w_satp(MAKE_SATP(p->kpagetable));
  sfence_vma();

  swtch(&c->context, &p->context);

  // Back in scheduler context: use global page table again.
  kvminithart();

  c->proc = 0;
}
```

逻辑是：

```
scheduler 使用全局 kernel_pagetable
        ↓
选中 p
        ↓
切到 p->kpagetable
        ↓
swtch 到 p 的内核栈
        ↓
进程运行、系统调用、Trap、yield、sleep
        ↓
swtch 回 scheduler
        ↓
恢复全局 kernel_pagetable
```

必须有：

```
sfence_vma();
```

因为切换 `satp` 后必须刷新当前 hart 的 TLB。

---

# 7. `trap.c`：确认从用户态回内核态时使用私有页表

检查 `usertrapret()`。

如果你看到的是：

```
p->trapframe->kernel_satp = r_satp();
```

通常不用改。因为调度器已经在进程运行前切换到：

```
p->kpagetable
```

所以这里读取到的正是当前进程内核页表。

如果你的源码写死成：

```
p->trapframe->kernel_satp = MAKE_SATP(kernel_pagetable);
```

则改成：

```
p->trapframe->kernel_satp = MAKE_SATP(p->kpagetable);
```

否则用户态 Trap 进入内核时会切回全局页表，后续 `copyin_new()` 无法直接使用用户虚拟地址。

---

# 8. 映射用户地址到进程内核页表

在 `kernel/vm.c` 新增：

```
int
uvm2kvmcopy(pagetable_t upgtbl, pagetable_t kpgtbl,
            uint64 oldsz, uint64 newsz)
{
  uint64 va;
  pte_t *upte;
  pte_t *kpte;

  if(newsz < oldsz)
    return -1;

  // PLIC 从 0x0c000000 开始；不能让用户映射与设备寄存器冲突。
  if(newsz >= PLIC)
    return -1;

  for(va = PGROUNDUP(oldsz); va < newsz; va += PGSIZE){
    upte = walk(upgtbl, va, 0);
    if(upte == 0 || (*upte & PTE_V) == 0)
      panic("uvm2kvmcopy: missing user pte");

    kpte = walk(kpgtbl, va, 1);
    if(kpte == 0)
      return -1;

    // 同一物理页、同样的 R/W/X/V 位，但内核页表不能保留 PTE_U。
    *kpte = *upte & ~PTE_U;
  }

  return 0;
}
```

这里故意不调用 `mappages()`，而是直接写 `*kpte`。

原因：`exec()` 会用新进程映像替换旧映像。若使用 `mappages()`，同一个 VA 已有旧映射时会触发：

```
panic: mappages: remap
```

直接覆盖叶子 PTE 可以安全替换：

```
旧用户页 VA 0x0 → old PA
新用户页 VA 0x0 → new PA

只改 kpagetable 的叶子 PTE 指向 new PA
```

旧物理页仍由旧 `pagetable` 持有，稍后由 `proc_freepagetable()` 释放。

---

# 9. 限制用户地址不能撞上 PLIC

在 `uvmalloc()` 开头加入：

```
if(newsz >= PLIC)
  return 0;
```

原因：

```
用户页镜像被放在 p->kpagetable 的低地址
PLIC 映射从 0x0c000000 开始
用户堆若增长到 PLIC，会覆盖设备寄存器映射
```

这正是实验要求限制用户地址空间的原因。

---

# 10. 在所有用户页表变化点同步映射

## 10.1 `userinit()`

原始代码通常是：

```
uvmfirst(p->pagetable, initcode, sizeof(initcode));
p->sz = PGSIZE;
```

紧接着加入：

```
if(uvm2kvmcopy(p->pagetable, p->kpagetable, 0, p->sz) < 0)
  panic("userinit: uvm2kvmcopy");
```

---

## 10.2 `fork()`

在 `uvmcopy()` 成功之后：

```
if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
  freeproc(np);
  release(&np->lock);
  return -1;
}

np->sz = p->sz;
```

紧接着加入：

```
if(uvm2kvmcopy(np->pagetable, np->kpagetable, 0, np->sz) < 0){
  freeproc(np);
  release(&np->lock);
  return -1;
}
```

注意必须是：

```
np->pagetable
np->kpagetable
```

而不是父进程的 `p->pagetable` / `p->kpagetable`。

---

## 10.3 `growproc()`：处理 `sbrk()`

正向增长时，先扩用户页表，再复制新增页范围到内核页表：

```
int
growproc(int n)
{
  uint64 sz;
  struct proc *p = myproc();

  sz = p->sz;

  if(n > 0){
    uint64 newsz = uvmalloc(p->pagetable, sz, sz + n);
    if(newsz == 0)
      return -1;

    if(uvm2kvmcopy(p->pagetable, p->kpagetable, sz, newsz) < 0){
      uvmdealloc(p->pagetable, newsz, sz);
      return -1;
    }

    sz = newsz;
  } else if(n < 0){
    uint64 newsz = uvmdealloc(p->pagetable, sz, sz + n);

    uint64 oldtop = PGROUNDUP(sz);
    uint64 newtop = PGROUNDUP(newsz);

    // 内核页表只是镜像，不释放用户物理页。
    if(newtop < oldtop)
      uvmunmap(p->kpagetable, newtop,
               (oldtop - newtop) / PGSIZE, 0);

    sz = newsz;
  }

  p->sz = sz;
  return 0;
}
```

缩小时要先后保持一致：

```
用户页表：取消映射并释放物理页
内核页表：只取消镜像映射，不释放物理页
```

---

## 10.4 `exec()`：替换进程映像

在 `exec()` 末尾、真正提交新页表前，通常能看到：

```
oldpagetable = p->pagetable;
oldsz = p->sz;

p->pagetable = pagetable;
p->sz = sz;
```

在这之前加入：

```
// Replace the low-address user mirror in p->kpagetable.
if(uvm2kvmcopy(pagetable, p->kpagetable, 0, sz) < 0){
  // Restore the old mapping if making the new mapping failed.
  if(uvm2kvmcopy(oldpagetable, p->kpagetable, 0, oldsz) < 0)
    panic("exec: kpagetable rollback");
  goto bad;
}

// If the old process image was larger, remove now-unused tail mappings.
if(PGROUNDUP(oldsz) > PGROUNDUP(sz)){
  uvmunmap(p->kpagetable, PGROUNDUP(sz),
           (PGROUNDUP(oldsz) - PGROUNDUP(sz)) / PGSIZE, 0);
}
```

然后保留原提交逻辑：

```
p->pagetable = pagetable;
p->sz = sz;
p->trapframe->epc = elf.entry;
p->trapframe->sp = sp;

proc_freepagetable(oldpagetable, oldsz);
return argc;
```

`exec()` 是这个实验最容易漏掉的位置。若忘记同步：

```
shell 可能能启动
但 exec 新程序后 copyin_new 会访问旧映射或无映射
usertests 常会失败
```

---

# 11. 用新实现替换 `copyin/copyinstr`

`pgtbl` 分支一般提供了 `kernel/vmcopyin.c`，里面已有：

```
copyin_new(...)
copyinstr_new(...)
```

它们不再自己走用户页表，而是直接访问当前用户虚拟地址，因为当前的 `p->kpagetable` 已经有用户页镜像。

在 `kernel/vm.c` 中，把原先冗长的 `copyin()` 替换为：

```
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}
```

把 `copyinstr()` 替换为：

```
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

`copyout()` 不需要按本实验要求修改。

---

# 12. `vmprint` 题中的文字答案

若 `answers-pgtbl.txt` 要求回答这三个问题，可写：

```
Page 0 contains the first page of init's user program, including its
text/data. Page 1 is the guard page below the user stack. Page 2 is the
user stack page. A user process cannot read or write page 1 because its
PTE_U bit has been cleared, even though the mapping is otherwise present.
```

中文理解：

|页|内容|
|---|---|
|page 0|`init` 的代码和数据|
|page 1|用户栈下方 guard page|
|page 2|用户栈|
|用户态能读写 page 1 吗？|不能，`PTE_U` 已被清除|

---

# 13. 测试顺序

先编译：

```
make clean
make qemu
```

进入 xv6 Shell 后：

```
usertests
```

正常应走到：

```
ALL TESTS PASSED
```

退出 QEMU 后运行：

```
make grade
```

重点关注：

```
pte printout
usertests
```

---

# 14. 常见报错定位

|症状|最可能原因|
|---|---|
|`panic: freewalk: leaf`|调 `freewalk()` 前漏掉 user mirror、kstack 或 kernel leaf 映射|
|`panic: mappages: remap`|`exec()` 时对已有用户镜像再次调用 `mappages()`|
|`kerneltrap`|进程 `kpagetable` 缺 kernel text、RAM、trampoline 或 kstack 映射|
|切换到第一个进程后死机|`scheduler()` 未切换到 `p->kpagetable` 就 `swtch()`|
|返回 scheduler 后异常|忘记 `kvminithart()` 恢复全局内核页表|
|`copyin_new` 触发 page fault|用户页镜像仍然带 `PTE_U`，或没有同步 `fork/exec/sbrk`|
|`usertests` 内存增长时失败|忘记在 `growproc()` 中同步新页|
|大用户地址时 panic/remap|忘记在 `uvmalloc()` 限制 `< PLIC`|
|fork 后子进程异常|`fork()` 中误把父进程映射复制到父进程 kpagetable，而不是 `np->kpagetable`|

完成这个实验后，你应该能清楚解释：

```
为什么全局 kernel_pagetable 无法直接访问用户指针？
为什么每进程内核页表能解决它？
为什么用户镜像必须去掉 PTE_U？
为什么 exec、fork、sbrk 都要同步映射？
为什么用户地址上限是 PLIC？
为什么释放 kpagetable 时不能释放用户物理页？
```


# `walk()` 流程
## 两个核心概念
- **硬件 CPU**：指的是芯片本身（硬件电路）。当 CPU 执行一条访存指令（如 `ld` 或 `sd`）时，CPU 内部的 **MMU（内存管理单元）** 会自动读取 `satp` 寄存器里的页表地址，在硬件电路层面自动完成虚拟地址到物理地址的转换。
    
- **软件（内核代码）**：指的是写在 C 语言文件里的代码逻辑（例如 `walk()` 函数）。它是操作系统开发者用 C 语言手写的程序，需要 CPU 执行多条指令来模拟硬件查页表的过程。
## 存在两套页表
1. **全局内核页表（`kernel_pagetable`）**：只有一个，里面映射了内核的代码、数据以及物理设备（如串口、磁盘）。**里面没有映射用户程序的数据**。
    
2. **独立的用户页表（`p->pagetable`）**：每个进程各有一个，里面只映射了该进程自己的代码、栈、堆等用户数据。
## **当用户发起 `write(fd, buf, len)` 系统调用时，发生了什么？**

1. **进入内核态**：CPU 陷入内核，`satp` 寄存器被切换为**全局内核页表**。
    
2. **遇到难题**：内核代码需要从 `buf`（用户虚拟地址）读取数据。但此时 CPU 处于内核态，使用的是**全局内核页表**。如果让硬件 CPU 直接去读 `buf`，硬件 MMU 会查全局内核页表，发现根本没有 `buf` 这个地址的映射，直接触发缺页异常崩溃。
    
3. **软件接管（调用 `walk()`）**：
    
    - 为了拿到数据，内核不能靠硬件 MMU 自动转换，只能用**软件**写好的 `walk()` 函数。
        
    - 内核把**进程独立的用户页表 `p->pagetable`** 作为参数传给 `walk()`。
        
    - `walk()` 在**软件层面**用 C 语言代码一层层读取 `p->pagetable` 的 L2、L1、L0 页表项（PTE），最后用 `PTE2PA` 计算出对应的物理地址 `pa`。
        
4. **硬件读物理地址**：内核拿到物理地址 `pa` 后，因为全局内核页表中对所有物理内存都有直接映射，此时硬件 CPU 就可以安全地去读取这个物理地址的数据了。
# 默认 xv6 的 `copyin` 是如何工作的？
在默认的 xv6 中，`copyin` 和 `copyinstr` 是内核用来**从用户空间安全读取数据到内核空间**的两个关键函数。理解它们的工作原理和性能痛点，是搞懂“每进程内核页表”这个实验的核心关键。
## `copyin` 是如何工作的
当用户程序发起系统调用的（如 `write(fd, buf, len)`）时，`buf` 是一个**用户虚拟地址**。
此时 CPU 处于内核态，`satp` 寄存器指向的是**全局内核页表**。而全局内核页表里只映射了内核自身的代码、数据和设备，**根本没有映射用户空间的 `buf` 地址**。如果内核直接去解引用 `buf`（如 `*buf`），CPU 就会因为找不到页表映射而崩溃。
为了拿到数据，默认的 `copyin` 必须用软件逻辑来“模拟”CPU 硬件查找页表的过程：

```
// C 
// 简化后的默认 copyin 实现逻辑 (kernel/vm.c)
int copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len) {
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(srcva); // 取出当前虚拟地址所在的页起始地址
    
    // 【核心痛点】手动查页表：调用 walk() 沿着 3 级页表一层层往下查
    pte_t *pte = walk(pagetable, va0, 0); 
    if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0)
      return -1; // 页表无效或不是用户页
      
    pa0 = PTE2PA(*pte); // 从页表项中提取出物理地址 (PA)
    
    // 计算当前页剩余可拷贝的字节数
    n = PGSIZE - (srcva - va0);
    if(n > len) n = len;
    
    // 终于拿到了真实的物理地址，通过物理地址进行内存拷贝
    memmove(dst, (char *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE; // 推进到下一个虚拟页
  }
  return 0;
}
```
## 默认实现的“核心痛点”在哪里？**

- **开销极大（逐页手动查页表）**：
    如果用户传输一段 16KB 的数据（跨越 4 个页面），内核就需要调用 4 次 `walk()` 函数。每次 `walk()` 都要在软件里进行 3 次指针跳转（L2 $\rightarrow$ L1 $\rightarrow$ L0）。CPU 硬件本身就有超快的寻页逻辑和 TLB 缓存，但内核却不得不白白浪费 CPU 周期去用**纯软件**跑一遍寻页流程。
    
- **无法直接利用硬件优化**：
    内核不能简单地使用 C 语言的 `memmove()` 或 `memcpy()` 一口气把数据拷过来，必须按页拆分、计算偏移量、手写循环，代码既复杂又低效。
    
    
# `copyin`改造后的方案与极速优化


这个实验的核心思路是：**让每个进程拥有自己的内核页表，并在进程的内核页表里，把该进程的用户空间内存也“顺手”映射进去。**

一旦映射建立完成，用户虚拟地址 `srcva` 在内核态下就变成了**合法且有效**的虚拟地址！

**改造后的 `copyin` 代码对比：**
```
// Lab 改造后的 copyin (只需要调用系统高效的 memmove)
int copyin_new(pagetable_t kpagetable, char *dst, uint64 srcva, uint64 len) {
  // 1. 简单检查地址是否超出用户空间范围，防止越权
  if (srcva >= PLIC || srcva + len < srcva || srcva + len >= PLIC)
    return -1;

  // 2. 直接拷贝！硬件 CPU 和 TLB 会自动帮我们完成虚拟地址到物理地址的转换
  memmove(dst, (char *)srcva, len);
  
  return 0;
}
```
|**对比项**|**默认 xv6 的 copyin**|**Lab 优化后的 copyin**|
|---|---|---|
|**地址翻译方式**|软件调用 `walk()` 手动查 3 级页表|硬件 CPU + TLB 自动翻译|
|**拷贝方式**|循环计算页偏移，按页分割拷贝|直接一行 `memmove()` 搞定|
|**性能**|极低（大量 CPU 耗在软件寻页上）|极高（接近硬件极限传输速度）|
|**实现代价**|页表简单，但内核操作繁重|每次用户页表更新时，需同步映射到进程内核页表|

# kernel/vm.c解读
`kernel/vm.c` 是 xv6 操作系统中**最核心的文件之一**，负责管理虚拟内存、页表创建、地址翻译以及内核与用户空间的数据传输。
以下是 `kernel/vm.c` 中最关键的几个核心函数详解：

**1. 内核页表初始化：`kvminit()` & `kvmmake()`**

这两函数负责在系统启动时建立**全局内核页表**：

- **`kvmmake()`**：调用 `kalloc()` 分配一个根页表页，然后使用 `mappages()` 建立内核虚拟地址到物理地址的映射：
     **映射 I/O 设备**：如 UART 串口、PLIC 中断控制器、VIRTIO 磁盘等（这些地址通过直接映射访问）。
        
    - **映射内核代码与数据**：从 `KERNBASE`（`0x80000000`）到 `PHYSTOP`（RAM 顶部）建立直接映射（`VA == PA`）。
    - **映射 Trampoline 页**：将中断处理跳板代码映射到虚拟地址的最顶端（`MAXVA - PGSIZE`）。
- **`kvminithart()`**：将创建好的内核页表物理地址写入当前 CPU 的 `satp` 寄存器，并执行 `sfence.vma` 刷新 TLB，正式开启分页机制。    

**2. 核心寻页引擎：`walk()`**

`walk()` 是整个页表系统的心脏，它用**软件模拟**了 CPU 硬件 MMU 查找三级页表的过程：
```C
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  // 循环拆解三级页表：L2 -> L1 -> L0
  for(int level = 2; level > 0; level--) {
    // 提取当前层级的 9 位 VPN 索引
    pte_t *pte = &pagetable[PX(level, va)];
    
    if(*pte & PTE_V) {
      // 页表项有效，取出下一级页表的物理地址
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      // 页表项无效：如果不允许分配或内存不足，返回 0
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      // 将新分配的页表物理地址写入当前页表项，并标记 PTE_V
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  // 返回 L0 层级最终指向数据页的 PTE 指针
  return &pagetable[PX(0, va)];
}
```
- **入参 `alloc`**：如果为 `1`，当发现中间节点页表不存在时，会自动调用 `kalloc()` 创建新的页表节点。
    
**3. 建立映射：`mappages()`**

将一段连续的虚拟地址映射到指定的物理地址：
```C
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  // 遍历要映射的每一个 4KB 页面
  for(a = last; ; a += PGSIZE, pa += PGSIZE){
    if((pte = walk(pagetable, a, 1)) == 0) // 寻页（若无则分配）
      return -1;
    if(*pte & PTE_V)
      panic("remap");
    // 将物理页号 (PPN) 与权限标志位 (perm | PTE_V) 写入 PTE
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
  }
  return 0;
}
```

**4. 用户页表的管理：`uvmcreate()`, `uvmalloc()`, `uvmdealloc()`**
用于管理进程的**用户页表**（`p->pagetable`）：

- **`uvmcreate()`**：创建一个新的空用户页表。
    
- **`uvmalloc()`**：当进程增加内存（如 `sbrk()`）时调用。调用 `kalloc()` 分配物理内存，并用 `mappages()` 映射到用户页表中，设置 `PTE_U | PTE_R | PTE_W` 等权限。
    
- **`uvmdealloc()`**：当进程释放内存时，取消映射并调用 `kfree()` 回收物理页。
    

**5. 进程销毁与复制：`freewalk()` & `uvmcopy()`**
- **`freewalk()`**：递归释放页表树结构。它只释放**页表本身的节点页**，不释放末端指向的数据页(不释放叶子PTE)。叶子 PTE 指向**用户程序的数据 / 代码物理页**，freewalk 不会释放用户内存。调用前提：必须先 uvmunmap 清除全部叶子 PTE。 for循环结束，本页表512项全部遍历完毕。本页表没有任何有效 PTE，调用`kfree`释放**当前这一级页表占用的 4KB 物理页**。
```c
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];

    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```
 
- **`uvmcopy()`**：在 `fork()` 时使用。它会为子进程创建一个全新的页表，并**分配新的物理页**深拷贝父进程的所有用户内存数据。
    

**6. 内核与用户态数据传输：`copyin()` & `copyout()`**

在默认 xv6 中，内核态下 `satp` 指向全局内核页表，无法直接解引用用户指针。

- **`copyin()`**：利用软件调用 `walk()` 查找用户页表，将数据从用户虚拟地址拷贝到内核缓冲区。
    
- **`copyout()`**：利用 `walk()` 查找用户页表，将内核数据写入用户虚拟地址。
    

**总结 `vm.c` 的宏观逻辑**

Plaintext

```
[操作系统需求]           [vm.c 函数实现]              [物理内存/硬件]
创建新进程     --->  uvmcreate() / uvmfirst()  ---> 分配物理页 (kalloc)
进程申请内存   --->  uvmalloc()                ---> mappages() 更新 PTE
硬件查地址     --->  (硬件 MMU 自动处理)        ---> 依靠 satp 根指针
内核读写用户页 --->  copyin() / copyout()      ---> 软件调用 walk() 查表
```
# **A kernel page table per process** 
要完成 **A kernel page table per process** 这个实验任务，我们需要打破 xv6 原有的“所有进程共享一个 `kernel_pagetable`”的逻辑，为每个进程单独维护一份内核页表。
以下是详细的代码修改步骤：

## 思路：
  要彻底理解“每进程内核页表（Per-process Kernel Page Table）”的设计，我们需要跳出繁琐的代码细节，从**操作系统为什么需要这样设计**的核心矛盾出发。

  

### 一、 核心痛点：默认 xv6 为什么要这样“折腾”？

在默认的 xv6 中，存在两条硬性规则：

  

1. **用户态**：CPU 使用进程的 **`p->pagetable`**（用户页表），里面只有用户代码、栈和堆。
    
      
    
2. **内核态**：CPU 陷入内核后，必须使用全局唯一的 **`kernel_pagetable`**（内核页表）。
    
      
    
所有进程进入 kernel 后，都进入同一个 kernel 地址空间。
**这就带来了一个巨大的矛盾**：

当用户程序发起系统调用（如 `write(fd, buf, len)`）时，传入的 `buf` 是一个**用户虚拟地址**。

但此时 CPU 在内核态，使用的是**全局内核页表**，全局内核页表里**根本没有映射用户内存 `buf`**。

  

如果你在内核里写 `*buf` 直接去读，CPU 发现全局内核页表中查无此页，就会立即引发 **Page Fault** 崩溃。

  

**原版的妥协方案**：

为了读到数据，内核不得不写一套 C 语言函数 `copyin()`，在里面调用 `walk()`。**用纯软件的方式去翻看进程的用户页表**，算出来物理地址后再去访问。这就相当于 CPU 拥有极快的自动寻页硬件（MMU），但内核却硬要用慢如蜗牛的软件代码去一步步爬页表。

  copyin()：“你给我一个用户虚拟地址，我去这个进程的用户页表里帮你找对应的物理页面，然后 kernel 再访问它。”
  
用户虚拟地址
      ↓
查 user pagetable
      ↓
找到 physical address
      ↓
kernel 访问 physical address

### 二、 终极解决方案：给每个进程发一个“定制版内核页表”

如果我们想在内核代码里直接写 `*buf` 指针解引用，并且让 CPU 硬件 MMU 自动帮我们完成物理地址翻译，**内核页表中就必须同时包含“内核代码”和“该进程的用户内存映射”**。

  

但是，全局只有一个内核页表，如果把所有进程的用户内存都映射到同一个全局内核页表里，不同进程的虚拟地址就会严重冲突（大家都是从 `0x0` 开始）。

  

**解决方案**：

取消全局唯一的内核页表，**为每个进程都单独复制一份内核页表（`p->kpagetable`）**。

  不是：用户页  复制一份给 kernel 页，而是两个 page table entry：指向同一个物理页面。
```
[进程 A 运行在内核态] ---> satp 指向 A 的 kpagetable (映射了内核设备/代码 + 进程 A 的内核栈)
                                       ↓ (下一阶段实验)
                                 (还将映射 进程 A 的用户内存)

[进程 B 运行在内核态] ---> satp 指向 B 的 kpagetable (映射了内核设备/代码 + 进程 B 的内核栈)
                                       ↓ (下一阶段实验)
                                 (还将映射 进程 B 的用户内存)
```
![425](Pasted%20image%2020260822141140.png)
### 三、 整套架构的核心设计脉络

为了支撑“每进程内核页表”的运转，整个操作系统的生命周期需要做四件事情：

  

#### 1. 副本构造（克隆内核基础环境）

- **思路**：每个进程的内核页表，本质上必须具备原版全局内核页表的基本能力（能访问 UART 串口、磁盘、中断控制器、内核代码段等）。
    
      
    
- **设计**：在 `allocproc()` 创建进程时，调用 `proc_kpagetable()`，给这个新进程分配一个根页表页，并把所有硬件设备和内核代码的物理映射关系**原封不动地抄一份**复制给它。
    
      
    

#### 2. 私有资源绑定（内核栈拆分）

- **思路**：原版 xv6 把所有进程的内核栈都打包挂在全局内核页表里。现在既然页表独立了，每个进程的内核页表就**只映射自己的内核栈**。
    
      
    
- **设计**：
    
      
    - 删掉启动时 `procinit()` 一次性映射所有内核栈的逻辑。
        
          
        
    - 改为在 `allocproc()` 中，每当申请一个新进程，才为它申请一块 4KB 物理内存作为内核栈，并使用 `mappages()` 把它**单独打通**到该进程的 `p->kpagetable` 中。
        
          
        

#### 3. 调度器无缝切换（SATP 寄存器的动态掌控）

- **思路**：CPU 在执行不同进程的内核代码时，`satp` 寄存器必须指向当前正在运行的那个进程的 `p->kpagetable`。
    
      
    
- **设计**：
    
      
    - 在 `scheduler()`（调度器）选中某个 `RUNNABLE` 进程并准备切入时，执行 `w_satp(MAKE_SATP(p->kpagetable))`，把 CPU 硬件的页表根指针切到该进程的内核页表，并执行 `sfence_vma()` 刷新 TLB。
        scheduler -> 找 RUNNABLE A -> 切换到 A -> 使用 A 的 kpagetable -> A 执行
          
        
    - 当进程让出 CPU、回到调度器自身运行时，再切回全局的 `kernel_pagetable`，保证内核无进程运行时的基础安全。
        当 A 让出 CPU：A ->  yield() -> sched() ->  scheduler
	    A → B 此时必须 kpagetable_A -> kpagetable_B
        

#### 4. 安全回收（只销毁结构，不销毁资源）

- **思路**：进程销毁（`freeproc`）时，它的 `p->kpagetable` 也必须被释放。但**绝对不能**把页表里映射的物理硬件（如 UART0、内核代码段物理内存）给 `kfree` 掉了！
    
      
    
- **设计**：写一个专门的 `uvmfree_kpt()` 函数，它会像树遍历一样递归往下找，**只释放 3 级页表树自身的页表页节点**，把所有的叶子数据页完好无损地留在物理内存里。
    
      
    

### 四、 一句话总结

**“每进程内核页表”的本质，就是通过在进程创建和调度时多做一点页表切换的维护工作，换取内核态下可以直接使用 CPU 硬件 MMU 自动翻译用户指针的极致性能。**
## 代码
allocproc()：创建进程的时候创建 kpagetable
proc_kpagetable()：创建 kernel page table


**1. 修改 `struct proc` (`kernel/proc.h`)**

  

在 `struct proc` 结构体中添加每进程内核页表字段：
```c
struct proc {
  // ...
  pagetable_t kpagetable;       // 每进程独立的内核页表
  // ...
};
```

**2. 在 `kernel/vm.c` 中增加创建与释放内核页表的函数**

  

我们需要一个类似 `kvminit()` 的函数来生成一份全新的内核页表（**注意：不要映射内核栈**，内核栈在 `allocproc` 中独立映射）。

proc_kpagetable()干三件事：
① 创建一个空的 page table 
② 把 kernel 必须的映射加入进去 
③ 返回这个 kernel page table

在 `kernel/vm.c` 末尾添加以下代码：

```C
/*
 * 为进程分配并初始化独立的内核页表
 */
pagetable_t
proc_kpagetable(struct proc *p)
{
  pagetable_t kpt = (pagetable_t) kalloc();
  if(kpt == 0)
    return 0;
  memset(kpt, 0, PGSIZE);

  // 映射全局硬件设备和内核代码段（与 kvminit 类似）
  if(mappages(kpt, UART0, PGSIZE, UART0, PTE_R | PTE_W) < 0 ||
     mappages(kpt, VIRTIO0, PGSIZE, VIRTIO0, PTE_R | PTE_W) < 0 ||
     mappages(kpt, PLIC, 0x400000, PLIC, PTE_R | PTE_W) < 0 ||
     mappages(kpt, KERNBASE, (uint64)etag - KERNBASE, KERNBASE, PTE_R | PTE_X) < 0 ||
     mappages(kpt, (uint64)etag, PHYSTOP - (uint64)etag, (uint64)etag, PTE_R | PTE_W) < 0 ||
     mappages(kpt, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree_kpt(kpt);
    return 0;
  }
  return kpt;
}

/*
 * 释放进程内核页表（只释放页表节点页，绝不释放底层映射的物理内存页）
 */
void
uvmfree_kpt(pagetable_t kpagetable)
{
  for(int i = 0; i < 512; i++){
    pte_t pte = kpagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // 说明这是下一级的中间页表节点
      uint64 child = PTE2PA(pte);
      uvmfree_kpt((pagetable_t)child);
      kpagetable[i] = 0;
    }
  }
  kfree((void*)kpagetable);
}
```

并在 `kernel/defs.h` 中声明这两个函数：


```c
pagetable_t     proc_kpagetable(struct proc *);
void            uvmfree_kpt(pagetable_t);
```

**3. 重构内核栈映射与 `allocproc()` / `freeproc()` (`kernel/proc.c`)**

  

原版 xv6 在 `procinit()` 中一次性把所有进程的内核栈都映射到了全局 `kernel_pagetable`。现在我们需要将其移至 `allocproc()` 中动态处理。

  

**修改 `procinit()`**：删掉为每个进程映射内核栈的代码。

```C
void
procinit(void)
{
  struct proc *p;
  initlock(&pid_lock, "nextpid");
  initlock(&wait_lock, "wait_lock");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
      // 注意：删除原有的 kstack 映射逻辑！
  }
}
```

**修改 `allocproc()`**：为新进程生成 `kpagetable`，并分配/映射其内核栈。

```c
static struct proc*
allocproc(void)
{
  struct proc *p;
  // ...（前面的 pid 获取和结构体寻找逻辑保持不变）

  // 1. 生成独立的内核页表
  p->kpagetable = proc_kpagetable(p);
  if(p->kpagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // 2. 分配物理页作为内核栈
  char *pa = kalloc();
  if(pa == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  uint64 va = KSTACK((int) (p - proc));
  
  // 3. 将内核栈映射到该进程的独立内核页表中
  if(mappages(p->kpagetable, va, PGSIZE, (uint64)pa, PTE_R | PTE_W) < 0){
    kfree(pa);
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  p->kstack = va;

  // ...（其余 trapframe、context 初始化保持不变）
}
```

**修改 `freeproc()`**：进程退出时，清理内核栈映射并释放 `kpagetable`。
```C
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;

  // 1. 取消内核栈在 kpagetable 中的映射并释放物理内存
  if(p->kstack){
    pte_t *pte = walk(p->kpagetable, p->kstack, 0);
    if(pte && (*pte & PTE_V)){
      kfree((void*)PTE2PA(*pte));
    }
    uvmunmap(p->kpagetable, p->kstack, 1, 0);
  }
  p->kstack = 0;

  // 2. 释放进程内核页表
  if(p->kpagetable){
    uvmfree_kpt(p->kpagetable);
  }
  p->kpagetable = 0;

  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  // ...
}
```

**4. 修改 `scheduler()` 进行切换页表 (`kernel/proc.c`)**

  

调度器选中进程运行时，将 CPU 的 `satp` 寄存器切至该进程的 `kpagetable`；当没有进程运行（或进程切换回调度器）时，切回全局 `kernel_pagetable`。
```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;
  
  for(;;){
    intr_on();

    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        p->state = RUNNING;
        c->proc = p;

        // 【关键逻辑】切入进程独立的内核页表并刷 TLB
        w_satp(MAKE_SATP(p->kpagetable));
        sfence_vma();

        swtch(&c->context, &p->context);

        // 【关键逻辑】进程让出 CPU 后，切回全局内核页表并刷 TLB
        kvminithart();

        c->proc = 0;
      }
      release(&p->lock);
    }
  }
}
```

**验证方式**

  

重新编译并运行测试：

  

Bash

```
make clean
make qemu
```

在 xv6 shell 中运行 `usertests`，若所有测试均通过（输出 `ALL TESTS PASSED`），则表明第一阶段实现无误。
# **为每个进程独立分配一份内核页表**”

  ## **1. 操作系统中的“内核态”与“用户态”**

  

- **双模式运行**：CPU 在运行普通程序时处于用户态（User Mode / U 态），在运行内核代码时处于内核态（Supervisor Mode / S 态）。
    
      
    
- **空间隔离**：用户程序不能随意访问内核内存，只有通过“系统调用”（System Call）陷入内核时，CPU 才会切换到内核态。
    
      
    

## **2. 传统 xv6 的内存映射架构（默认机制）**

  

- **全局内核页表**：默认情况下，xv6 只有一个全局的 `kernel_pagetable`。所有进程在陷入内核态执行代码（如处理系统调用或中断）时，CPU 的 `satp` 寄存器都指向这个全局内核页表。
    
      
    
- **独立的用户页表**：每个进程在用户态运行时，拥有自己独立的用户页表 `p->pagetable`。
    
      
    
- **地址空间切换**：在默认机制下，进程从用户态进入内核态，CPU 必须切换 `satp` 寄存器（从用户页表切到全局内核页表）。
    
      
    

## **3. 虚拟地址空间的“上限与重叠”**

  

- **Sv39 的地址上限**：在 RISC-V 39 位虚拟地址下，用户态程序占用的地址一般从 `0` 开始往上增长（如 `0x0` 到 `0x3FFFFFFFFF` 内的低地址区）。
    
      
    
- **为什么能合并**：用户程序占用的虚拟地址空间通常比较小（比如只用了几十 KB 到几 MB），而内核占用的虚拟地址大多在极高的位置（如 `0x80000000` 以上）。
    
      
    
- **Lab 的核心思路**：既然用户地址在低位、内核地址在高位，两者在虚拟地址空间上**互不冲突**，那为什么不把“内核的映射”和“该进程用户的映射”放到同一份内核页表中呢？
    
      
    

## **4. `copyin` / `copyinstr` 的工作原理与痛点**

  

- **默认做法**：当用户调用 `write(fd, buf, n)` 时，传入的 `buf` 是一个用户虚拟地址。此时 CPU 已经在内核态（用的是全局内核页表），全局内核页表里**没有**用户 `buf` 的映射。因此，内核必须用软件模拟 CPU 寻页（调用 `walk()` 函数），手动查用户页表找到物理地址，再把数据拷贝过来。这非常低效。
    
      
    
- **Lab 优化后的做法**：如果每个进程有自己专属的内核页表，且这份内核页表里**同时映射了内核代码和该进程的用户内存**，内核代码就能直接用 `memmove` 解引用 `buf` 指针，彻底省去复杂的页表查询。
    
      
    

## **5. 页表树的内存开销与深拷贝**

  

- **页表本身占用内存**：创建一个页表（即分配页表节点页）本身需要占用物理内存（通过 `kalloc()` 分配）。
    
      
    
- **映射 vs 内存本身**：映射（Mapping）只是在页表项（PTE）里填上物理页号（PPN），并不是把数据重新复制一份。为进程创建内核页表，是将内核已有的物理页地址（如设备寄存器、内核代码区）**重复填入**到新的页表树中。
    
      
    

把这些基础串联起来后，这个 Lab 的本质就很清晰了：**把“全局共享的一张大内核页表”，变成“每个进程各有一张内核页表”，并在其中顺手把该进程的用户态内存也映射进去，以此提升内核处理用户数据的效率。**
# 手工分解虚拟地址三级页表索引
虚拟地址在多级页表（如常见的三级页表）下的手工分解，主要是将一个**十六进制或二进制的虚拟地址**，按照操作系统或体系结构规定的位段（Bit Fields）切分成不同的部分。
以下以常见的 **48 位虚拟地址空间、4KB 页大小（12 位页内偏移）的三级页表结构** 为例说明分解步骤：

### **1. 明确结构与位数划分**

在一个标准的 4KB 页大小系统中，后 12 位固定为页内偏移量（$2^{12} = 4096$ 字节）。假设三级页表每级索引各占 9 位： 

- **PGD / L1 索引（一级页表）**：高位索引（如 9 位）
    
- **PMD / L2 索引（二级页表）**：中位索引（如 9 位）  
    
- **PTE / L3 索引（三级页表）**：低位索引（如 9 位）   
    
- **Offset（页内偏移）**：最低 12 位

### **2. 手工分解步骤**
设虚拟地址为：`0x7FFF800418`
**第一步：转换为二进制**

将十六进制补齐并展开为二进制（按每位十六进制对应 4 位二进制）：

`0x7FFF800418` $\rightarrow$ `0111 1111 1111 1111 1000 0000 0000 0100 0001 1000`

**第二步：按位数截取（从右往左）**

1. **Offset（12位）**：截取最低 12 位
    - 二进制：`0001 0001 1000` $\rightarrow$ 十六进制：`0x118`
        
2. **L3 / PTE 索引（9位）**：截取接下来的 9 位
    - 二进制：`000 0000 01` $\rightarrow$ 十进制：`1` / 十六进制：`0x001`
        
3. **L2 / PMD 索引（9位）**：截取接下来的 9 位
    - 二进制：`111 1100 00` $\rightarrow$ 十进制：`496` / 十六进制：`0x1F0`
        
4. **L1 / PGD 索引（9位）**：截取接下来的 9 位
    - 二进制：`011 1111 11` $\rightarrow$ 十进制：`255` / 十六进制：`0xFF`
        

### **3. 快捷计算技巧（十六进制掩码与位移）** 

不需要完全展开为二进制时，可以直接利用十六进制位移与掩码（以 4KB 页为例）： 

- **Offset** = `VA & 0xFFF`（取最后 3 位十六进制）
    
- **L3 索引** = `(VA >> 12) & 0x1FF`（右移 12 位后，取低 9 位）
    
- **L2 索引** = `(VA >> 21) & 0x1FF`（右移 21 位后，取低 9 位）
    
- **L1 索引** = `(VA >> 30) & 0x1FF`（右移 30 位后，取低 9 位）

# 解释切换页表为什么刷新TLB

切换页表时必须刷新 TLB（Translation Lookaside Buffer，页表缓存/快表），是因为 **TLB 中缓存的虚实地址映射关系发生了失效**。

具体原因可以分为以下三点：

**1. 防止地址翻译错误（脏数据污染）**

- **现象**：不同进程的虚拟地址空间是相互独立的，但它们的虚拟地址范围重合（例如，进程 A 和进程 B 都有一个虚拟地址 `0x1000`）。
    
- **后果**：如果不刷新 TLB，当 CPU 从进程 A 切换到进程 B 时，TLB 里还保留着进程 A 的 `0x1000 -> 物理地址 X` 的映射。进程 B 访问 `0x1000` 时，CPU 会直接命中 TLB，从而错误地读取到进程 A 的物理内存 `X`。故需要：
```
    csrw satp, t0 # 切换页表基址到进程 B 
	sfence.vma # 刷新 TLB（清空所有旧条目）
```

- eg:
	- **1. 初始状态（进程 A 运行）**
		- **进程 A 页表**：虚拟地址 `0x1000` → 物理地址 `0xA000`（保存着进程 A 的私密密码数据）
	    
		- **TLB 缓存**：`[VA: 0x1000 -> PA: 0xA000]`
	- **2. 发生切换（未刷新 TLB）**
		- 操作系统将 CPU 切换到**进程 B**，把页表基址寄存器 `satp` 指向进程 B 的页表。
		    
		- **进程 B 页表**：虚拟地址 `0x1000` → 物理地址 `0xB000`（保存着进程 B 的普通变量）
		    
		- _由于没执行 `sfence.vma`（刷新 TLB 指令），TLB 里依然残留着进程 A 的缓存条目。_
	- **3. 发生错误（进程 B 读取内存）**
		- 进程 B 执行代码读取自己的虚拟地址 `0x1000`。
		    
		- CPU 查找 TLB，发现命中 `0x1000`，直接返回物理地址 `0xA000`。
		    
		- **后果**：进程 B 成功读取到了进程 A 的私密密码（越权隔离失效），或者写入数据覆盖掉了进程 A 的内存（数据损坏）。
    

**2. 维护内存隔离与安全**

- 每个进程拥有不同的内存访问权限（如只读、读写、内核/用户态权限）。如果旧的 TLB 映射依然有效，新进程可能会借用旧进程的 TLB 缓存越权访问不属于自己的内存区域，导致崩溃或安全漏洞。
    

**3. 确保数据一致性**

- 切换页表意味着修改了 CPU 的页表基址寄存器（如 x86 的 `CR3` 或 RISC-V 的 `satp`）。TLB 只是页表在 CPU 内部的硬件高速缓存，**硬件通常不会自动将 TLB 与内存中新页表的内容同步**，必须由软件发出明确指令（如 `invlpg` 或 `sfence.vma`）强制清空旧缓存。
    

切换页表导致刷新 TLB 的核心原因可以概括为以下几点：

**1. 防止地址映射混淆（最关键原因）** 在多任务操作系统中，每个进程都有自己独立的虚拟地址空间。不同进程的相同虚拟地址（例如 `0x08048000`）通常对应不同的物理内存地址。

- 当 CPU 从进程 A 切换到进程 B 时，必须切换页表（通常是通过修改 CPU 的控制寄存器，如 x86 的 CR3）。
    
- 如果不刷新 TLB，TLB 中依然保存着进程 A 的“虚拟地址 → 物理地址”映射。
    
- 进程 B 运行时，CPU 访问某个虚拟地址时可能会误中 TLB 中属于进程 A 的旧条目，从而读取或修改了进程 A 的内存数据，导致严重的**数据损坏**和**内存安全漏洞**。
    

**2. TLB 的工作原理决定了其数据失效** TLB 本质上只是一层缓存。当底层的数据源（进程页表）发生改变时，上层的缓存数据就变成了过期数据（Stale Data）。为了保证内存地址转换的绝对正确，必须清除这些旧的缓存项。

**补充：现代 CPU 的优化机制（PCID / ASID）**

频繁刷新 TLB 会带来较大的性能开销（因为 TLB 未命中会导致慢速的内存页表查询）。为了减少不必要的 TLB 刷新，现代 CPU 引入了**进程上下文标识符（PCID / ASID）** 技术：

- 每一个 TLB 条目不仅记录“虚拟地址 → 物理地址”，还额外标记该条目属于哪一个进程（PCID）。
    
- 切换进程页表时，CPU 只需要切换当前的 PCID，而**不需要清空 TLB**。
    
- 当 CPU 查询 TLB 时，只有虚拟地址和 PCID 同时匹配才算命中。这样既保证了不同进程间的内存隔离，又避免了清空 TLB 带来的性能损失。