---
title: "Linux驱动开发 Pinctrl子系统" # <--- 修改这一行
date: "2025-12-05T19:24:02+08:00"
draft: false
tags: ["Linux", "驱动开发"]
location: ""
---

- **GPIO 子系统**：管**电平**（高低/输入输出）。
- **Pinctrl 子系统**：管**模式**（这个脚是作 GPIO 用，还是作 I2C 用？）和**特性**（上拉、下拉、驱动能力）。

在 RK3568 这种复杂的 SoC 上，Pinctrl 尤为重要。我将结合你提供的目录结构，为你拆解这个子系统。

## 📂 相关文件

想要有能力自己分析问题，你必须知道代码在哪。对于 RK3568：

1.  **Pinctrl 核心框架（Linux 官方写的，也就是教程里分析的通用的部分）**：

    - 路径：`drivers/pinctrl/core.c` (核心逻辑)
    - 路径：`drivers/pinctrl/pinmux.c` (复用逻辑)
    - 路径：`drivers/pinctrl/pinconf.c` (配置逻辑)
    - _作用：制定标准，提供 `pinctrl_register` 等 API。_

2.  **RK3568 平台驱动（Rockchip 厂家写的，负责操作硬件寄存器）**：

    - 路径：`drivers/pinctrl/pinctrl-rockchip.c`
    - _作用：这是最重要的文件！它读取 DTS，解析 RK 特有的属性，并真正去写 RK3568 的硬件寄存器。_

3.  **设备树配置（我们修改的地方）**：
    - 路径：`arch/arm64/boot/dts/rockchip/rk3568-pinctrl.dtsi`
    - _作用：定义了 RK3568 所有引脚的预设配置。_

---

---

## 🗺️ 第一部分：Pinctrl 子系统在讲什么？（全局观）

想象 RK3568 是一个拥有几百个引脚的“瑞士军刀”。同一个引脚（比如 GPIO0_B7），既可以是普通的刀片（GPIO），也可以是剪刀（PWM），还可以是螺丝刀（UART）。

**Pinctrl 子系统就是那个“选择机构”**：

1.  **复用（Muxing）**：决定把这个引脚连到芯片内部的哪个模块（GPIO 控制器？PWM 控制器？）。
2.  **配置（Conf）**：决定这个引脚的电气属性（上拉、下拉、驱动电流大小）。

---

## ⚙️ 第二部分：概念与源码对应（硬核知识通俗化）

### 1. Pin, Group 和 Function

- **Pin (引脚)**：芯片上最基本的物理焊盘。
- **Group (引脚组)**：有时一个功能需要多个引脚（比如 UART 需要 TX 和 RX）。这几个引脚这就叫一个 Group。
- **Function (功能)**：这个 Group 要干什么？是做 "uart" 还是做 "gpio"？

**在代码中（`pinctrl-rockchip.c`）：**
厂家驱动里会定义一堆数组，把物理引脚编号和名字对应起来。

### 2. 核心结构体：`pinctrl_map`

这是最关键的概念。**Map（映射）** 就像是一个“订单”。
当我们在设备树里写下配置时，内核会把它解析成一张张 `map`。

- **Map 包含什么？**
  1.  **设备名**：谁要用这个引脚？（比如 `&uart1`）
  2.  **状态名**：这是什么状态下的配置？（比如 `default` 或 `sleep`）
  3.  **控制类型**：是复用（MUX）还是配置（CONFIG）？

### 3. 从 DTS 到 Map：`dt_node_to_map`

这是驱动初始化的核心流程。
**函数位置**：`drivers/pinctrl/pinctrl-rockchip.c` 中的 `rockchip_dt_node_to_map`。

**流程解析：**

1.  内核启动，解析设备树。
2.  遇到 Pinctrl 节点。
3.  调用 `dt_node_to_map`。
4.  这个函数会把 DTS 里的 `<&gpio0 RK_PB7 RK_FUNC_GPIO &pcfg_pull_up>` 解析出来，生成内核能看懂的配置表。

