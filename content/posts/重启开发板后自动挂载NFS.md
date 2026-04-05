---
date: "2025-10-31T20:31:31+08:00"
draft: false
tags: ["Linux", "NFS", "系统编程"]
title: "重启开发板后自动挂载NFS" # <--- 修改这一行
summary: "针对：按照系统编程手册设置nfs实现共享目录，但是重启开发板之后开发板上无法同步共享目录的问题，本文实现了开机自动挂载"
---

按照文件：16\_开发板学习教程（重要）\【北京迅为】itop-3568 开发板系统编程手册（新）.pdf 中 12.3 章操作之后，开发板重启，无法同步共享目录。

## 共享目录生效

### 方法一

每次更新了共享目录之后需要在开发板上输入

```
sudo mount -t nfs -o nfsvers=3,nolock 10.105.2.107:/home/nfs /mnt
```

### 错误：mount.nfs: failed to apply fstab options

先试试：sudo！！！
第一步：检查 /etc/fstab 文件内容
在您的虚拟机终端中，打开并检查 /etc/fstab 文件。您可以使用 cat 或 less 命令来查看：
code
Bash
cat /etc/fstab
现在，请仔细查看文件的内容，重点关注以下几点：
寻找包含 /mnt 的行：是否存在类似下面这样的行？
code
Code

### 错误示例

10.105.2.28:/home/nfs /mnt nfs defaults,some_bad_option 0 0
寻找包含 10.105.2.28:/home/nfs 的行：这个 NFS 服务器地址是否被配置挂载到其他地方了？
检查整个文件是否有明显的语法错误：比如某一行字段数量不对、有奇怪的字符等。
第二步：修复 /etc/fstab 文件
根据您第一步的发现，进行如下操作：
情况 A：如果您找到了一行关于 /mnt 的配置
这是最可能的情况。您需要暂时禁用这一行。最好的方法是在该行的开头加上一个 # 字符，将其注释掉。
使用 nano 或 vim 编辑器打开文件：
code
Bash
sudo nano /etc/fstab
找到包含 /mnt 的那一行，在行首添加 #：

```bash
# 像这样注释掉它

# 10.105.2.28:/home/nfs /mnt nfs defaults 0 0
```

保存文件并退出 (在 nano 中是 Ctrl+X，然后按 Y，再按 Enter)。
情况 B：如果您没有找到 /mnt 的配置，但怀疑有其他语法错误
请仔细检查每一行，确保格式都正确：
<设备> <挂载点> <文件系统类型> <选项> <dump> <pass>
如果不确定，可以尝试将可疑的非系统关键行（比如您自己添加的）都注释掉。
第三步：重新执行挂载命令
在您注释或修复了 /etc/fstab 文件之后，无需重启，直接再次运行您原来的 mount 命令：
code
Bash
sudo mount -t nfs -o nfsvers=3,nolock 10.105.2.28:/home/nfs /mnt
(注意：挂载操作通常需要 sudo 权限)
此时，因为 fstab 中的冲突项已经被禁用，mount 命令将完全按照您在命令行中提供的选项来执行，应该就能成功挂载了。
第四步（可选）：验证挂载
执行以下命令查看是否挂载成功：
code
Bash
df -h
如果您在输出列表中看到了 /mnt 挂载点，那就说明成功了。

## 解决方案：编辑 /etc/fstab 实现开机自动挂载

1. 编辑 fstab 配置文件

```bash
sudo vi /etc/fstab
```

2. 添加 NFS 挂载条目

```bash
10.105.2.28:/home/nfs  /mnt  nfs
nfsvers=3,nolock,noatime,nofail  0  0
# 10.105.2.28:/home/nfs                      NFS 服务器的共享目录（你的虚拟机共享目录）
# /mnt                                                  开发板上的本地挂载点
# nfs                                                     文件系统类型为 NFS
# nfsvers=3,nolock,noatime,nofail      挂载参数，和你手动挂载的参数一致，其中nofail避免 NFS 连不上时卡住开机
#0 0                                                      是否备份（0 = 不备份）  是否开机自检（0 = 不自检）

```

3. 测试配置有效性

```bash
sudo mount -a
```

若没有报错，说明配置没问题；若报错，检查 IP、目录路径或参数是否写错。

4. 验证自动挂载
   重启开发板测试效果：

```bash
sudo reboot
```

重启后执行 `ls /mnt`，若能看到文件，说明自动挂载成功
