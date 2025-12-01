---
title: "Linux驱动开发 平台总线&设备树" # <--- 修改这一行
date: "2025-11-25T11:57:34+08:00"
draft: false
tags: ["Linux", "驱动开发"]
location: ""
---

本篇融合了**【北京迅为】itop-3568 开发板驱动开发指南（重制版）1.8.pdf** 官方文档中平台总线和设备树两章内容，并且带有完整且详尽实例分析，可以先看说明部分理清文章主要所讲内容与侧重点

## 关于平台总线

### 一、 为什么需要“平台总线”？

#### 1. 没有平台总线时的痛苦

假设你写了一个 LED 驱动。

- **代码里写死**：`寄存器地址 = 0xFE001000`。
- **问题**：
  - RK3568 的地址是这个，换成 RK3588 地址变了，你得改 C 代码重新编译。
  - 板子上接在 GPIO1，换到 GPIO2，你得改 C 代码重新编译。
- **结论**：驱动代码和硬件参数（地址、中断号）**强耦合**。移植性极差。

#### 2. 平台总线的思想

Linux 引入了 **“总线 - 设备 - 驱动”** 模型。

- **驱动（Driver）**：只负责逻辑（怎么亮、怎么灭），**不包含**具体的硬件地址。
- **设备（Device）**：只包含硬件资源（地址是 0xFE...，中断是 5 号），**不包含**控制逻辑。
- **总线（Bus）**：红娘。负责把“驱动”和“设备”撮合在一起。

#### 理解

想象一下，现在你有 5 款不同的板子，用的都是同一个 LED 驱动逻辑，但 LED 接的引脚不同。

- **以前**：你需要写 5 个 `.c` 文件，每个文件里引脚宏定义不同。
- **现在（平台总线）**：
  - 你只需要写 **1 个** 通用的 `.c` 驱动文件（不包含任何具体地址）。
  - 你需要写 **5 个** 不同的 `.dts` 文件（只包含地址和引脚信息）。
  - **结果**：驱动代码实现了“一次编写，到处运行”。这就是 Linux 所谓“机制与策略分离”的体现。

#### 总结

没有平台总线概念的时候，驱动和硬件紧密连接，一个硬件参数变了的话，就得改 C 代码重新编译，麻烦；
平台总线：设立了驱动和设备两套代码，驱动负责逻辑（一般般不改变），设备负责硬件参数。以后硬件资源变换了之后，就只需要修改设备代码（简单），不需要改动驱动。平台总线就是把两个东西联合起来。

---

### 二、 平台总线的“铁三角”

以上所说的驱动，设备，平台总线，对应下面三个结构体。要理解平台总线，必须看懂这三个结构体：

#### 1. 平台设备 (`struct platform_device`) —— “我是谁，我在哪”

- **代表**：硬件设备。
- **来源**：**通常不需要手写 C 代码定义它**，而是由 **设备树 (Device Tree)** 解析后自动生成的，就是由写的 dts 自动生成`platform_device` 。
- **核心内容**：
  - `name`: 设备名字。
  - `resource`: 资源（寄存器物理地址、中断号）。

#### 2. 平台驱动 (`struct platform_driver`) —— “我会干什么”

- **代表**：软件驱动程序。
- **来源**：**这是你需要手写的 C 代码**。
- **核心内容**：
  - `probe()`: **探测函数**。当匹配成功时执行（相当于驱动的入口）。
  - `remove()`: 移除函数。
  - `driver.of_match_table`: **匹配表**（用来和设备树里的 `compatible` 对暗号）。

#### 3. 平台总线 (`platform_bus_type`) —— “金牌中介”

- **代表**：内核里的匹配机制（已经写好了，不用你写）。
- **工作机制**：
  - 每当有一个新驱动注册，总线就会遍历所有设备，问：“你俩匹配吗？”
  - 每当有一个新设备注册，总线就会遍历所有驱动，问：“你俩匹配吗？”

---

### 三、 它是怎么“自动匹配”的？

系统里有了 platform_device。你加载了 platform_driver。内核发现两者的 .compatible 名字一样（比如都叫 "rk3568,led"）。
“相亲成功” -> 调用 probe()。

**匹配凭证**：主要看 **`compatible`** 属性（兼容性字符串）。

