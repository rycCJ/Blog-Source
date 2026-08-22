## 第一周：页表、Trap、系统调用完整路径 📅 2025‑08‑17 ⏳ 2025‑08‑23
- [ ] 8月17日：页表原理与源码地图 📅 2025‑08‑17
    - [ ] 阅读源码：`kernel/memlayout.h`、`kernel/riscv.h`、`kernel/vm.c`、`kernel/kalloc.c`、`kernel/exec.c`
	    - [x] xv6 book Chapter 3 ✅ 2026-08-18
		- [x] `kernel/memlayout.h` ✅ 2026-08-18
		- [x] `kernel/riscv.h` ✅ 2026-08-18
		- [x] `kernel/vm.c` ✅ 2026-08-18
		- [x] `kernel/kalloc.c` ✅ 2026-08-18
		- [ ] `kernel/exec.c`
    - [x] 搞懂概念：虚拟地址、物理地址、页表、PTE ✅ 2026-08-18
    - [x] 搞懂 Sv39三级页表索引机制 ✅ 2026-08-18
    - [x] 搞懂函数：`walk()`、`mappages()`、`uvmalloc()`、`uvmfree()` ✅ 2026-08-18
    - [x] 搞懂：内核地址空间 / 用户地址空间 ✅ 2026-08-18
    - [ ] 搞懂 `satp`、`sfence.vma`
    - [x] 练习题：手工分解虚拟地址三级页表索引 ✅ 2026-08-20
    - [ ] 练习题：画出xv6内核虚拟地址布局
    - [x] 练习题：解释切换页表为什么刷新TLB ✅ 2026-08-20
    - [ ] 练习题：write用户指针为什么内核不能直接解引用
- [ ] 8月18日：完成 vmprint 📅 2025‑08‑18
    - [x] Page‑tables lab第一题：实现`vmprint()` ✅ 2026-08-18
    - [x] 打印完整三级页表 ✅ 2026-08-18
    - [ ] 分析PID 1页表输出结果
    - [ ] GDB单步跟踪一次`walk()`执行
- [ ] 8月19‑20日：每进程内核页表 📅 2025‑08‑19 ⏳ 2025‑08‑20
    - [x] 为每个进程构建独立内核页表 ✅ 2026-08-21
    - [x] 进程调度切换页表 ✅ 2026-08-21
    - [ ] 内核栈+guard页正确映射
    - [ ] 进程退出释放对应页表资源
    - [ ] 简化`copyin/copyinstr`
    - [ ] 运行`usertests`、执行`make grade`
		- [ ]  必须理解设计；
		- [ ] 必须完成每进程内核页表；
		- [ ] `copyin/copyinstr` 优化可以参考提示后完成，不必自己闭门试错两天。
- [ ] 8月21日：RISC‑V调用约定与Trap 📅 2025‑08‑21
    - [ ] 阅读 xv6‑book Chapter4（跳过4.6）
    - [ ] 阅读源码：`kernel/trampoline.S`、`kernel/trap.c`、`kernel/syscall.c`、`kernel/proc.h(trapframe)`
    - [ ] Traps‑lab RISC‑V assembly 汇编习题
	    - [ ] 参数放在哪些寄存器
	    - [ ] 理解 ra / sp / s0 寄存器作用
	    - [ ] caller‑saved、callee‑saved 寄存器区分
	    - [ ] 大小端字节序
	    - [ ] 参数数量与格式串不匹配现象
- [ ] 8月22日：Backtrace 栈回溯 📅 2025‑08‑22
    - [ ] 读取frame‑pointer栈帧指针
    - [ ] 遍历内核栈
    - [ ] 输出函数返回地址
    - [ ] 使用`addr2line`把地址映射源代码行号
    - [ ] 在panic()内部调用backtrace
- [ ] 8月23日：Alarm 📅 2025‑08‑23
    - [ ] 实现 sigalarm、sigreturn
    - [ ] trapframe保存、恢复用户现场
    - [ ] 防止handler重入
    - [ ] 跑通 alarmtest、usertests
    - [ ] 梳理链路：定时器中断 → 保存现场 → 修改返回地址 → 用户回调 → 恢复现场
- [ ] 第一周验收：默写完整write系统调用路径 📅 2025‑08‑23
    > 用户write() → usys.S → ecall → trampoline.S → usertrap() → syscall() → sys_write() → usertrapret() → trampoline.S → 返回用户态