### `dt_node_to_map`函数详细讲解

个人感觉只需知道是干嘛的就行，细节可以等学完再抠。
简单来说，**`rockchip_dt_node_to_map` 是一个“翻译官”**。

它的工作是：将 Rockchip 驱动内部自定义的 **Group（引脚组）结构体**，转换成 Linux Pinctrl 子系统通用的 **Map（映射）结构体**。

我们结合你提供的代码和 RK3568 的 DTS 实例，把这几行代码像剥洋葱一样拆解开来。

---

#### 0. 核心背景：数据从哪来？

在分析代码前，必须先明白一个**隐藏的前提**。
在这个函数被调用之前，Rockchip 的驱动（`pinctrl-rockchip.c`）其实已经干了一件大事：**预解析**。

驱动在初始化时，已经扫描了设备树里的 `rockchip,pins` 属性，把每一个 pinctrl 节点（比如 `i2c0_xfer`）解析成了一个 `rockchip_pin_group` 结构体，存放在 `info` 里了。

- **`grp` (Rockchip 内部结构)**：已经存好了“这个组有哪些引脚”、“每个引脚要配什么电平”。
- **`map` (Linux 通用结构)**：是这个函数要生成的目标。

---

#### 1. 代码逐行拆解与实战映射

假设我们的 DTS 是这样的（I2C0 的例子）：

```dts
/* rk3568-pinctrl.dtsi */
&pinctrl {
    /* 父节点：代表 Function (功能) */
    i2c0 {
        /* 子节点：代表 Group (组) */
        i2c0_xfer: i2c0-xfer {
            /* 具体的配置数据 */
            rockchip,pins =
                /* 0号控制器, PB1引脚, 功能1(I2C), 无上拉 */
                <0 RK_PB1 1 &pcfg_pull_none>,
                /* 0号控制器, PB2引脚, 功能1(I2C), 无上拉 */
                <0 RK_PB2 1 &pcfg_pull_none>;
        };
    };
};
```

现在内核要解析 `i2c0_xfer` 这个节点，传入的参数 `np` 就是 `i2c0-xfer` 的节点指针。

#### 第一步：找到对应的 Group (查户口)

```c
	/* 通过节点名字，在已加载的驱动数据中找到对应的 group 结构体 */
	grp = pinctrl_name_to_group(info, np->name);
	if (!grp) {
        // ... 报错 ...
		return -EINVAL;
	}
```

- **解析**：`np->name` 是 `"i2c0-xfer"`。驱动在之前已经把这个名字注册过一遍了。这里就是通过名字把之前解析好的数据（包含那是哪几个引脚、配置值是多少）拿出来，赋值给 `grp`。
- **此时 `grp` 里有什么**：
  - `grp->name`: "i2c0-xfer"
  - `grp->npins`: 2 (因为 `rockchip,pins` 里写了两行)
  - `grp->pins`: [GPIO0_B1 的编号, GPIO0_B2 的编号]
  - `grp->data`: [预存的配置值 1, 预存的配置值 2]

#### 第二步：计算内存大小 (算账)

```c
	/*
     * map_num 初始为 1，是为了放 "MUX Map" (复用映射)。
     * 加上 grp->npins，是为了放 "CONFIG Map" (电气配置映射)。
     * 每个引脚都需要单独的一个 CONFIG Map。
     */
	map_num += grp->npins;

    /* 申请内存数组 */
	new_map = kcalloc(map_num, sizeof(*new_map), GFP_KERNEL);
```

- **对应 DTS**：我们有两个引脚。所以 `map_num` = 1 (Mux) + 2 (Configs) = 3。
- **为什么分开算？** 因为 Linux 规定，“把这一组引脚切到 I2C 模式”是一个动作（MUX），而“把 PB1 设为无上拉”、“把 PB2 设为无上拉”是针对具体引脚的独立动作（CONFIG）。

#### 第三步：构建 MUX Map (填第一张表：复用)