#### 流程讲解

1.  **左边（设备树/设备）**：
    你在 DTS 里写了一个节点：

    ```dts
    my_led {
        compatible = "rk3568,my-led";  // <--- 暗号
        reg = <0xFE001000 0x4>;
    };
    ```

    内核启动时，自动转换成了一个 `platform_device`，挂在总线左边的链表上。

2.  **右边（驱动代码）**：
    你写了一个 `platform_driver`，注册到内核：

    ```c
    static const struct of_device_id my_ids[] = {
        { .compatible = "rk3568,my-led" }, // <--- 暗号
        { }
    };
    /* ... 注册驱动 ... */
    ```

3.  **中间（总线匹配）**：

    - 平台总线发现新驱动来了，立刻启动 `platform_match` 函数。
    - 它拿着驱动里的 `"rk3568,my-led"` 去跟设备链表里的每一个设备比对。
    - **比对成功！**

4.  **触发 Probe**：
    - 总线会自动调用你驱动里的 `probe(struct platform_device *pdev)` 函数。
    - 并将那个匹配上的设备结构体指针 `pdev` 传给你。
    - 你在 `probe` 里，从 `pdev` 里取出资源（0xFE001000），申请 GPIO，点亮 LED。

---

### 五、 代码模板

这是你接下来写所有驱动的基础骨架，请务必熟悉：

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>

// 3. 真正的初始化逻辑在这里（当且仅当匹配成功时执行）
static int my_probe(struct platform_device *pdev)
{
    printk("【平台总线】匹配成功！设备树节点是: %pOF\n", pdev->dev.of_node);

    // 在这里获取资源、注册字符设备、生成类和节点...
    // 之前学的 register_chrdev 写在这里面

    return 0;
}

// 4. 清理逻辑
static int my_remove(struct platform_device *pdev)
{
    printk("【平台总线】驱动移除\n");
    // 注销字符设备...
    return 0;
}

// 2. 匹配表（身份证验证）
static const struct of_device_id my_match_table[] = {
    {.compatible = "my_rk3568"},
    {}
};
MODULE_DEVICE_TABLE(of, my_match_table);

// 1. 定义驱动结构体
static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my_driver_v1", // 在 /sys/bus/platform/drivers/ 下的名字
        .of_match_table = my_match_table, // 挂载匹配表
    },
};