## 第二周：中断、缺页、Lazy allocation、COW 📅 2025‑08‑24 ⏳ 2025‑08‑30
- [ ] 8月24日：中断与设备 📅 2025‑08‑24
    - [ ] 阅读 xv6‑book Chapter5
    - [ ] 阅读源码：`kernel/kernelvec.S`、`kernel/trap.c`、`kernel/plic.c`、`kernel/uart.c`、`kernel/console.c`
    - [ ] 区分：系统调用 / 异常 / 外部中断
    - [ ] 用户态Trap与内核态Trap两套入口为什么分开
    - [ ] UART中断到用户进程被唤醒完整链路
    - [ ] 中断处理函数为什么不能睡眠
    - [ ] sleep()必须搭配锁的原因
    - [ ] 以跟踪调用链为主，暂不写代码
- [ ] 8月25‑26日：Lazy allocation 惰性内存分配 📅 2025‑08‑25 ⏳ 2025‑08‑26
    - [ ] 完成 Lazy‑allocation lab全部
    - [ ] sbrk：仅扩大逻辑虚拟地址空间，不分配物理页
    - [ ] 用户访问触发page fault缺页异常
    - [ ] 缺页处理分配物理页，建立PTE映射
    - [ ] 处理边界：非法地址、负数sbrk、越界访问、内存耗尽
    - [ ] 修复fork、uvmunmap、copyin等周边bug
    - [ ] 理解问题清单
        - [ ] sbrk成功 ≠ 已经分配物理内存
        - [ ] faulting‑address保存在哪个寄存器
        - [ ] 缺页处理为什么需要页对齐
        - [ ] 合法缺页与非法内存访问如何区分
- [ ] 8月27‑29日：Copy‑on‑Write 写时复制 📅 2025‑08‑27 ⏳ 2025‑08‑29
    - [ ] 完成COW lab
    - [ ] fork父子共享物理页，清除PTE写位，打上COW标记
    - [ ] 写访问触发缺页异常，复制页面恢复写权限
    - [ ] 物理页引用计数实现
    - [ ] copyout处理COW页面
    - [ ] 跑通 cowtest、usertests
    - [ ] 重点思考
        - [ ] 父进程PTE同样清除写权限的原因；仅改子进程会出现什么bug
        - [ ] 引用计数增减时机
        - [ ] COW缺页 vs 只读段写异常怎么区分
        - [ ] 多核下引用计数为什么需要锁
        - [ ] TLB残留旧权限带来风险
    - [ ] 快速取舍备选：只完成设计、理解PTE标志、引用计数，阅读完整实现思路
- [ ] 8月30日：复盘各类缺页异常 📅 2025‑08‑30
    - [ ] 画图整理五类fault：正常访问、Lazy缺页、COW写缺页、非法地址、只读页写故障
    - [ ] 每一类记录：产生原因、内核判断条件、是否分配新物理页、是否杀死进程

## 第三周：调度、线程、并发和锁 📅 2025‑08‑31 ⏳ 2025‑09‑06
- [ ] 8月31日：调度器 📅 2025‑08‑31
    - [ ] 阅读 xv6‑book Chapter7
    - [ ] 阅读源码：`kernel/proc.c`、`kernel/swtch.S`、`kernel/spinlock.c`、`kernel/sleeplock.c`
    - [ ] 跟踪完整调用链：进程运行 →时钟中断 → yield() → sched() → swtch() → scheduler() →挑选新进程
    - [ ] 回答问题
        - [ ] swtch为什么只保存callee‑saved寄存器
        - [ ] trapframe 和进程上下文context的区别
        - [ ] 每个CPU独立scheduler线程原因
        - [ ] sleep(chan, lock)传入锁的目的
        - [ ] lost wakeup（丢失唤醒）问题
- [ ] 9月1日：Uthread 用户级线程 📅 2025‑09‑01
    - [ ] Multithreading lab：用户线程
    - [ ] 为线程准备独立栈空间
    - [ ] 寄存器保存与恢复
    - [ ] 实现 thread_switch
    - [ ] GDB单步调试汇编切换
- [ ] 9月2日：并行哈希表 📅 2025‑09‑02
    - [ ] pthread哈希表实验
    - [ ] 复现数据竞争丢失key现象
    - [ ] 加锁修复正确性
    - [ ] 从全局锁优化桶锁bucket‑lock
    - [ ] 对比1/2/4线程吞吐量，记录锁竞争、性能数据
- [ ] 9月3日：Barrier 屏障 📅 2025‑09‑03
    - [ ] 实现 barrier
    - [ ] mutex互斥锁
    - [ ] condition variable条件变量
    - [ ] generation轮次，避免上一轮线程闯入下一轮