这是最关键的一步，决定了引脚的**功能**。

```c
	/* 获取父节点，也就是 DTS 里的 "i2c0" 节点 */
	parent = of_get_parent(np);

    // ...

	/* 设置类型为 MUX_GROUP (复用组) */
	new_map[0].type = PIN_MAP_TYPE_MUX_GROUP;

	/* 重点！Function 的名字来源于父节点的名字 */
	new_map[0].data.mux.function = parent->name; // "i2c0"

	/* Group 的名字来源于当前节点的名字 */
	new_map[0].data.mux.group = np->name;        // "i2c0-xfer"

    of_node_put(parent);
```

- **逻辑揭秘**：这里解释了为什么 Rockchip 的 pinctrl DTS 结构必须是两层！
  - **父节点名**（`i2c0`）自动变成了 Linux Pinctrl 里的 **Function Name**。
  - **子节点名**（`i2c0-xfer`）自动变成了 Linux Pinctrl 里的 **Group Name**。
- **结果**：`new_map[0]` 告诉核心层：“请把 `i2c0-xfer` 这一组引脚，切到 `i2c0` 这个功能上去。”

#### 第四步：构建 CONFIG Map (填剩下的表：电气配置)

```c
	/* 指针后移，跳过刚才填好的第0个 map，开始填第1, 2...个 */
	new_map++;

    /* 遍历每一个引脚 (PB1, PB2) */
	for (i = 0; i < grp->npins; i++) {
        /* 类型是针对单个引脚的配置 */
		new_map[i].type = PIN_MAP_TYPE_CONFIGS_PIN;

        /* 1. 获取这个引脚的名字 (如 "GPIO0_B1") */
		new_map[i].data.configs.group_or_pin =
				pin_get_name(pctldev, grp->pins[i]);

        /* 2. 把之前预解析好的配置值 (&pcfg_pull_none) 塞进去 */
		new_map[i].data.configs.configs = grp->data[i].configs;
		new_map[i].data.configs.num_configs = grp->data[i].nconfigs;
	}
```

- **操作**：它从 `grp` 结构体里把之前从 `rockchip,pins` 解析出来的配置值（比如上拉、驱动能力）直接复制到了 `map` 里。
- **结果**：
  - `new_map[1]` 告诉核心层：“把 `GPIO0_B1` 这个引脚，配置成 `pcfg_pull_none`。”
  - `new_map[2]` 告诉核心层：“把 `GPIO0_B2` 这个引脚，配置成 `pcfg_pull_none`。”

---

#### 2. 总结：数据流向图

为了让你看透彻，我们画个数据流向：

1.  **DTS 源文件**:

    ```text
    i2c0 {                 <-- 父节点名
        i2c0_xfer {        <-- 子节点名
             rockchip,pins = <...配置数据...>;
        }
    }
    ```

2.  **Rockchip 驱动预处理 (`rockchip_pinctrl_parse_groups`)**:
    drivers/pinctrl/pinctrl-rockchip.c，入口函数：rockchip_pinctrl_probe

    - 生成 `struct rockchip_pin_group grp`:
      - `name` = "i2c0-xfer"
      - `pins` = [PB1, PB2]
      - `configs` = [NoPull, NoPull]

3.  **本函数 (`rockchip_dt_node_to_map`) 的转换**:
    - 读取 `grp` 和 `父节点名`。
    - **生成 Map 数组**:
      - **Map[0] (MUX)**:
        - Function: "i2c0" (来自父节点名)
        - Group: "i2c0-xfer" (来自子节点名)
      - **Map[1] (CONFIG)**:
        - Pin: "GPIO0_B1"
        - Config: NoPull
      - **Map[2] (CONFIG)**:
        - Pin: "GPIO0_B2"
        - Config: NoPull

现在我们把所有知识串起来：

