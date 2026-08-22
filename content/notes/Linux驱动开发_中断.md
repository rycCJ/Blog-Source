---
title: "Linux驱动开发 中断" # <--- 修改这一行
date: "2025-12-25T14:56:02+08:00"
draft: false
tags: ["Linux", "驱动开发"]
location: ""
---

中断下文-Tasklet
共享工作队列
自定义工作队列
延迟工作
工作队列传参步骤：

1.  打包数据包
2.  在中断下半部分拆包
    使用 container_of 获取地址

CMWQ(并发管理工作队列)