// 0. 向内核注册这个平台驱动
module_platform_driver(my_driver);
MODULE_LICENSE("GPL");
```

### 总结

1.  **平台总线**是 Linux 用来管理 SoC 内部外设的虚拟总线。
2.  它的核心目的是**把“驱动代码”和“硬件地址”拆开**。
3.  **设备树（DTS）** 负责生成 `platform_device`（提供硬件信息）。
4.  **你写的 C 代码** 负责注册 `platform_driver`（提供软件逻辑）。
5.  **Probe 函数** 是它们“相亲成功”后的约会地点，所有的初始化代码都要从 `module_init` 搬到 `probe` 里来。

## 设备树细节

### 第一阶段：什么是设备树？怎么写它？

#### 1. 通俗理解

在没有设备树之前，驱动代码里到处都是硬件信息，比如 `Base Address = 0xFE001000`, `IRQ = 5`。如果板子改了个电阻，引脚变了，你就得去改 C 代码，重新编译内核。

**设备树（DTS）** 就像是一个**“硬件菜单”**或**“配置文件”**。

- **DTS 文件**（`.dts`）：给程序员看的文本文件，描述板子上有哪些硬件（CPU、内存、LED、I2C 控制器）。
- **DTB 文件**（`.dtb`）：给内核看的二进制文件。DTS 编译后生成 DTB。
- **驱动代码**（`.c`）：厨师。厨师不记菜单，而是读菜单做菜。

**核心思想**：把硬件信息（DTS）和驱动逻辑（C 代码）彻底分开。

#### 2. 基本语法（代码实战）

设备树是树状结构，只有一个根节点 `/`。

**代码任务**：
我们需要在 RK3568 的设备树文件中，添加一个我们自己的虚拟节点。打开 RK3568 的设备树文件（在内核源码 `/Linux/linux_sdk/kernel/arch/arm64/boot/dts/rockchip/rk3568-evb1-ddr4-v10-linux.dts` ）在根节点 `/ { ... };` 内部，添加以下内容：

```dts
rk3568-evb1-ddr4-v10-linux.dts
/{
    my_test{
        #address-cells = <1>;
        #size-cells = <1>;
        compatible = "simple-bus";
        myLed{
            compatible = "my_rk3568";
            reg = <0xFDD60000 0x00000004>;
        };
    };
};
```

- **`compatible`**：这是最重要的属性！它是**驱动程序和设备树节点“相亲”的凭证**。驱动代码里也会写一个一模一样的字符串，匹配上了，驱动的 `probe` 函数才会执行。

#### 设备树展开流程

dts -> dtb -> device_node -> platform_device (device_node 扁平化结构有点像链表)
dts->dtb 的命令：

```C
/home/topeet/Linux/linux_sdk/kernel/scripts/dtc/dtc -I dts -O dtb -o test.dtb test.dts
```

反编译 dtb -> dts

```
/home/topeet/Linux/linux_sdk/kernel/scripts/dtc/dtc -I dtb -O dts -o 1.dts test.dtb
```

(DTC 源代码和相关工具放在：/home/topeet/Linux/linux_sdk/kernel/scripts/dtc/dtc)

在内核启动时会执行 `of_platform_default_populate_init()`函数，该函数就是将设备树节点（device_node）转换成平台设备（platform_device）的入口函数。
该函数会遍历设备树中的设备节点，并为每个符合条件的设备节点创建一个对应的 `platform_device` 结构，然后将其注册到内核中，使得设备驱动程序能够识别和操作这些设备。
`platform_device` 的节点可以在**`/sys/bus/platform/devices`** 下查看

#### device_node 转换为 platform_device 的条件/规则

规则 1：根节点下有 compatible 属性的子节点。
规则 2：节点中 compatible 属性值是 `simple-bus`、`simple-mfd` 和` isa` 其中之一，并且该节点下的子节点有 `compatible` 属性。该节点下的子节点会被转换成 `platform_device`。
规则 3：节点中`compatible`属性有`arm`或 `primecell`，则对应的节点不会被转换成 `platform_device`。

### 第二阶段：驱动如何“自动”加载？（设备树下 platform_device 和 platform_driver 匹配）

上面两节，书写了.dts 文件和 driver 驱动，接下来进行实操

1.  在目录`/Linux/linux_sdk`下编译内核（./build.sh kernel）并烧写带 `my_test` 的内核（生成的`boot.img`在目录`/home/topeet/Linux/linux_sdk/kernel/boot.img`,烧写时记得接 usb 线）。
2.  编译加载(insmod)这个 `.ko` 模块。
3.  使用 `dmesg` 查看，如果你看到“**匹配成功！发现设备树节点！**”，说明你已经打通了设备树到驱动的任督二脉！

---

## 在驱动里读取设备树数据（OF 操作函数）

### 1. 理论讲解

目标：在驱动里读取 DTS 里写的参数。如果不掌握这些函数，你的驱动虽然能匹配上（进入 `probe`），但就像**“进了餐厅却看不懂菜单”**，不知道具体的配置参数（比如阈值是多少、ID 是多少、名字叫什么）。这些以 **`of_`** 开头的函数（**O**pen **F**irmware），就是用来翻译菜单的工具。
原理：设备树节点中可以包含属性（如 `reg`, `my-value`）。驱动匹配成功后，内核会将节点信息保存在 `pdev->dev.of_node` 中。我们需要使用 `of_property_read_u32` 等函数提取这些数据。

---

### 2. 核心概念：节点指针 (`device_node`)

在 `probe` 函数里，你的第一步永远是**先拿到“菜单”**。这个“菜单”就是 `struct device_node` 指针。

```c
static int my_probe(struct platform_device *pdev)
{
    // 1. 获取当前设备对应的设备树节点指针
    struct device_node *np = pdev->dev.of_node;

    if (!np) {
        printk("没有设备树节点！\n");
        return -1;
    }
    // ... 后面所有的 of_ 函数都要用到这个 np
}
```

---

### 3. 最常用的 3 招（覆盖 90% 场景）

假设我们在设备树里写了这样一个节点：

```c
/* DTS 文件 */
my_sensor {
    compatible = "rk3568,sensor";
    /* 1. 整数属性 */
    sensor-id = <0xA5>;
    max-speed = <1000>;

    /* 2. 字符串属性 */
    chip-name = "ICm-20608";

    /* 3. 数组属性 (很少用，但得知道) */
    default-levels = <0 10 20 30>;
};
```

我们看看如何在驱动代码里把它们读出来。

#### 1. 读取整数 (`of_property_read_u32`)

这是最常用的，用来读 ID、配置参数、波特率等。

**函数原型**：
`int of_property_read_u32(const struct device_node *np, const char *propname, u32 *out_value)`

**代码实战**：

```c
u32 id = 0;
u32 speed = 0;
int ret;

