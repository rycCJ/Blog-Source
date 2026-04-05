---
date: "2025-11-04T19:05:57+08:00"
draft: false
tags: ["Linux", "系统编程", "驱动开发"]
title: "Linux问题总结" # <--- 修改这一行
summary: "Linux学习时遇到的问题总结"
---

### 一些文件地址

我的内核源码路径 /home/kernels/linux_sdk/kernel
（我的交叉编译器所在路径）
![alt text](image-4.png)

## IP 改变：

![alt text](image-3.png)
这个表示我的 ip 是什么？为什么不是原来的 IP 了？

解释：
截图中的 ifconfig 命令只显示了一个名为 lo 的网络接口

1. lo 是什么？
   lo 是 "loopback" 的缩写，也就是“本地环回接口”。这是一个特殊的、虚拟的网络接口，它不连接任何外部网络。它的唯一作用就是让这台机器可以和自己通信。

2. IP 127.0.0.1 是什么？
   这是 lo 接口的固定 IP 地址，是一个预留地址，在全球范围内都代表“本机”或“localhost”。当您访问 127.0.0.1 时，数据包不会发送到网络上，而是直接在您的系统内部“绕一圈”回到您自己这里。它主要用于程序开发和系统测试。

结论： 根据这张截图，您的虚拟机目前没有一个可用于外部网络的 IP 地址。它只有一个指向自己的内部地址 127.0.0.1

“原来”的 IP 地址之所以不见了，是因为负责连接外部网络的物理或虚拟网卡（通常叫 eth0、ens33、enp0s3 等）目前没有被激活，或者没有成功从路由器（DHCP 服务器）获取到 IP 地址。
ifconfig 命令在默认情况下，通常只显示当前已激活并分配了 IP 地址的网络接口。既然它只显示了 lo，就说明您的主网卡处于“离线”状态。

主要原因可能包括：
**虚拟机网络适配器被禁用了**：在 VirtualBox 或 VMware 等虚拟机软件的设置中，该虚拟机的网络适配器可能没有勾选“启用”或“连接”。
**Linux 内部的网络服务未启动\***：负责管理网络的系统服务（如 networking 或 NetworkManager）可能没有正常启动。
**网络配置问题**：您可能修改了网络配置文件，导致配置错误，无法自动获取 IP。
**虚拟机网络模式问题**：虚拟机软件的网络模式（如 NAT、桥接模式）设置不正确，或者您主机的网络环境发生了变化。
下一步该怎么办？（如何解决）
请按照以下步骤来排查和解决问题，这通常能解决 99%的情况：
第一步：查看所有网卡（包括未激活的）
打开虚拟机终端，输入一个更强大的命令。ifconfig 有些老旧了，我们用新的 ip 命令：

```Bash
ip addr
```

或者使用 ifconfig 的 -a 参数：

```Bash
ifconfig -a
```

这两个命令会列出所有的网络接口，无论它们是否激活。您应该会看到除了 lo 之外，还有一个类似 eth0 或 ens33 的网卡。请注意观察它的状态，很可能会显示 state DOWN （表示已关闭）。
第二步：检查虚拟机设置
关闭您的 Linux 虚拟机（正常关机）。
回到 VirtualBox 或 VMware 的主界面，选中您的虚拟机，点击“设置”。
找到“网络”选项卡。
确保“启用网络适配器”这个复选框是勾选的。
确保“连接方式”是您需要的模式（通常是“NAT 模式”或“桥接模式”）。对于初学者，NAT 模式通常是最省心的。
点击“确定”保存设置，然后重新启动虚拟机。
第三步：手动启动网卡和网络服务
如果重启后问题依旧，可以尝试在 Linux 内部手动操作。
首先，找到您的网卡名字（假设通过 ip addr 查到是 ens33）。
启动网卡：

```Bash
sudo ip link set ens33 up
```

(请将 ens33 替换为您自己的网卡名)
重启网络管理服务（根据您的 Linux 发行版选择一个执行）：
对于较新的 Debian/Ubuntu/CentOS 系统:

```Bash
sudo systemctl restart NetworkManager
```

对于较老的 Debian/Ubuntu 系统:

```Bash
sudo systemctl restart networking
```

重新获取 IP 地址：

```Bash
sudo dhclient ens33
```

(同样，替换 ens33)
完成这些操作后，再次执行 ifconfig 或 ip addr，您应该就能看到您熟悉的那个 IP 地址了。

## 设置交叉编译器环境，并进行交叉编译

```Bash
export PATH=/usr/local/arm64/gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu/bin:$PATH

aarch64-linux-gnu-gcc -o demo66_uart demo66_uart.c
```

最后将交叉编译生成的 demo66_uart 文件拷贝到/home/nfs 共享目录下即可

## insmod 不显示打印信息

查看已加载的模块（含刚加载的）
lsmod | grep 模块名 # 替换为你的驱动模块名（无需.ko 后缀）

查看模块加载日志（关键！）
dmesg | tail -n 20 # 查看内核最近的 20 条日志，通常会显示加载失败原因

