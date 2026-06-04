---
title: "Linux驱动开发 字符设备" # <--- 修改这一行
date: "2025-11-17T15:53:29+08:00"
draft: false
tags: ["Linux", "驱动开发"]
location: "广州"
---

## 🌟 核心概念：什么是字符设备？

在 Linux 的世界里，**一切皆文件**。
字符设备（Character Device）就是**像水流一样，按顺序一个字节一个字节传输数据的设备**。

- **例子**：键盘、鼠标、串口、LED 灯、蜂鸣器。
- **目的**：你的驱动程序要把硬件伪装成一个文件（例如 `/dev/my_led`），让应用程序可以通过 `open`、`read`、`write` 这些标准函数来控制硬件。

---

## 🏗️ 第一关：传统功夫 —— 手动打造字符设备驱动

**(知识点：申请设备号、注册 cdev、cdev_add)**

#### 🗣️ 通俗讲解

你要在 Linux 内核里开一家“店”（驱动），必须办两件事：

1.  **申请营业执照（设备号）**：
    - **主设备号**：代表行业（比如“这是 LED 行业”）。
    - **次设备号**：代表具体分店（比如“这是 LED 1 号店，那是 LED 2 号店”）。
    - _静态申请设备号_：`register_chrdev_region`（申请前需要`MKDEV(major, minor)`传入主次设备号）。
      - _动态申请设备号_：`alloc_chrdev_region`（让内核自动分配一个没被占用的号码，更安全）。
2.  **关联具体操作与营业执照**：

    - 拿到执照后，你要告诉工商局(内核)，顾客来了能做什么。比如，可以“开门”(`open`)，“读取商品信息”(`read`)，“下单”(write)。这些操作的集合，就是一个 `file_operations` 结构体。我们需要把这个结构体和设备号关联起来，这个过程叫注册字符设备 (`cdev`)。

### 💻 代码实践

这是一个最基础的驱动骨架，加载后内核里有了你的位置，但还没有文件节点。

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/cdev.h>

// 定义设备名
#define DEV_NAME "test_chrdev"

// 全局变量
static dev_t dev_num;       // 存放申请到的设备号(主+次)
static struct cdev my_cdev; // 字符设备核心结构体

// 1. 定义文件操作集 (店员的工作手册)
// 目前什么都不做，只是为了编译通过
static int my_open(struct inode *inode, struct file *file) {
    printk("迅为RK3568: 设备被打开了\n");
    return 0;
}

static struct file_operations my_fops = {
        /* 顾客来了能做什么,定义商店的经营方法 */
    .owner = THIS_MODULE,
    .open = my_open,
};

// 2. 驱动入口 (开业)
static int __init my_driver_init(void) {
    int ret;

    // A. 动态申请设备号
    // 参数: 存放结果的指针, 次设备号起始(0), 申请数量(1), 名字
    ret = alloc_chrdev_region(&dev_num, 0, 1, DEV_NAME);
    if (ret < 0) {
        printk("申请设备号失败\n");
        return ret;
    }
    printk("申请成功: 主设备号=%d, 次设备号=%d\n", MAJOR(dev_num), MINOR(dev_num));

    // B. 初始化cdev (绑定我们要用的操作函数集)
    cdev_init(&my_cdev, &my_fops);

    // C. 添加cdev到内核 (正式注册)
    ret = cdev_add(&my_cdev, dev_num, 1);
    if (ret < 0) {
        printk("注册cdev失败\n");
        unregister_chrdev_region(dev_num, 1); // 失败了记得退还设备号
        return ret;
    }

    printk("字符设备注册完成！\n");
    return 0;
}

// 3. 驱动出口 (关门)
static void __exit my_driver_exit(void) {
    // A. 删除cdev
    cdev_del(&my_cdev);
    // B. 归还设备号
    unregister_chrdev_region(dev_num, 1);
    printk("字符设备已卸载\n");
}

module_init(my_driver_init);
module_exit(my_driver_exit);
MODULE_LICENSE("GPL");
```

---

## 🚪 第二关：自动挂牌 —— 创建设备节点

**(知识点：class_create, device_create)**

### 🗣️ 通俗讲解

在第一关，你的店注册了，但是在街道上（`/dev/` 目录）没有门牌，用户找不到你。这个门牌就是` /dev` 目录下的一个文件，比如` /dev/mydevice`.我们希望加载驱动时，这个文件能自动出现；卸载时，自动消失。

1.  **创建类 (`class_create`)**：在 `/sys/class` 下建个分类目录。 相当于创建一条商业街
2.  **创建设备 (`device_create`)**：在分类下建个设备信息，系统会自动在 `/dev` 下生成对应的文件。在商业街上创建我们的“店面”，并取名

### 💻 代码实践

在第一关的基础上，增加自动创建节点的代码。

```c
// ... (保留上面的头文件和变量)
#include <linux/device.h> // 新增头文件

static struct class *my_class;   // 类
static struct device *my_device; // 设备