1.  **系统启动 (Probe 阶段)**：

    - 执行 `rockchip_pinctrl_parse_groups`。
    - 它把 DTS 里的 `<0 RK_PB1 1 &pcfg_pull_none>` **读** 出来。
    - **存** 到了 `info->groups` 这个链表里。
    - _注意：此时还没有生成 pinctrl_map，只是把数据从 DTS 搬到了内存里备用。_

2.  **I2C 驱动加载 (Bind 阶段)**：

    - I2C 驱动说：“我要用 `i2c0-xfer`”。
    - 内核核心层调用 `rockchip_dt_node_to_map`。

3.  **生成映射 (转换阶段 - 上一个问题的内容)**：
    - `rockchip_dt_node_to_map` 调用 `pinctrl_name_to_group`。
    - **去哪里找？** 就去 **第 1 步** 生成的那个 `info->groups` 链表里找！
    - 找到后，把这些内部数据，填入 Linux 标准的 `map` 结构体返回。

#### 3.💡 形象比喻

**Pinctrl 节点解析流程 = 搬运工 + 翻译官**

1.  **搬运工 (Probe 时)**：`rockchip_pinctrl_parse_groups` 负责把 DTS 里那串难懂的数字（`rockchip,pins`），一个个抠出来，整理好，放在家里的仓库（`rockchip_pin_group` 结构体）里。
2.  **翻译官 (被调用时)**：`rockchip_dt_node_to_map` 负责当有人来要数据时，去仓库里把数据拿出来，翻译成普通话（Linux 标准 `pinctrl_map`），填到表格里交出去。

#### 4. 分析建议

下次如果你想自己分析这种函数，抓住这两个重点：

1.  **输入输出是什么？**
    - 这里输入是 `device_node` (DTS 节点)，输出是 `pinctrl_map` (Linux 标准结构)。所以这肯定是一个转换函数。
2.  **DTS 的层级结构对应代码的哪个变量？**
    - 看到 `parent = of_get_parent(np)` 和 `parent->name` 赋值给 `function`，你就应该恍然大悟：**“原来 Rockchip 强制要求写两层节点，是因为它把父节点名字当成了功能名！”**

### 💻 第四部分：实战 —— 怎么用？（Consumer 角度）

作为一个驱动工程师，我们通常是 **Consumer（消费者）**。

#### 场景 1：最常见的“自动生效”

你可能会问：“我写 Platform 驱动时，没调用任何 pinctrl 函数，为什么引脚就自动配好了？”

**秘密在于**：Linux 的驱动核心模型（Driver Core）在调用你的 `probe` 函数之前，会自动处理 Pinctrl。

**DTS 写法**：

```dts
/* 在你的设备节点里 */
my_device {
    compatible = "my,device";

    /* 核心魔法 */
    pinctrl-names = "default";  /* 状态名字叫 default */
    pinctrl-0 = <&my_pin_config>; /* 对应具体的引脚配置节点 */
};

/* 在 pinctrl 节点里 (通常在 rk3568-pinctrl.dtsi 或你的 dts 底部) */
&pinctrl {
    my_pins {
        /*
        my_pin_config (冒号前)：这是给编译器看的标签（类似于 C 语言里的变量名）。这个名字你可以随便起，叫 abc、led_cfg 都行。
        my-pin-config(冒号后)：这是最终生成在 /proc/device-tree 里的节点名。
        */
        my_pin_config: my-pin-config {
            /*
             * 格式：<控制器号 引脚号 复用功能 配置>
            第一个数：0（GPIO控制器编号）
            第二个数：RK_PB7（引脚编号）
            第三个数：RK_FUNC_GPIO/0（GPIO功能） 查数据手册（7：属于高位，0123属于低位）
            第四个数：&pcfg_pull_up（电气配置）
            这些名字不是随便写的，它们定义在 arch/arm64/boot/dts/rockchip/rk3568-pinctrl.dtsi 文件底部。
            常用的有这几个（死记硬背即可）：
            &pcfg_pull_up：开启内部上拉电阻（默认高电平）。
            &pcfg_pull_down：开启内部下拉电阻（默认低电平）。
            &pcfg_pull_none：无上下拉（浮空，通常用于输出模式或外部已有上拉）。
            &pcfg_output_high：设为输出且默认高。
            &pcfg_output_low：设为输出且默认低。
             */
            rockchip,pins = <0 RK_PB7 RK_FUNC_GPIO &pcfg_pull_up>;

        };
    };
};
```

