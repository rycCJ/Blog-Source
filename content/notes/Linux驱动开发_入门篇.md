---
title: "Linux驱动开发 入门篇" # <--- 修改这一行
date: "2025-11-11T19:17:40+08:00"
draft: false
tags: ["Linux", "驱动开发"]
location: ""
summary: "在这里写下您的文章摘要..."
---

### Linux 驱动的分类

1. 字符设备驱动（重要 ）：IO 的传输过程是以字符为单位的，没有缓冲。比如 I2C,SPI 都是字符设备
2. 块设备驱动：IO 的传输过程是以块为的单位的。与存储相关，如：tf 卡
3. 网络设备驱动：与前两个不同，都是以 socket 套接字节来访问的

### 驱动四个部分

1. 头文件
2. 驱动模块的入口和出口
3. 声明信息
4. 功能实现

### 编译驱动的两种方法

1. 将驱动编译成内核模块
2. 将驱动编译进内核

### 把驱动编译成内核模

参考文章：D:\Wechatfile\WeChat Files\wxid_jdbkf3gcd0e422\FileStorage\File\2025-11\【北京迅为】itop-3568 开发板驱动开发指南（重制版）1.8.pdf 第三章 内核模块实验

1. 写.c 文件和 Makefile 文件在同于一个文件夹下面

```Makefile
export ARCH=arm64
#设置交叉编译器前缀，依据具体开发板对应的交叉编译器前缀选择。
export CROSS_COMPILE=aarch64-linux-gnu-
#生成 helloworld.o 文件。helloworld.o 名称要与 helloworld.c 驱动名称保持一致。
obj-m += helloworld.o
#内核源码所在虚拟机 ubuntu 的实际路径，并且保证该路径下的内核源码已经编译通过，否则编译驱动会出现报错。具体编译方法请参考对应开发板使用手册，如 iTOP-RK3568
开发板编译手册。
KDIR :=/home/kernels/linux_sdk/kernel
PWD ?= $(shell pwd)
all:
make -C $(KDIR) M=$(PWD) modules
#make 操作
clean:
make -C $(KDIR) M=$(PWD) clean
#make clean 操作
```

2. 编译为内核模块
   输入 make 命令即可把 helloworld 驱动编译成内核模块
   内核模块是以 ko 为后缀名，因此编译成功得到的 helloworld.ko 文件即内核模块，也就是编译好的驱动程序
   make clean 命令可以清除编译文件
3. 模块加载与卸载
   将 helloworld.ko 内核模块拷贝到 iTOP-RK3568 开发板上,使用：

   ```
   insmod helloworld.ko
   ```

   命令加载 helloworld 内核模块，会执行驱动入口函数；

   ```
   rmmod helloworld.ko
   ```

   执行 rmmod helloworld 命令会希望在内核模块，在卸载内核模块的时候会执行驱动出口函数

### Linux 下编译驱动实践 USB 转串口

### make menuconfig 图形化配置

1.  在内核源码路径下输入：make menuconfig 即可打开图形化界面（内核源码：/home/kernels/linux_sdk/kernel）

2.  make menuconfig 图形化配置的操作
    输入"/"即可弹出搜索界面，然后输入想搜索的内容
3.  配置驱动状态
    (1) 将驱动编译成内核模块 M
    (2) 将驱动编译进内核 \*
    (3) 不编译
    使用空格按键配置这三种状态
4.  make menuconfig 有关文件
    Makefile：编译规则，告诉我们在 make 的时候要怎么编译 菜的做法
    Kconfig:内核配置选项 菜单
    .confiig: 配置完内核以后生成的配置选项 点完菜
5.  make menuconfig 会读取目录下的 Kconfig 文件
    Arch/&ARCH/目录下的 Kconfig

    内核源码/arch/arm/configs 下面有很多配置文件，相当于特色菜

6.  为何将/arch/arm/configs 下的文件复制为.config 到 内核源码/.config 而不复制成其他文件
    内核默认读取.config 作为默认配置选项

7.  复制的默认配置不符合要求怎么办？
    我们就要点菜，菜单是 Kconfig，通过 make menuconfig 来调出这个菜单，配置完以后自动更新到.config
8.  如何与 Makefile 文件建立联系？
    当 make menufig 保存退出之后，Linux 会将所有的配置选项以宏定义的形式保存在 include/generated/下面的 autocon.h 里面

### 把驱动编译进内核

视频没看懂思密达，但是文档讲的挺清楚......等学上几节再回来看

### 杂项设备

1.  是字符设备的一种。可以自动生成设备节点；可以通过 cat /proc/misc 命令来查看
2.  杂项设备除了比字符设备代码简单，是否还有别的区别？
    杂项设备的主设备号是相同的，均为 10，次设备号是不同的。主设设备号相同就可以节
    省内核的资源。
3.  主设备号和次设备号是什么？
    设备号包含主设备号和次设备号，主设备号在 Linux 系统里面是唯一的，次设备号不一定唯一。
    设备号是计算机识别设备的一种方式，主设备相同的就被视为同一类设备。
    主设备号可以比做成电话号码的区号。比如北京的区号是 010 ，次设备号可以比作成电话号码。
    主设备号可以通过命令 cat /proc/devices

4.  杂项设备的描述
    定义在内核源码路径下的：include/linux/miscdevice.h

```C
struct miscdevice  {
        int minor;  //次设备号
        const char *name;  //设备的节点的名字
        const struct file_operations *fops;  //文件操作集
        struct list_head list;
        struct device *parent;
        struct device *this_device;
        const struct attribute_group **groups;
        const char *nodename;
        umode_t mode;
};

extern int misc_register(struct miscdevice *misc);  //注册杂项设备
extern void misc_deregister(struct miscdevice *misc);  //注销杂项设备

```

    file_operation 文件操作集在定义在 include/linux/fs.h 下面，里面的的一个结构体成员对应一个调用

5. 注册杂项设备流程
   （1）填充 miscdevice 结构体
   （2）填充 file_operation 这个结构体
   （3） 注册杂项设备并生成设备节点

   **......未完待续**
