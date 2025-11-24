---
title: "字符设备驱动 LED" # <--- 修改这一行
date: "2025-11-24T20:50:51+08:00"
draft: false
tags: ["Linux", "驱动开发", "字符设备"]
location: "广州"
---

虽然现在的 Linux（如 RK3568）通常推荐使用设备树（Device Tree）和 GPIO 子系统（pinctrl/gpiod）来控制 LED，但为了学习**字符设备驱动**的底层原理，我们需要回归最原始的方法：**直接操作寄存器**。

这就好比现在的车都是自动挡（GPIO 子系统），但我们要学开车原理，得先学手动挡（直接操作寄存器）。

我将以 **RK3568** 为例，控制的 LED 接在 **`GPIO0_B7`** 这个引脚上

---

### 🕵️‍♂️ 前置知识：控制 LED 的三把钥匙

要控制一个 LED，我们需要操作三个寄存器（就像三把钥匙）：

1.  **复用寄存器 (IOMUX)**：决定这个引脚是干嘛的。引脚可以做串口、PWM，也可以做普通 GPIO。我们要把它设置为 **GPIO 模式**。
2.  **方向寄存器 (DDR - Data Direction Register)**：决定数据是进还是出。LED 是发光，所以要设置为 **输出 (Output)**。
3.  **数据寄存器 (DR - Data Register)**：决定输出电平的高低。写 1 灯亮，写 0 灯灭（或者反过来）。

---

### 📘 第一步：准备手册

你需要两份核心文档（RK3568 官方资料）：

1.  **RK3568 底板原理图**：查看引脚

- D:\study file\01*iTOP-RK3568 硬件资料\02*底板资料\01*原理图\02*底板原理图\iTOP_RK3568_C_MAIN_V191.pdf

2.  **RK3568 TRM (Technical Reference Manual)**（技术参考手册）：这是重点，用来查寄存器地址。**一定要 Part1 和 Part2 都备好**。

- D:\study file\01*iTOP-RK3568 硬件资料\01*核心板资料\07_RK3568 技术参考手册\Rockchip RK3568 TRM Part1 V1.1-20210301.pdf

---

### 🔍 第二步：寻找“复用寄存器” (IOMUX)

**目标**：找到 `GPIO0_C0` 的复用寄存器地址，并设置为 GPIO 模式。

在 RK3568 中，复用寄存器通常归 **GRF (General Register Files)** 管理。

- **注意**：`GPIO0` 比较特殊，它属于 **PMU 电源域**，所以它的复用寄存器在 **PMU_GRF** 里。
- 其他 GPIO（如 GPIO1, 2, 3...）通常在 **SYS_GRF** 里。

#### 1. 查找基地址 (Base Address)

打开 **TRM Part 1**，搜索关键字 `Memory Map`（内存映射表）。在表中寻找 `PMU_GRF`：

- **PMU_GRF Base Address = `0xFDC20000`**

#### 2. 查找偏移量 (Offset)

打开 **TRM Part 1** 的 `GRF` 章节，搜索 `GPIO0B` 或 `IOMUX`。
你会找到一个寄存器叫 `PMU_GRF_GPIO0B_IOMUX_H` (L 代表低位，控制 B0-B3；H 代表高位，控制 B4-B7)。

- **PMU_GRF_GPIO0B_IOMUX_H Offset: `0x000C**

#### 3. 计算物理地址

> **复用寄存器地址 = 基地址 + 偏移量** > `0xFDC20000` + `0x000C` = **`0xFDC2000c`**

#### 4. 怎么设置值？

往下翻，查看对应的：`PMU_GRF_GPIO0B_IOMUX_H`
Address: Operational Base + offset (0x000C)
查看想要的 `GPIO0_B7`，控制 LED 灯需要将引脚复用设置为 GPIO0_B7，

`bit[12:14]` 控制 `GPIO0_B7`。手册会写：

- `3'h0`: `GPIO0_C0` (这是我们要的)
- `3'h1`: PWM\*M0 ...

所以，我们要把低 4 位设置为 `0`。

\*(注：Rockchip 寄存器通常有写保护，高 16 位对应写使能，后面代码里会讲)\_

使用`io -r -4 0xFDC2000C` 命令可以查看此寄存器的默认值,可以看出复用寄存器值为 00000001,此时[14:12]位为 000,此时 GPIO0_B7 引脚默认为 gpio 功能。因此无需在设置引脚复用为 gpio 功能

---

### 🚦 第三步：寻找“方向寄存器” (DDR)

