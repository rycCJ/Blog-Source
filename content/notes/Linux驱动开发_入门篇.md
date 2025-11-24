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

## 编译驱动

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

```Bash
  insmod helloworld.ko
```

查看加载的模块

```Bash
lsmod
```

命令加载 helloworld 内核模块，会执行驱动入口函数；

```C
rmmod helloworld.ko
```

执行 rmmod helloworld 命令会希望在内核模块，在卸载内核模块的时候会执行驱动出口函数

4. 验证设备节点是否创建成功
   在终端里输入以下命令：

```Bash
ls -l /dev/hello_misc
# 这里的hello_misc是结构体里面的名字
```

如果成功，你会看到类似 crw-rw-rw- 1 root root 10, 58 Nov 13 10:00 /dev/hello_misc 的输出。
开头的 c 代表它是一个字符设备 (Character Device)。
10, 58 是它的主设备号 (Major Number) 和次设备号 (Minor Number)。所有 misc 设备的主设备号都是 10。
如果失败，提示“No such file or directory”，说明你的驱动加载可能有问题。这时应该用 dmesg 命令查看内核日志。

5. 与设备进行交互 (有的话才操作 触发你的驱动代码)

假设你的 misc_fops 结构体里定义了 read 和 write 函数：

- 测试 write 函数：
  使用 echo 命令向设备文件“写入”数据。这个操作会触发你驱动里的 write 函数。

```Bash
# 需要root权限才能写入
sudo echo "some test data" > /dev/hello_misc
```

- 测试 read 函数：
  使用 cat 命令从设备文件“读取”数据。这个操作会触发你驱动里的 read 函数。

```Bash
cat /dev/hello_misc
```

6. 查看内核打印信息 (最重要的调试手段)

你的 read 和 write 函数里，一定有 printk 语句吧？printk 的信息不会显示在你的当前终端里，而是被放到了内核的环形缓冲区中。你需要用 dmesg 命令来查看它们。

```Bash

# 执行完 echo 或 cat 操作后，立刻执行 dmesg

dmesg
```

你应该能在 dmesg 输出的末尾，看到你在 write 或 read 函数中用 printk 打印出来的信息，例如：

```
[12345.67890] hello_misc_write called! Received 15 bytes.
[12345.98765] hello_misc_read called!
```

看到这些打印，就证明你的驱动代码确实被成功调用了！

7. 卸载驱动：

```Bash
 sudo rmmod 1_misc
#  (注意，卸载时用模块名，不带.ko)
```

### Linux 下编译驱动实践 USB 转串口

### make menuconfig 图形化配置

1.  在**内核源码**路径下输入：make menuconfig 即可打开图形化界面（内核源码：/home/kernels/linux_sdk/kernel）

2.  make menuconfig 图形化配置的操作
    输入"/"即可弹出搜索界面，然后输入想搜索的内容
3.  配置驱动状态
    (1) 将驱动编译成内核模块 M
    (2) 将驱动编译进内核 \*
    (3) 不编译
    使用空格按键配置这三种状态
4.  make menuconfig **有关文件**
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

### 与图形化相关的文件

1. Kconfig:内核配置选项 菜单
   Kconfig 文件是图形化配置界面的的源文件，图形化配置界面中的选项由 Kconfig 文件决定。当我们执行命令 make menuconfig 命令的时候，内核的配置工具会读取内核源码目录下的 arch/xxx/Kconfig。xxx 是命令 export ARCH=arm 中的 ARCH 的值。然后生成对应的配置界面供开发者使用。
2. config 文件和.config 文件
   config 文件和.config 文件都是 Linux 内核的配置文件
   config 文件位于 Linux 内核源码的 arch/S(ARCH)/configs （arch/arm64/configs）目录下，是 Linux 系统**默认**的配置文件，要使用默认，操作：make config\_ 具体名字,这样就会生成.config
   .config 文件位于 Linux 内核源码的顶层目录下，编译 linux 内核时会使用.config 文件里面的配置来编译内核镜像。
   若.config 存在，make menuconfig 界面的默认配置即当前.config 文件的配置，若修改了图形化配置界面的设置并保存，则.config 文件会被更新若.config 文件不存在，make menuconfig 界面的默认配置则为 Kconfig
   文件中的默认配置使用命令 make xxx defconfig 命令会根据 arch/$(ARCH)/configs 日录下默认文件生成.config 文件。
3. Makefile 文件
   Makefile 文件里面包含了编译规则，告诉我们要如何编译 Linux。

### Kconfig 语法

详细请见：D:\study file\PDF\16\_开发板学习教程（重要）\【北京迅为】itop-3568 开发板驱动开发指南 v1.8.pdf 第七章 7.3