static int __init my_driver_init(void) {
    // ... (alloc_chrdev_region 和 cdev_add 代码同上，省略) ...

    // 1. 创建类 (会在 /sys/class/ 下生成 my_class_test 目录)
    // 创建一条“商业街”
    my_class = class_create(THIS_MODULE, "my_class_test");
    if (IS_ERR(my_class)) {
        return PTR_ERR(my_class);
    }

    // 2. 创建设备节点 (会在 /dev/ 下生成 my_device_node 文件)
    // 在商业街上创建我们的“店面”，并取名 "my_device_node"
    // 参数: 类, 父设备(NULL), 设备号, 私有数据(NULL), 设备名
    my_device = device_create(my_class, NULL, dev_num, NULL, "my_device_node");
    if (IS_ERR(my_device)) {
        class_destroy(my_class); // 失败了要回滚
        return PTR_ERR(my_device);
    }

    printk("设备节点 /dev/my_device_node 已自动生成\n");
    return 0;
}

static void __exit my_driver_exit(void) {
    // --- 新增步骤 3.销毁设备节点 (注意顺序：先销毁设备，再销毁类) ---
    device_destroy(my_class, dev_num);
    class_destroy(my_class);

    // 2.注销字符设备
    cdev_del(&my_cdev);
    // 1.归还设备号
    unregister_chrdev_region(dev, 1);
    printk(KERN_INFO "----------------MyDevice: 店铺已摘牌，打烊了！\n");
}
```

**验证**：加载驱动后，直接 `ls /dev/my_device_node`，你会发现文件已经存在了。

---

## 📦 第三关：数据搬运 —— 用户与内核的交互

**(知识点：copy_to_user, copy_from_user)**

### 🗣️ 通俗讲解

Linux 为了安全，把内存划分为**用户空间**（APP）和**内核空间**（驱动）。它们之间有一堵墙，不能直接互传指针。

- **APP 写数据给驱动 (`write`)**：驱动必须用 `copy_from_user` 把数据从墙外拉进来。顾客（用户）把东西给仓库（内核）

```c
 copy_from_user(void *to, const void __user *from, unsigned long n)
// to: 给谁？从用户空间拿来数据要给谁？当然是内核空间了。 kbuf(仓库自己的箱子)：目的地。

// from:从哪里来？当然是从用户空间来了。 buf (顾客手里的箱子)：源头。用户空间的数据地址。

// size (数量)：要拿多少个苹果。
```
当 应用程序 调用 write() 函数，想要写入或者发送数据给设备时。
怎么运作的？驱动.ko ─── 找到 fops.write ───> 执行 my_write() ───> 动用 copy_from_user
 my_write 被触发，copy_from_user(&text[*off], user_buf, to_copy) 把用户提交的 user_buf 数据，安全地复制并写入到内核的 text 数组中。

- **驱动发数据给 APP (`read`)**：驱动必须用 `copy_to_user` 把数据扔到墙外去。仓库（内核）把东西给顾客（用户）。

什么时候起作用？ 当 应用程序 调用 read() 函数，想要读取设备数据时。

怎么运作的？ 在你的代码中，my_read 被触发。内核里的数据存在 static char text[64] 数组中。copy_to_user(user_buf, &text[*off], to_copy) 把内核 text 数组里的内容，安全地复制到应用程序的 user_buf 缓冲区中。

- `copy_from_user` 和 `copy_to_user` 是对驱动而言；`read` 和 `write` 是对用户而言

### 💻 代码实践

实现一个简单的读写回显功能。

```c
#include <linux/uaccess.h> // 必须包含这个头文件

static char kbuf[128] = {0}; // 内核里的缓冲区

// 读函数：APP读取时调用
static ssize_t my_read(struct file *file, char __user *buf, size_t size, loff_t *ppos) {
    // copy_to_user(用户地址, 内核地址, 长度)
    // 返回值：没拷贝成功的字节数，0表示全成功
    int ret = copy_to_user(buf, kbuf, size);
    if(ret) {
        return -EFAULT;
    }
    printk("APP读走了数据: %s\n", kbuf);
    return size;
}

// 写函数：APP写入时调用
static ssize_t my_write(struct file *file, const char __user *buf, size_t size, loff_t *ppos) {
    // copy_from_user(内核地址, 用户地址, 长度)
    int ret = copy_from_user(kbuf, buf, size);
    if(ret) {
        return -EFAULT;
    }
    printk("APP写入了数据: %s\n", kbuf);
    return size;
}