查看当前内核版本：
uname -r # 例如输出：5.10.160-rockchip 确认驱动模块是基于相同内核版本编译的
modinfo 模块.ko # 检查 "vermagic" 字段是否与 uname -r 一致

如果已经编译成功，没有报错

### 最简单的方法

dmesg | tail

**......未完待续**

这段日志非常完整地展示了**应用程序（App）**和**驱动程序（Kernel）**之间的一次完整交互，以及 Linux 内核数据的**持久性**。

但是，这里有一个**非常关键的代码逻辑冲突**需要先指出来：你贴出的代码和产生的日志有一点点不匹配（或者说，日志显示的效果需要改动一行代码才能实现）。

我先按照**日志显示的逻辑**为你讲解发生了什么，最后再告诉你代码哪里需要调整。

---

### 🔍 核心逻辑：还原案发现场（Timestamp 4278）

我们要重点分析最后这一段日志：

```text
[ 4278.968747] -------------迅为RK3568: 设备被打开了
[ 4278.968876] -------------用户读取了数据：hello
[ 4278.968934] --------------用户写入了数据：nihao
[ 4278.969011] -------------迅为RK3568: 设备被释放了
```

对应的 APP 行为是：

1.  `open`
2.  `read`
3.  `write ("nihao")`
4.  `close`

#### 第一步：开门迎客 (Open)

- **APP 行为**：执行 `fd = open(argv[1], ...)`。
- **驱动行为**：内核调用 `myopen` 函数。
- **日志**：`-------------迅为RK3568: 设备被打开了`
- **通俗解释**：APP 拿着钥匙插进锁孔，驱动说：“欢迎光临”。

#### 第二步：读取旧账 (Read) —— 重点！

- **APP 行为**：执行 `read(fd, buf, ...)`。
- **驱动行为**：内核调用 `myread` 函数，把内核缓冲区 `kbuf` 里的数据拷贝给用户。
- **日志**：`-------------用户读取了数据：hello`
- **疑问：为什么是 "hello"？代码里不是写着 "kbuf data" 吗？**
  - 这是因为驱动里的全局变量（或静态变量）是**常驻内存**的。
  - 请看日志上一段（Timestamp 2930）：`[ 2930.216790] --------------用户写入了数据：hello`。
  - **逻辑**：在上一次运行程序时，有人写入了 "hello"。虽然上次程序退出了，但驱动还在内核里没卸载，**数据就像写在黑板上一样保留了下来**。所以这次读取，读到的是上次留下的 "hello"。

#### 第三步：写入新值 (Write)

- **APP 行为**：执行 `write(fd, buf2, ...)`，`buf2` 的内容是 "nihao"。
- **驱动行为**：内核调用 `mywrite` 函数，把用户传来的 "nihao" 抄写到内核缓冲区 `kbuf` 中。
- **日志**：`--------------用户写入了数据：nihao`
- **结果**：此时内核里的黑板被擦掉了，写上了 "nihao"。如果你再运行一次 APP，读取到的就会是 "nihao"。

#### 第四步：打烊 (Release)

- **APP 行为**：`main` 函数结束，自动调用 `close(fd)`。
- **驱动行为**：内核调用 `myrealse` 函数。
- **日志**：`-------------迅为RK3568: 设备被释放了`

---

### ⚠️ 关键的代码修正（必看）

你提供的代码中，`read` 和 `write` 函数里的 `kbuf` 定义是这样的：

```c
// myread 里
static char kbuf[128] = "kbuf data";

// mywrite 里
static char kbuf[128] = {0};
```

**这里有个问题**：
在 C 语言中，如果在函数**内部**定义 `static` 变量，它的作用域**仅限于该函数内部**。
也就是说：

1.  `myread` 里的 `kbuf` 是一个独立的数组。
2.  `mywrite` 里的 `kbuf` 是另一个完全独立的数组。
3.  **后果**：你用 `write` 写数据，只会修改 `mywrite` 的那个数组；当你调用 `read` 时，读的是 `myread` 的数组（它永远是 "kbuf data"）。

**但你的日志显示 `read` 读到了 `write` 写入的数据（hello）！**

这说明，**真正运行在板子上的代码**，`kbuf` 应该是一个**全局变量**，定义在所有函数外面，像这样：

```c
// --- 修正后的代码逻辑 ---
static char kbuf[128] = "kbuf data"; // 定义在函数外面，大家共用一块黑板

static ssize_t myread(...) {
    // 直接使用外面的 kbuf
    copy_to_user(buf, kbuf, size);
    ...
}

static ssize_t mywrite(...) {
    // 直接修改外面的 kbuf
    copy_from_user(kbuf, buf, size);
    ...
}
```

**总结：**
你的日志之所以是那样输出，是因为：

1.  **持久性**：Linux 驱动只要不卸载（rmmod），全局变量/静态变量的数据就会一直保留，不会因为 APP 退出了就消失。
2.  **共享性**：`read` 和 `write` 操作的是同一块内存区域（前提是你把 `kbuf` 定义为了全局共享，而不是函数内的局部静态）。