这两个寄存器属于 **GPIO 控制器** 内部。

#### 1. 查找 GPIO0 的基地址（第一章的 Address Mapping）

- `GPIO0   0xFDD60000`

#### 2. 查找 DDR 的偏移量

Part 1 的 GPIO 章节（第十六章节）

- **GPIO_SWPORT_DDR_L** 0x0008
- **GPIO_SWPORT_DDR_H** 0x000C
  GPIOA0~GPIOA7, GPIOB0~GPIOB7 对应的方向寄存器偏移地址为 GPIO_SWPORT_DDR_L，
  GPIOC0~GPIOC7, GPIOD0~GPIOD7 对应的方向寄存器偏移地址为 GPIO_SWPORT_DDR_H。
  因此 GPIO0_B7 引脚方向寄存器的偏移地址为 **GPIO_SWPORT_DDR_L**，即 **0x0008**

#### 3. 计算物理地址

找到具体的 GPIO_SWPORT_DDR_L 描述：

```
31:16 （高16位使能低16位,对应 GPIOA0~GPIOA7、GPIOB0~GPIOB7 引脚方向控制位的写使能）
Write enable for lower 16 bits, each bit is individual.
1'b0: Write access disable
1'b1: Write access enable


15:0 （低 16 位控制方向,对应 GPIOA0~GPIOA7, GPIOB0~GPIOB7 方向控制位）
Data direction for the lower 16 bits of I/O Port, each bit is individual.
1'b0: Input
1'b1: Output

GPIO0_A0 ~ A7 (0-7)
GPIO0_B0 ~ B7 (8-15) -> 第 16 个引脚 B7 对应 bit15
GPIO0_C0 ~ C7 (16-23)
```

- 使能 bit15：对应 bit31 写 1
- 控制方向：bit15 写 1 对应输出，0 对应输入
  **方向寄存器的地址（DDR）** = 基地址+偏移地址=`0xFDD60000` + `0x0008` = **`0xFDD60008`**
  使用 `io -r -4 0xfdd60008` 命令查看方向寄存器的默认值，方向寄存器第 15 位默认为 1，因此 GPIO0_B7 默认输出。所以无需在设置 GPIO0_B7 方向

---

### 第四步：寻找“数据寄存器” (DR)

**GPIO_SWPORT_DR_L** 0x0000
**GPIO_SWPORT_DR_H** 0x0004
同上方向寄存器，我们要找的是 **低位寄存器 (High)**：

- **数据寄存器 (DR)**: `GPIO_SWPORT_DR_L`，偏移量是 **`0x0000`**
  **数据寄存器的地址（DR）**：基地址+偏移地址=`0xFDD60000` + `0x0000` = **`0xFDD60000`**
  使用`io -r -4 0xfdd60000`命令查看该寄存器的默认值,该寄存器默认值为 **`0x0000c040`**，接下来控制 `GPIO0_B7` 高低电平需要在此寄存器默认值的基础上进行控制。

找到具体的 数据寄存器（GPIO_SWPORT_DR_L）的描述

- bit16-bit31 对应 GPIOA0~GPIOA7、GPIOB0~GPIOB7 引脚数据位的写使能。
- bit0-bit15 对应 GPIOA0~GPIOA7, GPIOB0~GPIOB7 数据位
  如果要设置 GPIO0_B7 输出电平，需要先将 GPIO0_B7 引脚对应的写使能为 bit31 写 1 使能，然后对 bit15 写入 0 即将 GPIO0_B7 引脚输出低电平，写入 1 则输出高电平。

如果让 GPIO0_B7 输出**高**电平则需要向数据寄存器写入 **`0x8000c040`**
如果输出**低**电平，需要设置第 15 位为 0 ，第 31 位为 1，那么向数据寄存器中写入**`0x80004040`**

### 📝 总结：我们找到的藏宝图

针对 **RK3568 GPIO0_B7**：