// 记得把这两个函数填入 file_operations 结构体
static struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .read = my_read,
    .write = my_write,
};
```

---

## 🚀 第四关：极简模式 —— 杂项设备驱动 (Misc Device)

**(知识点：misc_register)**

### 🗣️ 通俗讲解

如果你觉得上面的 `alloc_chrdev` -> `cdev` -> `class` -> `device` 这一套流程太繁琐了，Linux 提供了一个**懒人包**：**杂项设备 (Misc Device)**。

- 它自动固定主设备号为 **10**。
- 它自动帮你申请次设备号。
- 它自动帮你创建 `/dev/` 下的节点。
- **非常适合**：简单的字符设备（蜂鸣器、看门狗、ADC 等）。

### 💻 代码实践

你会发现代码量瞬间减少。

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/miscdevice.h> // 核心头文件
#include <linux/fs.h>

// 定义操作函数
static int my_misc_open(struct inode *inode, struct file *file) {
    printk("Misc设备被打开\n");
    return 0;
}

static struct file_operations misc_fops = {
    .owner = THIS_MODULE,
    .open = my_misc_open,
};

// 定义杂项设备结构体
static struct miscdevice my_misc_dev = {
    .minor = MISC_DYNAMIC_MINOR, // 自动分配次设备号
    .name = "my_misc_test",      // 自动生成 /dev/my_misc_test
    .fops = &misc_fops,
};

static int __init my_misc_init(void) {
    // 一步注册！
    int ret = misc_register(&my_misc_dev);
    if (ret < 0) {
        printk("Misc注册失败\n");
        return ret;
    }
    printk("Misc驱动注册成功\n");
    return 0;
}

static void __exit my_misc_exit(void) {
    // 一步卸载
    misc_deregister(&my_misc_dev);
    printk("Misc驱动卸载\n");
}

module_init(my_misc_init);
module_exit(my_misc_exit);
MODULE_LICENSE("GPL");
```

---

## 🧩 第五关：高手进阶 —— 私有数据与一驱多用

**(知识点：private_data, container_of, 面向对象的思想)**

### 🗣️ 通俗讲解

这是从入门到进阶最关键的一步。
**问题**：如果 RK3568 板子上有 3 个 LED 灯，难道我要写 3 个驱动（led1_drv, led2_drv...）吗？
**答案**：当然不！我们要写**1 个驱动**，管理**3 个设备**。

**怎么做？**

1.  **定义一个结构体**：把每个 LED 的属性（比如 GPIO 号、名字）包起来。
2.  **使用 `private_data`**：当 APP 打开 `/dev/led1` 时，驱动在 `open` 函数里识别出是“1 号灯”，然后把 1 号灯的结构体指针存入 `file->private_data` 这个“背包”里。
3.  **读写时**：在 `write` 函数里，从“背包”里拿出数据，就知道现在操作的是哪个灯，而不会乱套。

没问题！这一关确实是 Linux 驱动开发中**从新手到老手**的分水岭。如果只用全局变量，那叫“裸机思维”；学会用 `private_data`，才算真正有了“Linux 驱动思维”。

我们换一个更直观的场景：**住酒店**。

---

### 🏨 通俗讲解：全局变量 vs 私有数据

假设你开了一家酒店（驱动程序），有一个房间（内存缓冲区）。

#### 1. 全局变量的做法（错误示范）

你酒店里**只有一个**公共储物柜（全局变量 `static char kbuf[128]`）。

- **客人 A（APP 1）** 入住，把他的 **钱包** 放进了柜子。
- **客人 B（APP 2）** 入住，把他的 **臭袜子** 放进了同一个柜子（覆盖了钱包）。
- **客人 A** 想要取回东西，结果拿出了一双 **臭袜子**。
- **后果**：客人打起来了，数据乱套了。

#### 2. 私有数据的做法（private_data）

Linux 内核设计了一套机制：

- **Open (办理入住)**：当前台（`open`函数）接到客人时，立马给客人分配一个**专属房间**（`kmalloc` 申请一块内存），并把**房卡**（内存地址）挂在客人的档案（`file`结构体）上的 `private_data` 字段里。
- **Write/Read (回房间办事)**：客人要存取东西时，必须出示档案。驱动通过 `file->private_data` 拿到房卡，找到**只属于这个客人的房间**进行操作。
- **Release (退房)**：客人走的时候，驱动回收房卡，并打扫房间（`kfree` 释放内存）。

**结论**：不管有多少个客人同时入住，每个人都只在自己的房间里折腾，互不干扰。

---

### 💻 代码实战：多 APP 数据隔离实验

这个驱动实现了：**APP 1 写入的数据，只有 APP 1 能读到；APP 2 写入的数据，只有 APP 2 能读到。** 即使它们打开的是同一个设备文件 `/dev/test_private`。

#### 1. 驱动代码 (`private_data_drv.c`)

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/slab.h> // 必须包含！为了使用 kmalloc 和 kfree

// 定义设备名
#define DEV_NAME "test_private"

// 核心：定义一个结构体，代表“一个客人的专属房间”
struct session_data {
    char buffer[128]; // 客人的私有储物柜
};

// 全局变量只用来管“执照”和“门牌”，不管具体数据
static dev_t dev_num;
static struct cdev my_cdev;
static struct class *my_class;
static struct device *my_device;

// --- 1. 打开设备 (办理入住) ---
static int my_open(struct inode *inode, struct file *file)
{
    // A. 为当前这个“客人”申请一块独立的内存 (开一间新房)
    // GFP_KERNEL 表示正常的内核内存分配
    struct session_data *new_data = kmalloc(sizeof(struct session_data), GFP_KERNEL);

    if (!new_data) {
        return -ENOMEM; // 内存不足，入住失败
    }

    // B. 初始化一下内存，清零
    memset(new_data, 0, sizeof(struct session_data));

    // C. 关键！！把房间钥匙(地址)挂在 file->private_data 上
    // 以后这个 file 指针不管传给 read 还是 write，我们都能找回这块内存
    file->private_data = new_data;

    printk("驱动日志: 这里的 open 被调用了，已分配私有内存地址: %p\n", new_data);
    return 0;
}