**原理：**

1.  内核加载你的驱动。
2.  在执行 `probe` **之前**，内核核心层调用 `pinctrl_bind_pins`。
3.  它发现 DTS 里有 `pinctrl-names = "default"`。
4.  自动把引脚切换到 `pinctrl-0` 定义的状态。
5.  **进入你的 `probe` 函数时，引脚已经是你想要的状态了！**

#### 场景 2：手动切换状态

如果你想在驱动运行过程中，把一个引脚从 GPIO 模式切成 PWM 模式（比如为了省电或分时复用）。或者：控制一个引脚让它在两种状态之间动态切换（以此为例）

- 状态 A (Default)：配置为 GPIO 模式，且 上拉 (Pull Up)。
- 状态 B (Sleep)：配置为 GPIO 模式，但 下拉 (Pull Down)。

**C 代码实战**

1. 在 `&pinctrl` 节点下定义两种“施工方案”

```dts
&pinctrl {
    /* 自定义一个子节点 */
    my_pinctrl_test {

        /* 方案 1: 上拉 */
        my_pin_active: my-pin-active {
            /* <控制器号 引脚号 复用功能 配置phandle> */
            /* 1 = RK_FUNC_GPIO, &pcfg_pull_up 是内核预定义的节点 */
            rockchip,pins = <0 RK_PB7 1 &pcfg_pull_up>;
        };

        /* 方案 2: 下拉 */
        my_pin_sleep: my-pin-sleep {
            /* &pcfg_pull_down 是内核预定义的节点 */
            rockchip,pins = <0 RK_PB7 1 &pcfg_pull_down>;
        };
    };
};
```

2. 在你的设备节点里“引用”这些方案

```dts
my_pinctrl_drv {
    compatible = "my,pinctrl_test";

    /*
     * 关键！这里定义了两个状态的名字。
     * 这里的名字对应代码里 pinctrl_lookup_state 的参数。
     */
    pinctrl-names = "default", "my_sleep";

    /* 对应上面的方案 */
    pinctrl-0 = <&my_pin_active>; /* 使用标签名，而非节点名。对应 default */
    pinctrl-1 = <&my_pin_sleep>;  /* 对应 my_sleep */

    status = "okay";
};
```

3. 编写驱动代码

```C
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/pinctrl/consumer.h> /* 必须包含：Pinctrl 消费者头文件 */
#include <linux/delay.h>

struct pinctrl *my_pinctrl;
struct pinctrl_state *state_default;
struct pinctrl_state *state_sleep;

static int my_probe(struct platform_device *pdev)
{
    int ret;
    printk("========== Pinctrl Test Probe ==========\n");

    /* 1. 获取 Pinctrl 句柄 */
    /* 这步会把该设备的所有 Pinctrl 信息读进来 */
    my_pinctrl = devm_pinctrl_get(&pdev->dev);
    if (IS_ERR(my_pinctrl)) {
        dev_err(&pdev->dev, "Failed to get pinctrl\n");
        return PTR_ERR(my_pinctrl);
    }

    /* 2. 查找我们在 DTS 里定义的状态 */
    /* "default" 对应 pinctrl-0 */
    state_default = pinctrl_lookup_state(my_pinctrl, "default");
    if (IS_ERR(state_default)) {
        dev_err(&pdev->dev, "Could not find default state\n");
        return PTR_ERR(state_default);
    }

    /* "my_sleep" 对应 pinctrl-1 */
    state_sleep = pinctrl_lookup_state(my_pinctrl, "my_sleep");
    if (IS_ERR(state_sleep)) {
        dev_err(&pdev->dev, "Could not find sleep state\n");
        return PTR_ERR(state_sleep);
    }

    /* 3. 开始表演：切换状态 */

    /* 刚进来时，内核应该已经自动帮我们设为 default (上拉) 了 */
    printk("Current State: DEFAULT (Pull-Up)\n");
    /* 你可以在这里测量电压，或者看 debugfs */

    msleep(5000); /* 延时5秒，让你有时间反应 */

    /* 切换到 Sleep (下拉) */
    printk("Switching to: SLEEP (Pull-Down)...\n");
    ret = pinctrl_select_state(my_pinctrl, state_sleep);
    if (ret < 0) {
        dev_err(&pdev->dev, "Failed to select sleep state\n");
        return ret;
    }
    printk("Switch Success! Check your pin now.\n");

    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    printk("Driver Removed\n");
    return 0;
}

static const struct of_device_id my_match[] = {
    { .compatible = "my,pinctrl_test" },
    { },
};
MODULE_DEVICE_TABLE(of, my_match);

static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my_pinctrl_test",
        .of_match_table = my_match,
    },
};

module_platform_driver(my_driver);
MODULE_LICENSE("GPL");
```