| 寄存器名称                   | 物理地址 (Physical Address) | 作用                                              |
| :--------------------------- | :-------------------------- | :------------------------------------------------ |
| `PMU_GRF`                    | `0xFDC20000`                | 复用寄存器基地址                                  |
| `PMU_GRF_GPIO0B_IOMUX_H`     | `0x000C`                    | 复用寄存器的偏移量，B0-B3 是低位，B4-B7 是高位    |
| **`PMU_GRF_GPIO0C_IOMUX_H`** | **`0xFDC2000c`**            | **设为 GPIO 模式**                                |
| `GPIO0`                      | `0xFDD60000`                | 基于此设置 DR 和 DDR 寄存器，只需要加上偏移量即可 |
| `GPIO_SWPORT_DDR_L`          | `0x0008`                    | 方向寄存器的偏移地址                              |
| **`GPIO_SWPORT_DDR_L`**      | **`0xFDD60008`**            | **设为输出模式**                                  |
| `GPIO_SWPORT_DR_L`           | `0x0000`                    | 数据寄存器的偏移地址                              |
| **`GPIO0_SWPORT_DR_L`**      | **`0xFDD60000`**            | **写 1 亮，写 0 灭**                              |

**除了这些，还有一些规则，具体在代码里面体现，主要是使用**writel**函数，在计算完方向/数据寄存器地址之后控制方向与数据**

---

### 💻 代码实战：如何在驱动里用这些地址？

在 Linux 驱动中，我们**不能直接操作物理地址**（那是会报错的）。我们必须使用 `ioremap` 把物理地址映射成**虚拟地址**。
主要设计函数 **ioremap** 、**writel**、 **iounmap**。
在 init 函数中主要进行

1. 映射地址；
2. 设置复用寄存器（目的是设置为 GPIO 模式）；
3. 设置方向寄存器（目的是设置为 Output）；
4. 初始化状态（关灯）

   而在 write 函数中，主要是通过判断传入的指令（开灯？关灯？），进而对数据寄存器进行不同的操作

#### 驱动代码片段 (led_drv.c)

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/types.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/slab.h>
#include <linux/io.h>

// 定义设备名
#define CLA_NAME "led_class"
#define DEV_NAME "led_device"
#define DEV_NUM "led_dev"
// 基地址
#define RK3568_PMU_GRF_BASE 0xFDC20000
#define RK3568_GPIO0_BASE 0xFDD60000

// 偏移量
// 复用寄存器：PMU_GRF_GPIO0B_IOMUX_H (控制 GPIO0_B0 ~ B7)
#define PMU_GRF_GPIO0B_IOMUX_H_OFFSET 0x000C
// 数据寄存器：GPIO0_B7 属于低16位 (Bit 15)，所以用 DR_L
#define GPIO_SWPORT_DR_L_OFFSET 0x0000
// 方向寄存器：GPIO0_B7 属于低16位 (Bit 15)，所以用 DDR_L
#define GPIO_SWPORT_DDR_L_OFFSET 0x0008

// 规则
// 复用寄存器 需要把高16位对应的 bits [30:28] 置 1 来使能写操作  值: (0x7 << 28) | (0x0 << 12) = 0x7000 0000
#define PMU_REGULAR 0x70000000
// 方向寄存器 Bit 15 + 16 = Bit 31 是写使能位  值: (1 << 31) | (1 << 15) = 0x80008000
#define DDR_REGULAR 0x80008000

// 开灯：Bit 31 置1(使能), Bit 15 置1(高电平) -> 0x8000 8000
#define LED_OPEN_VAL 0x80008000
// 关灯：Bit 31 置1(使能), Bit 15 置0(低电平) -> 0x8000 0000
#define LED_CLOSE_VAL 0x80000000

// LED控制
#define LED_CLOSE 0x80004040
#define LED_OPEN 0x8000c040

struct device_test
{
    /* data */
    dev_t dev; // 设备号
    int major;
    int minor;
    struct cdev my_cdev; //// 结构体和设备号关联起来
    struct class *my_class;
    struct device *my_device;
    char buff[128];
    void __iomem *vir_gpio0;   // 映射 GPIO0 基地址
    void __iomem *vir_pmu_grf; // 用于复用控制
};
struct device_test dev1;
static int myopen(struct inode *inode, struct file *file)
{
    /*在使用时一般将设备结构体指向 file 结构体中 private_data 成员，此时设备结构体就变成了文件私有数据
    对 file 结构体中 private_data 成员赋值一般是在文件操作集中的 open() 函数中赋值*/
    file->private_data = &dev1;
    printk(KERN_INFO "-------------迅为RK3568: 设备被打开了\n");
    return 0;
}