// --- 2. 写入数据 (存东西) ---
static ssize_t my_write(struct file *file, const char __user *buf, size_t size, loff_t *ppos)
{
    int ret;
    // A. 关键！！从 private_data 里拿出当事人的专属内存指针
    struct session_data *data = (struct session_data *)file->private_data;

    // 防止溢出
    if (size > 128) size = 128;

    // B. 把数据写入到“他的”buffer里，而不是全局变量里
    ret = copy_from_user(data->buffer, buf, size);
    if (ret) return -EFAULT;

    printk("驱动日志: 写入数据到私有内存(%p): %s\n", data, data->buffer);
    return size;
}

// --- 3. 读取数据 (取东西) ---
static ssize_t my_read(struct file *file, char __user *buf, size_t size, loff_t *ppos)
{
    int ret;
    // A. 关键！！拿出当事人的指针
    struct session_data *data = (struct session_data *)file->private_data;

    // B. 从“他的”buffer里读数据
    ret = copy_to_user(buf, data->buffer, size);
    if (ret) return -EFAULT;

    printk("驱动日志: 从私有内存(%p)读取数据: %s\n", data, data->buffer);
    return size;
}

// --- 4. 关闭设备 (退房) ---
static int my_release(struct inode *inode, struct file *file)
{
    // A. 取出指针
    struct session_data *data = (struct session_data *)file->private_data;

    // B. 释放内存 (一定要做！否则这块内存就丢了，这叫内存泄漏)
    if (data) {
        printk("驱动日志: 释放私有内存地址: %p\n", data);
        kfree(data);
    }

    return 0;
}

// 操作函数集
static struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .read = my_read,
    .write = my_write,
    .release = my_release,
};

// 驱动入口
static int __init my_init(void)
{
    // 动态申请设备号
    alloc_chrdev_region(&dev_num, 0, 1, DEV_NAME);

    // 初始化cdev
    cdev_init(&my_cdev, &my_fops);
    cdev_add(&my_cdev, dev_num, 1);

    // 自动创建节点
    my_class = class_create(THIS_MODULE, "private_class");
    my_device = device_create(my_class, NULL, dev_num, NULL, DEV_NAME);

    printk("驱动日志: 驱动加载成功，设备节点 /dev/%s\n", DEV_NAME);
    return 0;
}

// 驱动出口
static void __exit my_exit(void)
{
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    printk("驱动日志: 驱动卸载成功\n");
}

module_init(my_init);
module_exit(my_exit);
MODULE_LICENSE("GPL");
```

---

#### 2. 测试应用程序 (`app_test.c`)

为了验证效果，我们可以运行两个这个程序的实例。

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main(int argc, char *argv[])
{
    if (argc != 2) {
        printf("用法: ./app <要写入的字符串>\n");
        return -1;
    }

    int fd = open("/dev/test_private", O_RDWR);
    if (fd < 0) {
        perror("打开失败");
        return -1;
    }

    char write_buf[128];
    char read_buf[128] = {0};

    // 1. 写入用户传入的字符串
    strncpy(write_buf, argv[1], 128);
    printf("[APP] 正在写入: %s\n", write_buf);
    write(fd, write_buf, strlen(write_buf));

    // 2. 模拟一段长时间的等待，为了让我们有时间去运行另一个APP
    printf("[APP] 写入完成。现在睡眠10秒...\n");
    printf("[APP] 请赶紧去另一个终端运行 ./app <其他字符串> 来测试干扰！\n");
    sleep(10);

    // 3. 醒来后读取数据
    printf("[APP] 醒来了，准备读取...\n");
    read(fd, read_buf, 128);
    printf("[APP] 读取到的内容: %s\n", read_buf);

    // 4. 验证
    if (strcmp(write_buf, read_buf) == 0) {
        printf("[APP] 测试成功！数据没被别人覆盖。\n");
    } else {
        printf("[APP] 测试失败！数据被别人篡改了！\n");
    }

    close(fd);
    return 0;
}
```

---

#### 🧪 实验步骤（见证奇迹的时刻）

请在板子上打开 **两个** 终端窗口。

1.  **加载驱动**：

    ```bash
    insmod private_data_drv.ko
    ```

2.  **在 终端 A 运行 APP（写入 "AAAAA"）**：

    ```bash
    ./app AAAAA
    ```

    _它会提示你：“写入完成，睡眠 10 秒...”_

3.  **迅速在 终端 B 运行 APP（写入 "BBBBB"）**：

    ```bash
    ./app BBBBB
    ```

    _它也会写入并睡眠。_

