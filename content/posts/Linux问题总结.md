---
date: "2025-11-04T19:05:57+08:00"
draft: false
tags: ["Linux", "系统编程"]
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

## 共享目录生效