- [ ] 9月4日：锁原理 📅 2025‑09‑04
    - [ ] 阅读 xv6‑book Chapter6
    - [ ] 原子指令原理
    - [ ] spinlock acquire/release
    - [ ] 内存重排序 memory‑ordering
    - [ ] spinlock vs sleeplock
    - [ ] 锁粒度、死锁
    - [ ] 锁与中断交互
    - [ ] false‑sharing伪共享
- [ ] 9月5日：每CPU内存分配器 📅 2025‑09‑05
    - [ ] Locks lab第一部分
    - [ ] 实现per‑CPU freelist，每个链表独立锁
    - [ ] 本地空闲不足，从其他CPU偷取内存steal
    - [ ] 对比修改前后锁竞争统计
- [ ] 9月6日：Buffer cache 缓冲缓存 📅 2025‑09‑06
    - [ ] Locks lab第二部分（完成或完成设计）
    - [ ] 消除全局bcache.lock瓶颈
    - [ ] block分桶，每桶独立锁
    - [ ] 保证同一个block只有一份buffer
    - [ ] buffer回收、迁移处理

## 第四周：文件系统和mmap 📅 2025‑09‑07 ⏳ 2025‑09‑13
- [ ] 9月7日：文件系统主干 📅 2025‑09‑07
    - [ ] 阅读 xv6‑book Chapter8，分层理解
        > 文件描述符 → file对象 → inode → 目录 → bmap块映射 → buffer cache →磁盘
    - [ ] 源码阅读：`file.c`、`sysfile.c`、`fs.c`、`bio.c`、`log.c`
    - [ ] 跟踪完整调用链 open → write → inode → bmap → bread/bwrite → log日志
- [ ] 9月8日：Large files 📅 2025‑09‑08
    - [ ] File‑system lab large‑files
    - [ ] direct直接块、single‑indirect一级间接块、double‑indirect二级间接块
    - [ ] 修改bmap()函数
    - [ ] 修改itrunc()截断，防止磁盘块泄漏
- [ ] 9月9日：日志与崩溃恢复 📅 2025‑09‑09
    - [ ] 理解预写日志WAL、transaction事务
    - [ ] commit提交点
    - [ ] buffer cache与磁盘持久状态
    - [ ] 元数据更新顺序错误造成文件系统损坏
    - [ ] write系统调用返回、page cache、真正磁盘落盘三者区别
    - [ ] symbolic links符号链接选做
- [ ] 9月10‑12日：mmap 📅 2025‑09‑10 ⏳ 2025‑09‑12
    - [ ] 完整完成mmap lab
    - [ ] VMA虚拟内存区域数据结构
    - [ ] mmap系统调用
    - [ ] 文件backed惰性缺页fault
    - [ ] munmap解除映射
    - [ ] MAP_SHARED写回磁盘
    - [ ] MAP_PRIVATE私有映射
    - [ ] fork继承VMA
    - [ ] 进程退出清理映射资源
    - [ ] 通过 mmaptest、usertests
- [ ] 9月13日：文件系统与内存复盘 📅 2025‑09‑13
    - [ ] mmap为什么不会立刻读取全部文件
    - [ ] 页面什么时候物理分配
    - [ ] MAP_SHARED 和 MAP_PRIVATE 实现层面区别
    - [ ] munmap为什么可能触发写盘
    - [ ] 进程退出不清理VMA会发生什么
    - [ ] mmap、page‑cache、DMA‑BUF三者联系区别

## 第五周：网络驱动与结束 xv6 📅 2025‑09‑14 ⏳ 2025‑09‑20
- [ ] 9月14日：驱动模型 E1000 📅 2025‑09‑14
    - [ ] 复习 xv6‑book Chapter5
    - [ ] 阅读源码 `kernel/e1000.c`、`kernel/pci.c`、`kernel/net.c`
    - [ ] 理解描述符环descriptor ring完整流程
        > 软件准备buffer →填充TX描述符 →更新tail →硬件DMA读取发送 →完成置位 →软件回收buffer
- [ ] 9月15‑16日：发送路径 📅 2025‑09‑15 ⏳ 2025‑09‑16
    - [ ] 实现 e1000_transmit()
    - [ ] 判断空闲描述符
    - [ ] 设置mbuf地址、长度
    - [ ] buffer所有权管理
    - [ ] 更新TX tail
    - [ ] 回收发送完成buffer
- [ ] 9月17‑18日：接收路径 📅 2025‑09‑17 ⏳ 2025‑09‑18
    - [ ] 实现 e1000_recv()
    - [ ] 判断接收完成标志位
    - [ ] 报文上交给网络栈
    - [ ] 分配新接收mbuf
    - [ ] 重置接收描述符，更新RX tail