4.  **观察结果**：
    - 如果用的是 **全局变量**：
      - 终端 A 醒来后，读到的会是 "BBBBB"（因为被后来者覆盖了）。
    - 如果用的是 **private_data**（上面的代码）：
      - 终端 A 醒来，读到的一定是 **"AAAAA"**。
      - 终端 B 醒来，读到的一定是 **"BBBBB"**。

#### 🔍 为什么会这样？（底层逻辑）

1.  终端 A 运行 `./app` -> 调用 `open` -> 驱动 `kmalloc` 了一块内存（假设地址 `0x1000`），挂在 A 的 `file->private_data` 上。
2.  终端 B 运行 `./app` -> 调用 `open` -> 驱动 **又** `kmalloc` 了一块新内存（假设地址 `0x2000`），挂在 B 的 `file->private_data` 上。
3.  A 写数据，写到了 `0x1000`。
4.  B 写数据，写到了 `0x2000`。
5.  A 读数据，驱动从 A 的 `file` 里取出 `0x1000`，读出 "AAAAA"。

这就实现了完美的**数据隔离**。这就是 Linux 驱动支持多用户并发访问的基础！

---

## 新增：官方文档私有数据处理方法

**（知识点：把全局变量打包，并通过指针传递）**
`struct device_test dev1`定义在函数外面，是全局变量。编译的时候内存就分配好了，一直在那儿。

### 做法

1.  **打包**：他不再零散地定义 `dev_t dev_num`、`struct cdev`、`char kbuf[]`，而是定义了一个大结构体 `struct device_test`，把所有跟这个设备有关的东西都塞进去。并且定义了一个**全局变量** `struct device_test dev1;`。
2.  **挂接**：在 `open` 函数里，他执行了 `file->private_data = &dev1;`。意思是：把这个**全局变量的地址**挂在这个文件指针上。
3.  **使用**：在 `write` 函数里，他不再直接使用全局变量名 `dev1.kbuf`，而是先取出指针 `test_dev = file->private_data`，再用 `test_dev->kbuf` 来操作。

### why（为什么不直接用全局变量 `dev1`？）

虽然在这个特定的简单例子里，直接用 `dev1.kbuf` 和用 `private_data` 效果一模一样，但他的目的是**为了规范化（面向对象思维）**。

### 优点

如果将来你的板子上有 2 个完全一样的设备（比如 dev1 和 dev2）。

- 如果不使用 `private_data`：你需要写两套 write 函数（`write_for_dev1` 和 `write_for_dev2`），或者在函数里写大量的 `if/else`。
- 如果使用 `private_data`：你的 `write` 函数**一行都不用改**！
  - 打开 dev1 时，`private_data` 指向 `&dev1`。
  - 打开 dev2 时，`private_data` 指向 `&dev2`。
  - `write` 函数只管操作指针，不管具体指向谁。

**总结官方代码：** 它是**静态**的用法。它主要解决的是**“驱动代码通用性”**的问题，让一套操作函数能服务于特定的硬件对象。

### 区别对比表

| 特性            | 官方代码 (全局变量 &dev1)     | 第五关 (kmalloc)                        |
| :-------------- | :---------------------------- | :-------------------------------------- |
| **内存来源**    | 全局静态区 (编译时决定)       | 堆区 (运行时动态申请)                   |
| **多 App 打开** | **共享同一个空间**            | **数据完全隔离**                        |
| **比喻**        | 公园的长椅 (大家坐同一把椅子) | 酒店的房间 (每人开一间房)               |
| **适用场景**    | **硬件控制** (比如 LED、GPIO) | **软件缓冲区** (比如虚拟串口、独立存储) |

### 场景还原：如果两个 APP 同时打开设备

- **官方代码的情况**：

  - APP A 打开设备 -> `private_data` 指向 `&dev1`。
  - APP B 打开设备 -> `private_data` 指向 `&dev1`。
  - **结果**：A 和 B 操作的是**同一块内存**。A 写进去的数据，B 能读到；B 写进去的数据，也会覆盖 A 的数据。
  - **为什么这样设计？** 因为对于**硬件**来说，物理上只有一个。比如板子上只有一个 LED 灯，无论 A 还是 B 来控制，控制的都是同一个物理寄存器。所以这里的 `dev1` 代表的是那个**唯一的物理实体**。

- **第五关代码 (`kmalloc`) 的情况**：
  - APP A 打开设备 -> `kmalloc` 申请了内存块 A -> `private_data` 指向 A。
  - APP B 打开设备 -> `kmalloc` 申请了内存块 B -> `private_data` 指向 B。
  - **结果**：A 和 B 操作的是**不同的内存**。互不干扰。
  - **为什么这样设计？** 这种通常用于**纯软件驱动**或者需要**每个用户独立上下文**的场景（比如每个连接都要有独立的接收缓冲区）。

---

## 第六关：使用 goto 跳转到错误处理处

你好！这一节的内容非常实战，也是区分“写着玩”和“工业级代码”的重要标准。

在 Linux 内核里，资源（内存、设备号、节点等）是非常宝贵的。如果你的驱动加载到一半失败了，却不把之前申请的资源还回去，就会造成**资源泄漏**（Resource Leak）。久而久之，系统就会因为资源耗尽而崩溃。