1. 主菜单
2. 配置选项
3. 依赖关系
4. 可选择项
5. source

### 把驱动编译进内核

以 helloworld 为例

1. 进入 drivers，这里面存放的是驱动。进入 char 目录下，这里存放的是字符设备。可以将 helloworld 存放到此。在这里面创建一个 hello 文件夹，里面创建一个 Kconfig
   想要在图形化界面里面添加 helloworld 选项

```C char/hello/Kconfig

```

2. 在 char 目录下的 Kcongfig 要将 hello 里面的 Kconfig 包含进去

```C char/Kconfig
source "drivers/char/hello/Kconfig"
```

3. 将写好的 helloworld.c 文件放入 hello 文件夹
4. 编译之前创建 Makefile，要让 Makefile 与图形化界面产生联系。进入内核源码目录查看.config 文件，复制（）CONFIG_helloworld.

```C char/hello/Makefile
obj-$(CONFIG_helloworld)       += helloworld.o
```

```C char/Makefile
obj-y           += hello/
```

5. 验证
   在内核源码目录下有.config 文件，而默认的.config 在内核源码下的 arch/arm64/configs 下的 rockchip_linux_deconfig，要是使用 make，生成新的.config 会覆盖 hello 下的.config 而无法编译。故需要将默认的.config 用 hello 的.config 替换掉

```C 内核源码下的.config(已配置helloworld)
cp .config arch/arm64/configs/rockchip_linux_deconfig
```

6. 编译内核镜像

```C /home/kernels/linux_sdk
./build.sh kernel
```

hello 文件下生成了中间文件

烧写内核源码下的 boot.img 文件到开发板

```
dmesg | grep "hello"
```

查看是否找到

视频没看懂思密达......等学上几节再回来看

## 杂项设备

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

   1. 填充 miscdevice 结构体
   2. 填充 file_operation 这个结构体
   3. 注册杂项设备并生成设备节点

### 驱动层传参

传参类型：

1. 基本类型：module_param
2. 数组：module_param_array
3. 字符串：module_param_string
   位置：/home/kernels/linux_sdk/kernel/include

### 内核符号表

B 模块使用 A 模块中的函数
A:里面要有 EXPORT_SYMBOL(函数)，这个函数是你 B 文件中要使用的函数/变量。EXPORT_SYMBOL(add);
B:声明你要是用的 A 里面的函数/变量。extern int add(int a, int b);
因为 B 里面用了 A 函数，所以要先编译 A
如果两个.c 在同一个目录下面：Makefile 中记得两个文件都要添加 obj-m
如果两个.c 不在同一个目录下面：编译完 A 之后生成的 Module.sumvers（符号表）要复制到与 B 同目录下

### 应用层和内核层数据传输

### 编译进内核的驱动如何运行

一路跳转

```C
#define module_init(x)	__initcall(x);

#define __initcall(fn) device_initcall(fn)

#define device_initcall(fn)		__define_initcall(fn, 6)

#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

#else
  #define ___define_initcall(fn, id, __sec) \
	static initcall_t __initcall_##fn##id __used \
		__attribute__((__section__(#__sec ".init"))) = fn;

// 例如，当 fn=hello_world、id=6、__sec=subsys 时，展开后为：
static initcall_t __initcall_hello_world6 __used
    __attribute__((__section__("subsys.init"))) = hello_world;


```

作用：是将函数注册为内核初始化过程的一部分，在系统启动时按特定优先级执行

Linux 内核启动过程需要执行大量初始化函数（如驱动初始化、子系统初始化等），这些函数通过不同的 initcall 宏注册，并按优先级分阶段执行：

- 内核定义了一系列封装好的宏（如 module_init、core_initcall、late_initcall 等），最终都会调用 **\_define_initcall，并传入不同的 id 和 **sec 来控制执行顺序（id 越小，优先级越高，越早执行）。
- 链接器会将所有 xxx.init 段的函数指针按顺序排列，内核启动时遍历这些指针并调用对应的函数，实现有序初始化。

\_\_\_define_initcall 是内核初始化机制的 “注册器”，通过函数指针和自定义链接段的结合，让内核能在启动时自动、按序执行所有注册的初始化函数。这也是驱动代码中 module_init(my_init) 能生效的底层原理（module_init 是对该宏的高层封装）。

_后半段没听懂_ 对应文件：D:\Wechatfile\WeChat Files\wxid_jdbkf3gcd0e422\FileStorage\File\2025-11\【北京迅为】itop-3568 开发板驱动开发指南（重制版）1.8.pdf

**......未完待续**