/* 读 sensor-id */
ret = of_property_read_u32(np, "sensor-id", &id);
if (ret == 0) {
    // 成功！注意 DTS 里是 0xA5，读出来就是十进制 165
    printk("读到 ID: %#x\n", id);
} else {
    printk("读取 sensor-id 失败\n");
}

/* 读 max-speed */
of_property_read_u32(np, "max-speed", &speed); // 简写，假设一定成功
```

#### 2. 读取字符串 (`of_property_read_string`)

用来读取名字、模式配置等。

**函数原型**：
`int of_property_read_string(const struct device_node *np, const char *propname, const char **out_string)`

**代码实战**：

```c
const char *name_str = NULL; // 这是一个指针，指向内核内存

ret = of_property_read_string(np, "chip-name", &name_str);
if (ret == 0) {
    printk("芯片名字是: %s\n", name_str);
}
```

#### 3. 读取数组 (`of_property_read_u32_array`)

用来读取一组数据，比如一些校准参数。

**代码实战**：

```c
u32 levels[4]; // 准备一个数组来接数据

/* 参数：节点，属性名，数组指针，读取数量 */
ret = of_property_read_u32_array(np, "default-levels", levels, 4);
if (ret == 0) {
    printk("Level 1: %d\n", levels[1]); // 输出 10
}
```

---

### 4. 特殊属性的读取（不要硬用 OF 函数）

虽然 `reg`（地址）、`interrupts`（中断）、`gpios`（引脚）本质上也是属性，**但内核提供了更高级的封装**，我们通常**不**直接用上面的 `of_property_read_xxx` 来读它们。

#### 1. 读取 GPIO（关键！）

不要去读什么 `xxx-gpios = <...>` 里的数字。
**使用 GPIO 子系统接口**，内核会自动处理设备树解析。

- **DTS**: `led-gpios = <&gpio0 15 0>;`
- **Code**:

  ```c
  #include <linux/gpio/consumer.h>

  struct gpio_desc *my_gpio;
  /* 内核会自动去DTS找 "led" + "-gpios" 这个属性 */
  my_gpio = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
  ```

#### 2. 读取资源（reg 和 interrupts）

- **DTS**: `reg = <0xFE000000 0x100>;`
- **Code**:

  ```c
  struct resource *res;
  /* IORESOURCE_MEM 表示获取 reg 属性 */
  res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
  // 拿到 resource 后，通常接着用 devm_ioremap_resource 映射内存
  ```

- **DTS**: `interrupts = <...>;`
- **Code**:
  ```c
  int irq;
  irq = platform_get_irq(pdev, 0); // 获取第0个中断号
  ```

---

### 5. 避坑指南（新手必看）

1.  **返回值检查**：
    所有 OF 函数，**返回 `0` 表示成功**，返回负数（如 `-EINVAL`）表示失败。不要以为返回 1 是成功！

2.  **u8 / u16 / u32 的区别**：
    设备树里的数字默认都是 **32 位（u32）** 的。除非你显式使用了 `/bits/ 8 <...>` 语法，否则一律用 `of_property_read_u32`。不要为了省内存用 `read_u8`，读不出来的。

3.  **const 指针**：
    `of_property_read_string` 返回的字符串指针指向的是设备树在内存中的原始数据，是 **Read Only** 的。千万不要试图去 `strcpy` 修改它，否则系统会崩。

---

### 6. 总结表

| 你想读什么？     | DTS 写法示例          | C 代码函数 (推荐)                |
| :--------------- | :-------------------- | :------------------------------- |
| **自定义整数**   | `my-val = <100>;`     | `of_property_read_u32`           |
| **自定义字符串** | `my-name = "abc";`    | `of_property_read_string`        |
| **GPIO 引脚**    | `xxx-gpios = <...>;`  | `devm_gpiod_get` (不要直接用 OF) |
| **寄存器地址**   | `reg = <...>;`        | `platform_get_resource`          |
| **中断号**       | `interrupts = <...>;` | `platform_get_irq`               |

掌握了这张表，你在驱动里“读菜单”的能力就完全够用了！

**意义**：以后你想修改参数，只需要改 DTS（不用重新编译驱动代码），这就是**配置与代码分离**。

---

## 控制硬件（GPIO 与 Pinctrl）

#### 1. 通俗理解

这是最复杂的，也是最常用的。

- **Pinctrl**：管脚控制器。RK3568 引脚多，你要在 DTS 里告诉内核：“我要把 GPIO0_A0 这个脚配置成 GPIO 模式，不要用作 PWM”。
- **GPIO 子系统**：配置好模式后，在驱动里控制高低电平。

#### 2. 代码实战（点灯）

**Step 1: 修改设备树（DTS）**
你需要查看原理图找到 LED 对应的 GPIO 号 GPIO0_B7, 在 DTS 中引用 Pinctrl 配置（迅为的 DTS 通常已经预定义好了 Pinctrl 节点，引用即可）。

```c dts /Linux/linux_sdk/kernel/arch/arm64/boot/dts/rockchip/rk3568-evb1-ddr4-v10-linux.dts
/ {
    my_led{
        compatible = "rk3568,my_led";
        status = "okay";
        led-gpios = <&gpio0 RK_PB7 GPIO_ACTIVE_HIGH>;
    };
};
```

**注意** 编写完所有代码之后发现 led 没有反应
排查原因：查看在 dts 中写的 gpio 引脚是否被占用？
`cat /sys/kernel/debug/gpio`
发现：
`gpio-15  (                    |work                ) out hi`
gpio-15：这就是我们要操作的 GPIO0_B7 (032 + 18 + 7 = 15)。|work：这表示这个引脚当前正在被一个名叫 "work" 的驱动占用着。out hi：它已经被配置为输出高电平了。也就是说：内核自带的 leds-gpio 驱动（或者类似的系统指示灯驱动）已经根据原厂的设备树，先把这个脚给占用了。
寻找&gpio0 RK_PB7，在：/home/topeet/Linux/linux_sdk/kernel/scripts/dtc/include-prefixes/arm64/rockchip/rk3568-evb.dtsi 中发现 work 驱动

```c
   leds: leds {
        compatible = "gpio-leds";
        work_led: work {
            status = "disabled";              // --->添加这一行,使其失效
            lable = "user1";
            gpios = <&gpio0 RK_PB7 GPIO_ACTIVE_HIGH>;
            linux,default-trigger = "heartbeat";
            default-state = "on";
        };
   }