---

### 🏰 通俗比喻

假设你的驱动程序初始化（`init`）就像勇者要去城堡见国王，必须连过三道关卡：

1.  **第一关**：申请设备号（拿通行证）。
2.  **第二关**：注册 cdev（登记身份）。
3.  **第三关**：创建设备节点（进入大殿）。

#### ❌ 如果没有错误处理（耍流氓）

- 你过了第一关（拿到通行证）。
- 你过了第二关（登记了身份）。
- **第三关失败了**（大殿门锁坏了，进不去）。
- **结果**：你直接转身回家了。
- **后果**：由于你没退还通行证，也没注销身份，城堡名册里一直有你，但实际上你人不在。下次再来申请，管理员说：“你不是已经在里面了吗？”——**这就是资源泄漏/冲突。**

#### ✅ 正确的错误处理（倒叙撤销）

- **第三关失败时**：你不能直接回家。你必须**倒着走回去**。
- **处理**：
  1.  先去第二关，注销身份。
  2.  再去第一关，退还通行证。
  3.  最后才能回家报错。

---

### 🛠️ 为什么要用 `goto`？

在 C 语言里，如果有 5 个步骤，用 `if-else` 层层嵌套会写出“代码金字塔”，非常难看且容易出错。
Linux 内核约定俗成：**使用 `goto` 语句跳转到统一的错误处理代码段**。

**核心口诀：先进后出（FILO）**

- 第 1 个申请的资源，要在 **最倒霉**（最后一步才出错）的时候才释放，所以它的释放代码放在最下面。
- 第 3 个申请的资源，如果在第 4 步出错，它是第一个需要被释放的。

---

### 💻 代码实战：带错误处理的驱动框架

请仔细看下面代码中的注释，特别是 `goto` 跳转的位置和标号的顺序。

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/err.h> // 必须包含，用于 IS_ERR 和 PTR_ERR

static dev_t dev_num;
static struct cdev my_cdev;
static struct class *my_class;
static struct device *my_device;

static int __init my_driver_init(void)
{
    int ret;

    // --- 第一关：申请设备号 ---
    ret = alloc_chrdev_region(&dev_num, 0, 1, "error_test_dev");
    if (ret < 0) {
        printk("第一关失败：申请设备号挂了\n");
        // 第一关就挂了，之前啥也没干，直接返回错误，不需要 goto
        return ret;
    }

    // --- 第二关：注册 cdev ---
    cdev_init(&my_cdev, NULL); // 这里省略fops仅做演示
    ret = cdev_add(&my_cdev, dev_num, 1);
    if (ret < 0) {
        printk("第二关失败：cdev注册挂了\n");
        // 这里失败了，需要把第一关申请的设备号还回去
        goto free_dev_num;
    }

    // --- 第三关：创建类 ---
    my_class = class_create(THIS_MODULE, "test_class");
    // 注意：class_create 返回的是指针，判断指针错误要用 IS_ERR
    if (IS_ERR(my_class)) {
        printk("第三关失败：创建类挂了\n");
        ret = PTR_ERR(my_class); // 把指针错误转换成错误码(int)
        // 这里失败了，要撤销第二关(cdev)和第一关(设备号)
        goto del_cdev;
    }

    // --- 第四关：创建设备节点 ---
    my_device = device_create(my_class, NULL, dev_num, NULL, "test_device");
    if (IS_ERR(my_device)) {
        printk("第四关失败：创建设备节点挂了\n");
        ret = PTR_ERR(my_device);
        // 这里失败了，要撤销第三关(类)、第二关(cdev)、第一关(设备号)
        goto destroy_class;
    }

    printk("恭喜！所有关卡全部通过！\n");
    return 0;

// ===========================================
// 下面是“倒叙”的错误处理区
// ===========================================

destroy_class:
    // 撤销第三关
    class_destroy(my_class);

del_cdev:
    // 撤销第二关
    cdev_del(&my_cdev);

free_dev_num:
    // 撤销第一关
    unregister_chrdev_region(dev_num, 1);

    // 返回错误码给内核，告诉它加载失败
    return ret;
}

static void __exit my_driver_exit(void)
{
    // 正常卸载时的顺序，和错误处理是一样的：倒着拆
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    printk("驱动安全卸载\n");
}