- [ ] 9月19日：抓包调试 📅 2025‑09‑19
    - [ ] 使用packets.pcap，tcpdump分析ARP/IP/UDP
    - [ ] 验证收发数据
    - [ ] 排查问题：描述符环卡死、丢包、重复回收buffer
- [ ] 9月20日：xv6收尾 📅 2025‑09‑20
    - [ ] 所有重点lab make grade全部通过
    - [ ] Git每个实验独立commit保存
    - [ ] 写总结文档（不超过5页）
    - [ ] 画出五条核心调用链路
        1. 系统调用完整路径
        2. 定时器中断+调度路径
        3. page‑fault缺页异常路径
        4. 文件write写入磁盘路径
        5. 网卡收发路径
    - [ ] 整理遗留不懂问题清单
    - [ ] 正式转入 Linux / RK3576项目


  root((xv6 页表实现虚拟内存))
    硬件基础 RISC‑V Sv39分页
      地址规格
        虚拟地址：只用低39位，页面4KB(2^12)
        三级页表：每级9位索引 + 12位页内偏移
      PTE页表项
        PPN：物理页号
        标志位
          PTE_V 有效位，缺页异常
          PTE_R/W/X 读写执行权限
          PTE_U 用户态是否可访问
      CPU寄存器与指令
        satp：保存根页表物理地址，切换页表
        sfence.vma：刷新TLB缓存
        TLB：页表翻译高速缓存
    两套独立页表
      👉内核页表(全局唯一)
        大部分：直接映射「虚拟地址=物理地址」
        特殊不直接映射区域
          trampoline页：用户/内核切换汇编，所有进程共享该物理页
          每个进程独立kstack内核栈 + guard保护页
            guard页：PTE_V无效，栈溢出触发panic
        映射设备寄存器 UART/PLIC/CLINT
      👉进程用户页表(每个进程1套)
        用户地址空间布局 从低到高
          text代码段、data数据段
          user stack用户栈 + guard保护页
          heap堆（sbrk扩张）
          trapframe：保存用户全部寄存器
          trampoline页（映射到最高虚拟地址）
        特性
          进程切换时修改satp完成页表切换
          虚拟地址相同，映射不同物理页 → 进程内存隔离
          用户看到连续虚拟内存，物理内存可以离散
    核心软件模块 vm.c 页表操作函数
      walk()：软件模拟硬件，遍历3级页表，获取PTE；可按需分配下级页表
      mappages()：批量建立虚拟地址→物理地址映射，设置PTE权限
      kvminit()：初始化内核页表，构建内核各段映射
      kvminithart()：写入satp寄存器，开启分页 + 刷新TLB
      uvmalloc / uvmdealloc：用户页表分配、释放内存
      copyin / copyout：安全在内核‑用户空间拷贝数据
    物理内存分配器 kalloc.c
      粒度：以完整4KB物理页面分配释放
      数据结构：空闲链表 struct run，结构体存放在空闲页面自身
      接口
        kinit：初始化空闲链表
        kalloc：取出空闲物理页，返回物理地址
        kfree：释放物理页，填充1检测悬空指针
      保护：kmem.lock自旋锁，多核防止链表竞争损坏
    系统调用如何使用页表
      sbrk() 堆内存伸缩
        growproc：处理sbrk系统调用
        n>0：uvmalloc分配物理页，填充用户PTE
        n<0：uvmdealloc解除映射，kfree归还物理页
        xv6页表同时充当进程物理内存记录
      exec() 加载ELF可执行程序，重建用户地址空间
        校验ELF魔数 0x7F ELF
        创建全新空白用户页表
        根据proghdr段头加载程序，bss段填0
        分配用户栈，复制argv参数，模拟main栈布局，设置guard页
        全部构建完成才替换进程页表；失败释放新内存返回‑1
        安全检查：防止恶意ELF访问内核地址
    现实局限（xv6简化实现）
      不支持写时复制COW fork
      无延迟内存分配
      没有页面交换swap到磁盘
      没有超级页super‑page
      内核依赖直接映射；真实硬件内存布局不一定固定
      kalloc只有一条全局空闲链表，多核锁竞争性能差
    关键作用
      🛡内存隔离：不同进程互不干扰
      📦虚拟地址抽象：程序使用连续虚拟地址，无视物理内存碎片化
      🔒内存保护：PTE标志位限制用户态访问权限；guard捕获栈溢出
      🧩资源复用：trampoline同一份物理页被所有进程映射

不一定每建立一个文件夹都要改配置，取决于用途。