```

**Step 2: 编写驱动（使用新版 GPIO API）**

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/of.h>
#include <linux/platform_device.h>
#include <linux/gpio/consumer.h>

/* 定义一个指针，指向我们要操作的那个 GPIO */
struct gpio_desc *my_led_gpio;
/*
 * 1. Probe 函数
 * 只有当 DTS 的 compatible 和 驱动的 compatible 一样时，
 * 这个函数才会被内核调用！
 */
static int my_prob(struct platform_device *pdev)
{
    printk(KERN_INFO "==========匹配成功！==========\n");
    printk(KERN_INFO "我是驱动，我找到了我的设备！\n");
    if (pdev->dev.of_node)
    {
        printk(KERN_INFO "设备树节点名: %s\n", pdev->dev.of_node->name);
    }
    /*
     * 【核心步骤：获取 GPIO】
     * 参数1: 设备指针
     * 参数2: 属性名前缀 "led"。
     *        内核会自动在 DTS 里找 "led" + "-gpios" = "led-gpios" 属性。
     * 参数3: 初始化标志。GPIOD_OUT_LOW 表示获取后立即设置为输出模式，且输出低电平（灭）。
     */
    my_led_gpio = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
    if (IS_ERR(my_led_gpio))
    {
        printk(KERN_ERR "【LED驱动】获取 GPIO 失败！\n");
        return PTR_ERR(my_led_gpio);
    }

    /* 点亮 LED！(设置为高电平 1) */
    gpiod_set_value(my_led_gpio, 1);
    printk(KERN_INFO "【LED驱动】灯已点亮！(GPIO0_B7 set High)\n");

    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    gpiod_set_value(my_led_gpio, 0);
    /* 注意：使用了 devm_gpiod_get，内核会自动释放 GPIO，不需要手动 free */
    printk(KERN_INFO "============= 驱动卸载 ============\n");
    return 0;
}
/*
 * 2. 匹配表
 * 这是整个匹配过程的核心！
 */
static const struct of_device_id my_match_table[] = {
    {.compatible = "rk3568,my_led"},
    {}};
/* 将匹配表注册到模块信息中 */
MODULE_DEVICE_TABLE(of, my_match_table);
/*
 * 3. 驱动结构体
 */
static struct platform_driver my_driver = {
    .probe = my_prob,
    .remove = my_remove,
    .driver = {
        .name = "my_led_driver", /* 驱动名字，也就是 /sys/bus/platform/drivers/ 下的名字 */
        .owner = THIS_MODULE,
        .of_match_table = my_match_table, /* 关联匹配表 */
    },
};
/* 4. 注册平台驱动 */
module_platform_driver(my_driver);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Can"); // 作者信息
```