module_init(my_driver_init);
module_exit(my_driver_exit);
MODULE_LICENSE("GPL");
```

---

### 🔍 关键知识点解释

1.  **标签（Label）命名规范**：

    - 标签名通常代表**“在这个地方要做什么清理工作”**。
    - 例如 `destroy_class:` 意味着跳到这里是为了销毁 class。
    - 也有人喜欢命名为 `err_step3`，看个人习惯，建议用动词更清晰。

2.  **执行流（瀑布流）**：

    - 注意看 `destroy_class:` 下面**没有 `break` 也没有 `return`**。
    - 这意味着，如果代码跳到了 `destroy_class`，它执行完 `class_destroy` 后，会**接着往下执行** `del_cdev`，再接着执行 `free_dev_num`。
    - 这正是我们想要的！因为如果在第 4 步出错，第 3、2、1 步申请的资源都需要释放。

3.  **IS_ERR 和 PTR_ERR**：
    - 有些函数（如 `class_create`）返回的是指针。如果失败了，它不会返回 `NULL`，而是返回一个特殊的“错误指针”（比如地址 0xFFFFFFF0）。
    - **判断方法**：必须用 `IS_ERR(ptr)` 来判断是否出错。
    - **获取错误码**：必须用 `PTR_ERR(ptr)` 把这个指针翻译成 `-ENOMEM` 之类的整数错误码返回给 `ret`。
    - (原因)传递真相：上层系统（比如 insmod 命令）接收到 ret 返回的 -12 后，它会查表，然后并在屏幕上打印出精美的报错信息："Out of memory"（而不是仅仅显示“加载失败”）。
    - 类型转换：my_class 是指针变量，ret 是整数变量。C 语言中，指针不能直接赋值给整数，必须通过 PTR_ERR 进行强制类型转换并还原数值。

### 🚀 总结

这一节的核心就是学会写**“后悔药”**：
**只要前面的步骤成功了，当前步骤失败了，就必须用 `goto` 跳到对应的错误处理段，把之前吃进去的资源全部吐出来。**

---
[YouTube教学视频](https://www.youtube.com/watch?v=hbSSi4bHF8E&list=PLCGpd0Do5-I3b5TtyqeF1UdyD4C-S-dMa&index=6)
[作者GitHub](https://github.com/Johannes4Linux/Linux_Driver_Tutorial)
cat /proc/devices //产看设备属于Character devices还是Block devices ，查看设备号

hexdump /dev/mmcblk0 | head //查看设备节点的前10行
mknod mymmc b 179 0 //创建设备节点 b 类型，设备号 179:0
ls -lh mymmc //查看设备节点的详细信息

major = register_chrdev(unsigned int major,"hello_cdev",&fops);
//major:major device number or 0 for dynamic allocation

```c hello_cdev.c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>

static int major;
static char text[64];

static ssize_t my_read(struct file *filp, char __user *user_buf, size_t len, loff_t *off)
{
	int not_copied, delta, to_copy = (len + *off) < sizeof(text) ? len : (sizeof(text) - *off);

	pr_info("hello_cdev - Read is called, we want to read %ld bytes, but actually only copying %d bytes. The offset is %lld\n", len, to_copy, *off);

	if (*off >= sizeof(text))
		return 0;

	not_copied = copy_to_user(user_buf, &text[*off], to_copy);
	delta = to_copy - not_copied;
	if (not_copied) 
		pr_warn("hello_cdev - Could only copy %d bytes\n", delta);

	*off += delta;

	return delta;
}

static ssize_t my_write(struct file *filp, const char __user *user_buf, size_t len, loff_t *off)
{
	int not_copied, delta, to_copy = (len + *off) < sizeof(text) ? len : (sizeof(text) - *off);

	pr_info("hello_cdev - Write is called, we want to write %ld bytes, but actually only copying %d bytes. The offset is %lld\n", len, to_copy, *off);

	if (*off >= sizeof(text))
		return 0;

	not_copied = copy_from_user(&text[*off], user_buf, to_copy);
	delta = to_copy - not_copied;
	if (not_copied) 
		pr_warn("hello_cdev - Could only copy %d bytes\n", delta);

	*off += delta;
	return delta;
}

static struct file_operations fops = {
	.read = my_read,
	.write = my_write
};

static int __init my_init(void)
{
	major = register_chrdev(0, "hello_cdev", &fops);
	if (major < 0) {
		pr_err("hello_cdev - Error registering chrdev\n");
		return major;
	}
	printk("hello_cdev - Major Device Number: %d\n", major);
	return 0;
}

static void __exit my_exit(void)
{
	unregister_chrdev(major, "hello_cdev");
}