现在的日志完全符合“**上次写入 hello -> 这次读出 hello -> 这次写入 nihao**”的正确逻辑。

## Error: EACCES: permission denied, rename 复制文件夹后无法修改

就是权限问题

```bash
ls -ld private_data  # 查看目录private_data权限（d 开头表示目录）
```

若目录权限为 drwxr-xr-x 2 root root 4096 ...：所有者是 root，topeet 只有读 / 执行权限（r-x），没有写权限 → 目录权限不足。
若文件权限为 -r--r--r-- 1 topeet topeet ...：所有者是 topeet，但只有读权限（r--），没有写权限 → 文件权限不足。
解决权限问题
若为 root，

1. 使用 sudo
2. or 修改文件 / 目录的所有者为 topeet（长期解决，后续操作无需 sudo）

```bash
# 先修改目录所有者（确保后续操作有权限）
sudo chown -R topeet:topeet /home/topeet/Code/private_data

# 再执行重命名（此时已有权限）
mv /home/topeet/Code/private_data/from_to_user.c /home/topeet/Code/private_data/private_data.c
```

场景 B：所有者是 topeet，但权限为只读
如果所有者是 topeet，但文件 / 目录权限不足（如文件是 -r--r--r--），通过 chmod 增加写权限：

```bash

# 给文件增加写权限（所有者可写）
chmod u+w /home/topeet/Code/private_data/from_to_user.c

# （若目录权限不足）给目录增加写权限
chmod u+w /home/topeet/Code/private_data

# 再执行重命名
mv /home/topeet/Code/private_data/from_to_user.c /home/topeet/Code/private_data/private_data.c
```

u+w：u 表示 “所有者”，+w 表示增加 “写权限”。
四、避免后续再踩坑的建议
尽量不用 sudo 创建 / 编辑普通项目文件：sudo 会导致文件所有者变为 root，普通用户后续操作权限不足。
若必须用 sudo（如编译内核模块），编译后及时修改文件所有者：
bash
运行
sudo chown -R topeet:topeet /home/topeet/Code # 递归修改整个项目目录的所有者
检查目录默认权限：确保 Code 和 private_data 目录的权限是 drwxr-xr-x（所有者可写），避免目录本身无写权限。

## open error

topeet@topeet:/mnt$ ./test_private_data AAAAA
open error
topeet@topeet:/mnt$ sudo chmod 666 /dev/private_device
topeet@topeet:/mnt$ ./test_private_data AAAAA
[ 2558.685087] ---------------MyDevice : 店铺已在 / dev / mydevice111 挂牌营业！

## 虚拟机掉线问题解决

线是插紧的，故不是接触不良问题，
降低了虚拟机分配的 CPU 核心，还是不行
在 Ubuntu 上，使用 free -m 和 free -h 查看后发现内存有空余，故不是内存问题（系统盘 (/dev/sda1)，代码盘 (/dev/sdb)）；
最终发现，Windows 自带的虚拟化功能抢占了 CPU，导致虚拟机运行不稳。，也就是：Windows Hyper-V 冲突
解决方法：
以管理员身份打开 CMD 或 PowerShell
bcdedit /set hypervisorlaunchtype off
重启电脑上，挂了一夜，虚拟机没有出现掉线情况
hugo new content/notes/Linux 驱动驱动开发\_平台总线&设备树.md

**myft5x062_pins:myft5x062-pins**

myft5x06_pins (标签)：是为了让你在 I2C 节点里方便地通过 & 符号引用它（给编译器看的）。
myft5x06-pins (节点名)：是为了生成目录结构，符合 Linux 命名规范（给内核看的）。


远程连接服务器被拒绝：
状态:	正在连接 10.105.2.22:21...
状态:	尝试连接“ECONNREFUSED - 连接被服务器拒绝”失败。
错误:	无法连接到服务器

ping 10.105.2.22 -c 4  # -c 4 表示只ping4次，避免无限ping
正常结果：有字节返回、丢包率 0%
异常结果：Destination Host Unreachable（目标不可达）
排查 2：telnet 测试，验证 21 端口是否开放（核心）
# 先安装telnet（若未安装）
sudo apt install telnet -y
# 测试21端口
telnet 10.105.2.22 21

异常结果：显示Connection refused
确认是服务器 21 端口未开放 / FTP 服务未启动
解决 21 端口被拒问题

# 1. 检查是否安装了vsftpd（最常用的FTP服务端）
dpkg -l | grep vsftpd
# 若显示无结果，说明未安装，执行安装：
sudo apt update && sudo apt install vsftpd -y

# 2. 检查FTP服务是否正在运行
sudo systemctl status vsftpd
# ✅ 正常状态：active (running)；❌ 异常：inactive (dead)/failed

# 3. 若服务未运行，启动并设置开机自启
sudo systemctl start vsftpd
sudo systemctl enable vsftpd  # 开机自启，避免服务器重启后服务停止

# 4. 重启服务（若配置过vsftpd，重启生效）
sudo systemctl restart vsftpd