#### 实操

1.  在目录`/Linux/linux_sdk`下编译内核（./build.sh kernel）并烧写带 `my_led` 的内核（生成的`boot.img`在目录`/home/topeet/Linux/linux_sdk/kernel/boot.img`,烧写时记得接 usb 线）。
2.  编译加载(insmod)这个 `.ko` 模块。
3.  使用 `dmesg` 查看，是否与驱动程序中匹配

---

## 说明

1. 关于官方文档上在**平台总线章节**里面所说的**注册 platform 设备实验**，个人认为完全没必要，因为只要会写 `dts`，完全没必要自己写 `platform_device` 实验，AI 说：几乎不用 (除非做纯软件实验)。毕竟**Linux is a fucking pain in the ass！**，笑死...
2. DTS 的编译原理：不用管 `dtc` 编译器怎么把文本变成二进制的，会用命令 `make dtbs` （`/home/topeet/Linux/linux_sdk/kernel/scripts/dtc/dtc -I dts -O dtb -o test.dtb test.dts`）就行。
3. DTS 的展开过程：不用管内核启动时怎么把二进制变成结构体的（Unflatten），知道 `probe` 的时候已经变好了就行。
4. 复杂的总线定义：不用管 CPU 节点、总线控制器节点是怎么写的，那些都是原厂写好的，你只需要在它们下面挂设备。
5. DTS 语法细节：比如 `#address-cells`, `#size-cells, ranges`。这些通常只有在写总线桥接或者内存映射时才用，普通的设备驱动开发几乎不用动。

## 学习建议与避坑

1.  **不要死磕 DTS 全部语法**：DTS 里有很多复杂的属性（如 `ranges`, `clock-names`），初学看不懂很正常。你只需要关注 `compatible`（匹配用）、`reg`（地址用）、自定义属性、以及 `gpios`（引脚用）。
2.  **编译是难点**：修改 DTS 后，需要重新编译设备树（`make dtbs`），然后将新的 `.img` 或 `.dtb` 烧录到板子上。如果烧录后板子起不来，通常是 DTS 语法写错了。
3.  **视频观看顺序**：
    - 先看 **"初识"** 和 **"语法"**。
    - 直接跳到 **"platform 匹配实验"**，把驱动框架搭起来。
    - 然后看 **"of 操作函数"**，学会读数据。
    - 最后看 **"GPIO"** 和 **"pinctrl"**，做点灯实验。
    - _至于“设备树展开流程”、“DTB 格式”，那是内核原理，可以等以后有空当故事听，不影响写代码。_

按照这个“四步走”路线，每一节都有代码反馈，你会学得很有成就感！加油！

**-------------设备树篇完结--------------**