|目的|需要修改|
|---|---|
|仅用来整理文章|只创建文件夹和 `.md` 文件|
|让文件夹拥有栏目首页|添加 `<文件夹>/_index.md`|
|显示在网站顶部导航|修改 [menus.en.toml (line 74)](D:/Blog/config/_default/menus.en.toml:74)|
|使用 `hugo new` 自动生成专属模板|可选添加 `archetypes/<文件夹名>.md`|
|让 Obsidian 新笔记默认进入它|修改 [app.json](D:/Blog/content/.obsidian/app.json) 中的 `newFileFolderPath`|

## 推荐做法

例如新建“项目”栏目：

```
content/
└─ projects/
   ├─ _index.md
   ├─ project-a.md
   └─ project-b.md
```

`content/projects/_index.md` 内容：

```
---
title: "项目"
description: "我的个人项目"
---
```

然后在 [menus.en.toml (line 74)](D:/Blog/config/_default/menus.en.toml:74) 添加：

```
[[main]]
name = "项目"
pageRef = "projects"
weight = 20
```

这样网站顶部就会出现“项目”，访问地址通常是：

```
https://ryccj.github.io/projects/
```

## 多层文件夹

例如：

```
content/
└─ books/
   ├─ _index.md
   └─ weread/
      ├─ _index.md
      └─ 玩笑.md
```

如果 `weread` 也需要独立列表页面，就添加 `weread/_index.md`；如果只是整理文件，可以不添加。

你现在已经有：

```
content/Books/WeRead/玩笑……md
```

但 `Books` 和 `WeRead` 都没有 `_index.md`，也没有添加到导航菜单。如果希望它们成为正式栏目，建议至少增加：

- `content/Books/_index.md`
- 可选：`content/Books/WeRead/_index.md`
- [menus.en.toml](D:/Blog/config/_default/menus.en.toml) 中的 Books 菜单项

## Obsidian 的影响

当前 `Ctrl+N` 新建文件默认进入 `posts`。如果想默认进入 `books`，把：

```
"newFileFolderPath": "posts"
```

改成：

```
"newFileFolderPath": "books"
```

不过它只能指定一个默认目录。其他目录可以在 Obsidian 左侧文件栏中右键文件夹，选择“新建笔记”。

以后新文件夹建议使用小写英文，例如 `books`、`projects`、`tutorials`，中文名称放在 `_index.md` 的 `title` 中，可以减少 Windows 与 GitHub Actions/Linux 之间的路径大小写问题。

你说的应该是 **Templater**。它是 Obsidian 的“动态模板”插件，可以在创建笔记时自动填写标题、日期、Hugo 元数据，甚至自动重命名和移动文件。[官方说明](https://github.com/SilentVoid13/Templater)。

对你的 Hugo 博客特别有用。

## 典型用途

例如创建文章时自动生成：

```
---
title: "<% tp.file.title %>"
date: "<% tp.date.now("YYYY-MM-DDTHH:mm:ssZ") %>"
draft: true
tags: []
summary: ""
---

<% tp.file.cursor() %>
```

创建 `Linux内核学习.md` 后，Templater 会自动变成：

```
---
title: "Linux内核学习"
date: "2026-08-18T10:30:00+08:00"
draft: true
tags: []
summary: ""
---
```

它还可以：

- 弹窗询问文章标题、标签和摘要
- 根据标题自动重命名文件
- 自动把文件放进 `posts`、`notes` 或 `books`
- 给不同文件夹设置不同模板
- 插入当前日期、剪贴板内容等
- 通过 JavaScript 实现更复杂的自动化

## 与其他模板的区别

|功能|Hugo Archetype|Obsidian 内置模板|Templater|
|---|---|---|---|
|使用位置|`hugo new` 命令|Obsidian|Obsidian|
|插入固定内容|支持|支持|支持|
|自动日期、文件名|支持|基础支持|强|
|弹窗输入标题/标签|不支持|不支持|支持|
|按文件夹自动套模板|不支持|不支持|支持|
|自动移动、重命名|不支持|不支持|支持|
|JavaScript 自动化|不支持|不支持|支持|

## 最适合你的配置

可以准备三个模板：

```
content/.templates/
├─ post.md
├─ note.md
└─ book.md
```

然后建立文件夹对应关系：

- `posts` → `post.md`
- `notes` → `note.md`
- `Books` → `book.md`

这样在 `posts` 新建文章时自动生成 Hugo 博客头信息，在 `notes` 新建文件时使用笔记模板。

Templater 可以执行 JavaScript 和系统命令，因此只应使用自己编写或可信来源的模板，这是插件官方特别提醒的安全事项。对你这个博客来说非常实用，建议安装。