static ssize_t mywrite(struct file *file, const char __user *buf, size_t size, loff_t *ppos)
{

    /*通 file 结构体可以将私有数据一路从 open()函数带到 read(), write()函数。接着就是可以在 read()、write()函数中使用文件私有数据。*/
    struct device_test *test_dev = (struct device_test *)file->private_data;
    void __iomem *dr_addr = test_dev->vir_gpio0 + GPIO_SWPORT_DR_L_OFFSET;

    int ret = copy_from_user(test_dev->buff, buf, size); // 应用层将buff传入内核
    if (ret != 0)
    {
        printk(KERN_ERR "******copy_from_user error!\n");
        return -ret;
    }
    // 计算数据寄存器虚拟地址

    if (dev1.buff[0] == 0) // 如果传入的是0  则关灯
    {
        writel(LED_CLOSE_VAL, dr_addr);
        // 对指针 “解引用”，访问指针指向的内存单元（即 GPIO 寄存器本身），可读写其值
        printk("LED Driver: OFF (Write 0x%X)\n", LED_CLOSE_VAL);
    }
    if (dev1.buff[0] == 1) // 如果传入的是1  则开灯
    {
        writel(LED_OPEN_VAL, dr_addr);
        printk("LED Driver: ON (Write 0x%X)\n", LED_OPEN_VAL);
    }
    printk("--------------写入数据到私有内存：%s\n", test_dev->buff);
    return size;
}
static ssize_t myread(struct file *file, char __user *buf, size_t size, loff_t *ppos)
{
    struct device_test *test_device = file->private_data;
    int ret = copy_to_user(buf, test_device->buff, size);
    if (ret)
    {
        printk("-----------------read error\n");
        return -ret;
    }
    printk("-------------从私有内存读取数据: %s\n", test_device->buff);
    return size;
}
// --- 4. 关闭设备 (退房) ---
static int myrealse(struct inode *inode, struct file *file)
{
    printk(KERN_INFO "-------------迅为RK3568: 设备被释放了\n");
    return 0;
}
static struct file_operations my_ops = {
    /* 顾客来了能做什么,定义商店的经营方法 */
    .owner = THIS_MODULE,
    .open = myopen,
    .read = myread,
    .write = mywrite,
    .release = myrealse,
};

static int __init module_cdev_init(void) // 驱动入口函数
{
    int ret;

    // --- 1.申请设备号 （动态申请）---
    ret = alloc_chrdev_region(&dev1.dev, 0, 1, DEV_NUM);
    if (ret < 0)
    {
        printk("-----------------alloc_chrdev_region error\n"); // 申请设备号失败
        return ret;
    }
    printk("--------------alloc_chrdev_region succeed,major %d，minor %d\n", MAJOR(dev1.dev), MINOR(dev1.dev)); // 营业执照申请成功！
    // --- 2. 注册字符设备 ---
    cdev_init(&dev1.my_cdev, &my_ops);
    // 将 cdev 添加到内核
    ret = cdev_add(&dev1.my_cdev, dev1.dev, 1); // 为什么是1
    if (ret < 0)
    {
        printk("------------------cdev_add error\n"); // 注册经营范围失败
        goto free_dev;
    }
    printk("--------------cdev_add secceed\n"); // 经营范围已在工商局注册
    // --- 3. 创建设备节点 ---
    // 创建一条“商业街”
    dev1.my_class = class_create(THIS_MODULE, CLA_NAME);
    if (IS_ERR(dev1.my_class))
    {
        // 错误处理...
        printk(KERN_ERR "---------------class_create error\n ");
        ret = PTR_ERR(dev1.my_class); // 把指针错误转换成错误码(int)
        // 这里失败了，要撤销第二关(cdev)和第一关(设备号)
        goto destroy_cdev;
    }
    printk(KERN_INFO "---------------private_class : 店铺已在 /sys/class/ 挂牌营业！\n ");
    // 在商业街上创建我们的“店面”，并取名 "mydevice111"
    // 参数: 类, 父设备(NULL), 设备号, 私有数据(NULL), 设备名
    dev1.my_device = device_create(dev1.my_class, NULL, dev1.dev, NULL, DEV_NAME);
    if (IS_ERR(dev1.my_device))
    {
        // 错误处理...
        printk(KERN_ERR "---------------device_create error\n ");
        ret = PTR_ERR(dev1.my_device);
        // 这里失败了，要撤销第三关(类)、第二关(cdev)、第一关(设备号)
        goto destroy_class;
    }
    printk(KERN_INFO "---------------MyDevice : 店铺已在 / dev / mydevice111 挂牌营业！\n ");
    /*添加*/
    // --- 硬件初始化 ---
    // A. 映射地址 (映射 4KB 足够覆盖寄存器)
    dev1.vir_gpio0 = ioremap(RK3568_GPIO0_BASE, 4096);
    dev1.vir_pmu_grf = ioremap(RK3568_PMU_GRF_BASE, 4096);
    if (!dev1.vir_pmu_grf || !dev1.vir_gpio0)
    {
        printk("ioremap error!\n");
        return -ENOMEM;
        goto destroy_ioremap;
    }
    // B. 设置复用 (IOMUX) -> 设为 GPIO 模式
    // 寄存器: PMU_GRF_GPIO0B_IOMUX_H (Offset 0x0C)
    // GPIO0_B7 对应 bits [14:12]，模式 000 是 GPIO
    // 写规则: 需要把高16位对应的 bits [30:28] 置 1 来使能写操作
    // 值: (0x7 << 28) | (0x0 << 12) = 0x7000 0000
    writel(PMU_REGULAR, dev1.vir_pmu_grf + PMU_GRF_GPIO0B_IOMUX_H_OFFSET);
    printk("LED Driver: IOMUX Set to GPIO\n");
    // C. 设置方向 (DDR) -> 设为 Output
    // 寄存器: GPIO_SWPORT_DDR_L (Offset 0x08)
    // GPIO0_B7 对应 Bit 15
    // 写规则: Bit 15 + 16 = Bit 31 是写使能位
    // 值: (1 << 31) | (1 << 15) = 0x80008000
    writel(DDR_REGULAR, dev1.vir_gpio0 + GPIO_SWPORT_DDR_L_OFFSET);
    printk("LED Driver: Direction Set to Output\n");
    // D：默认关灯  操作数据寄存器
    writel(LED_CLOSE_VAL, dev1.vir_gpio0 + GPIO_SWPORT_DR_L_OFFSET);
    printk("LED Driver: Init Success!\n");

    return 0;
destroy_ioremap:
    if (dev1.vir_pmu_grf) // 成功的才还  如果dev1.vir_pmu_grf成功，但是vir_gpio0失败，要销毁成功的
        iounmap(dev1.vir_pmu_grf);
    if (dev1.vir_gpio0)
        iounmap(dev1.vir_gpio0);
    device_destroy(dev1.my_class, dev1.dev);
destroy_class:
    class_destroy(dev1.my_class);

destroy_cdev:
    cdev_del(&dev1.my_cdev);
free_dev:
    unregister_chrdev_region(dev1.dev, 1);
    return ret;
}

