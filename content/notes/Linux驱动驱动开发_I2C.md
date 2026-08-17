---
title: "Linux驱动驱动开发 I2C" # <--- 修改这一行
date: "2026-01-15T20:32:44+08:00"
draft: false
tags: ["", ""]
location: ""
---

|      | MIPI                  | LVDS                      |
| ---- | --------------------- | ------------------------- |
| 引脚 | I2C1/GPIO3            | I2C1/GPIO3                |
| 芯片 | ft5x06                | ft5x06                    |
| I2C  | I2C1                  | I2C2                      |
| 屏幕 | #define LCD_TYPE_MIPI | #define LCD_TYPE_LVDS_7_0 |

| 文件                                                                                       | 操作                                                                                                                             |
| ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| /home/topeet/Linux/linux_sdk/kernel/arch/arm64/configs/rockchip_linux_defconfig            | 注释 CONFIG_TOUCHSCREEN_EDT_FT5X06=y                                                                                             |
| /home/topeet/Linux/linux_sdk/kernel/arch/arm64/boot/dts/rockchip/topeet_rk3568_lcds.dtsi   | &ft5x062 { status = "okay"; };                                                                                                   |
| /home/topeet/Linux/linux_sdk/kernel/arch/arm64/boot/dts/rockchip/topeet_screen_choose.dtsi | #define LCD_TYPE_LVDS_7_0                                                                                                        |
| /home/topeet/Linux/linux_sdk/kernel/arch/arm64/boot/dts/rockchip/topeet_rk3568_lcds.dtsi   | 启用 ft5x062，comoatible 为 okay &i2c2 { status = "okay"; myft5x062:my-ft5x062@38 { compatible = "my-ft5x06"; reg = <0x38>; };}; |