验证：

1. 编译并加载：

```
insmod pinctrl_test_drv.ko
```

观察串口打印，你会看到它先是 Default，5 秒后变成 Sleep。 2. 验证方法一：使用万用表
如果引脚悬空，Pull-Up 时应该是高电平（3.3V）。Pull-Down 时应该是低电平（0V）。
现象：加载驱动 5 秒后，电压跳变。

1. 验证方法二：使用 Debugfs
   在切换发生前（或注释掉切换代码），执行：

```
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinconf-pins | grep 23
(假设 GPIO0_B7 的全局 ID 是 23，具体ID你要自己算一下或者搜一下)
```

你应该能看到类似 pull up 的字样。在切换发生后，再次执行上面的命令：
你应该能看到它变成了 pull down。

---

### 🕵️‍♂️ 第五部分：调试方法

如果引脚没反应，或者电压不对，不要瞎猜，用 **Debugfs** 查看内核的真实情况。

**1. 查看所有引脚的当前状态**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pins
```

- _分析方法_：搜索你的引脚号（如 GPIO0_B7 对应的编号）。看它的 owner 是谁？如果是 `(null)` 说明没申请成功；如果是 `my_device` 说明被你的驱动占用了。

**2. 查看引脚的复用功能 (Mux)**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins
```

- _分析方法_：看右边的功能。是 `gpio` 还是 `pwm`？

**3. 查看引脚的电气配置 (Conf)**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinconf-pins
```

- _分析方法_：这能看到更底层的寄存器配置，比如 `pull up/down` 是否生效。

**4. 查看 Maps (设备树解析结果)**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-maps
```

- _分析方法_：如果你在 DTS 里写了但没生效，查查这里有没有生成对应的 map。如果没有，说明 DTS 语法或节点位置写错了。

---

### 🚀 总结与练习建议

**Pinctrl 的核心流程：**
DTS (编写配置) -> `pinctrl-rockchip.c` (解析并注册) -> Driver Core (在 Probe 前自动应用) -> 硬件生效。

**建议的练习步骤：**

1.  **只改 DTS**：找一个没用的引脚，在设备树里配置它为 GPIO 模式，加一个上拉（pull-up）。
2.  **验证**：启动板子，不写驱动，直接用万用表量那个脚。如果是高电平（3.3V），说明 Pinctrl 生效了（因为默认可能是悬空或下拉）。
3.  **写驱动**：按照“场景 2”的代码，写一个简单的驱动，每隔 5 秒切换一次状态（比如改变驱动能力 strength，或者在 GPIO 和特殊功能间切换），用示波器或万用表观察变化。

掌握了 `pinctrl-names`、`pinctrl-0` 和 `/sys/kernel/debug/pinctrl`，你就掌握了 Pinctrl 的精髓！