// 驱动出口函数
static void __exit module_cdev_exit(void)
{
    // 取消映射
    iounmap(dev1.vir_pmu_grf);
    iounmap(dev1.vir_gpio0);
    // 3.销毁设备节点
    device_destroy(dev1.my_class, dev1.dev);
    class_destroy(dev1.my_class);
    // 2.注销字符设备
    cdev_del(&dev1.my_cdev);
    // 1.归还设备号
    unregister_chrdev_region(dev1.dev, 1);
    printk(KERN_INFO "----------------MyDevice: 店铺已摘牌，打烊了！\n"); // MyDevice: 店铺已摘牌，打烊了！
}

module_init(module_cdev_init); // 注册入口函数
module_exit(module_cdev_exit); // 注册出口函数
MODULE_LICENSE("GPL v2");      // 同意GPL开源协议
MODULE_AUTHOR("Can");          // 作者信息
                               // 你的驱动已经可以在加载后自动创建 /dev/mydevice 文件了！


```

---

### 🔥 重点提示（避坑指南）

1.  **写使能位（Write Mask）**：这是 Rockchip 芯片最著名的特点！
    大多数寄存器（尤其是 GRF 和 GPIO），高 16 位是“写保护锁”。

    - 如果你想修改 bit 0，你必须同时把 bit 16 写为 1。
    - 如果你只写 bit 0 = 1，但 bit 16 = 0，硬件会忽略你的操作。
    - **公式**：`val = (1 << (bit + 16)) | (value << bit);`

2.  **iomap 失败**：
    如果 `ioremap` 失败，通常是因为这段地址已经被内核的其他驱动（比如原生的 GPIO 驱动）申请走了。在实验阶段，可以在设备树里把对应的 GPIO 节点 `status = "disabled";` 掉，防止冲突。

3.  **H 和 L 的区分**：
    RK3568 一个 GPIO Port 有 32 个引脚（A, B, C, D 各 8 个）。
    - `_L` 寄存器通常管 A 和 B (0-15)。
    - `_H` 寄存器通常管 C 和 D (16-31)。
    - 一定要数清楚你的引脚是第几个。

希望这个讲解能帮你顺利找到寄存器，点亮那颗 LED！