module_init(my_init);
module_exit(my_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Johannes 4Linux");
MODULE_DESCRIPTION("A sample driver for registering a character device");
```

```c test.c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main() 
{
	int fd;
	char c;

	fd = open("/dev/hello0", O_RDWR);

	if (fd < 0) {
		perror("open");
		return fd;
	}

	while (read(fd, &c, 1))   //我要从这个文件读 1 个字节，帮我存到变量 c 的内存地址里
		putchar(c);

	close(fd);
	return 0;
}
```
```shell
insmod hello_cdev.ko
mknod /dev/hello0 236 0
```
insmod hello_cdev.ko 的作用：在内核内部注册了一个字符设备，告诉内核：“我是 hello_cdev，我的主设备号是 236，以后要是有人找 236 号设备，请执行我写好的 fops 函数集。”（但此时用户空间根本不知道怎么找到它）。

mknod /dev/hello0 236 0 的作用：在用户空间（/dev 目录下）创建一个叫 hello0 的文件，并给它盖上一个印章：主设备号 236，次设备号 0。

解惑： mknod 时名字叫 hello0 还是 hello_abc 根本无所谓，内核只认主设备号 236。当 test.c 打开 /dev/hello0 时，系统一看它的印章是 236，就会立刻转头去内核里找注册了 236 的那个驱动模块（也就是你的 hello_cdev）。


copy_to_user（内核 -> 用户）
什么时候起作用？ 当 test.c 调用 read() 函数，想要读取设备数据时。

怎么运作的？ 在你的代码中，my_read 被触发。内核里的数据存在 static char text[64] 数组中。copy_to_user(user_buf, &text[*off], to_copy) 把内核 text 数组里的内容，安全地复制到应用程序的 user_buf 缓冲区中。

copy_from_user（用户 -> 内核）
什么时候起作用？ 当 test.c 调用 write() 函数，想要写入或者发送数据给设备时（虽然你的 test.c 没写这段，但如果写了就会触发）。

怎么运作的？ my_write 被触发，copy_from_user(&text[*off], user_buf, to_copy) 把用户提交的 user_buf 数据，安全地复制并写入到内核的 text 数组中。


步骤：

1. 打开文件 ：test.c 执行 open("/dev/hello0", O_RDWR)。内核发现该文件的驱动主设备号是 236(可以使用grep hello /proc/devices 查看设备号)，于是把这个文件描述符 fd 和你的 fops（包含 my_read, my_write）绑定在一起。

2. 发起读请求 ：test.c 执行 read(fd, &c, 1)，意思是：“我要从这个文件读 1 个字节，帮我存到变量 c 的内存地址里”
3. 路由到驱动 : 内核的虚拟文件系统（VFS）收到请求，查到绑定关系，立刻调用驱动里的 my_read 函数，并把接收缓冲区的地址传给参数 user_buf（此时 user_buf 指向 test.c 里的变量 c）。
4. 执行内核打印：驱动进入 my_read，执行 pr_info("hello_cdev - Read is called...")。这就是为什么你能看到“read函数被调用了”的内核日志。
5. 跨空间搬运数据:驱动调用 copy_to_user，把内核中 text 数组的第 *off 个字符，精准地复制到用户空间的 user_buf（即变量 c）中。
6. 返回并显示：用户态">my_read 函数返回 delta（实际复制的字节数，这里是 1）。test.c 的 read() 成功拿到 1，退出阻塞，执行 putchar(c) 把字符打印在终端上。



如果不弄 class，你就必须在每次加载驱动后，手动在终端敲 mknod /dev/hello0 236 0。而弄了 class，驱动一加载，/dev/hello0 就会自动蹦出来。

1. class_create("my_class")：在 /sys/class/ 目录下创建一个属于你这个驱动的分类文件夹（例如 /sys/class/my_class/）。

2. device_create(.....,"hello0")：在这个分类文件夹下，创建一个设备节点信息，并向系统发送一个叫 uevent 的广播：“报告！有一个新设备诞生了，主设备号 XX，次设备号 XX，名字叫 hello0！” /sys/class/my_class/hello0

3. udev 守护进程：在用户空间死死盯着内核的广播。一听到这个消息，udev 啪的一下站起来，自动在用户空间执行了类似于 mknod /dev/hello0 c 主设备号 次设备号 的操作。

所以，class 和 device 的引入，本质上是内核向应用层**自动上报设备信息**的通道。

'''shell

sudo dmesg -W 
# 监听 uevent 广播
udevadm monitor
'''

**kmalloc & kzalloc**

kmalloc 申请的内存空间，里面的残余数据是随机的（上一个用过这块内存的程序留下的垃圾）。在你的代码中，申请完内存后直接拿来用了，没有用 memset 清零。那堆 ELF... 字符串，其实是某个刚刚死掉的进程留在内存里的残余垃圾。
```shell
pi@raspberry:~/Programming/Linux_Driver_Tutorial/11_kmalloc
$ echo "Hello World" | sudo tee //dev/hello0
Hello World
pi@raspberry:~/Programming/Linux_Driver_Tutorial/11_kmalloc
$ sudo cat /dev/hello0
ELF���������@8@pi@raspberry:~/Programming/Linux_Driver_Tutorial/11_kmalloc
```

1. 执行 echo "Hello World" | sudo tee /dev/hello0
    - tee 进程打开了设备 ──> 触发 my_open，分配了 内存块 A。
    - tee 进程将 "Hello World" 写入 ──> 触发 my_write，数据存入 内存块 A。
    - tee 进程结束并关闭文件 ──> 触发 my_release，kfree 释放了内存块 A！ 你的 "Hello World" 瞬间灰飞烟灭。  

2. 执行 sudo cat /dev/hello0
    - cat 是一个全新的进程，它重新打开了设备 ──> 触发 my_open，分配了一块全新的内存块 B。
    - cat 进程调用 read ──> 触发 my_read，把这堆乱七八糟的随机垃圾当成数据读了出来，显示在屏幕上！

**lseek**
lseek(fd, 0, SEEK_SET)，相当于强行把内核里的文件指针拨回到了最开头（0 的位置）。这样接下来的 read 才能老老实实地从 text[0] 开始重新读取。
