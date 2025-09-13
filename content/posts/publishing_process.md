---
date: '2025-08-09T17:06:14+08:00'
draft: false
tags: ["Git"]
title: 'Publishing_process 博客发布流程'
summary: "  " # <--- 在这里定义摘要
---


**1.创建一篇新文章**
打开Git Bash 终端，进入根目录d:/Blog
>hugo new content posts/hello-world.md   

告诉Hugo，在 content/posts/ 目录下，创建一个名为 hello-world.md 的新文章。

**2.编写文章内容**
- 打开VS Code，在 hello-world.md里面写内容
- 将draft：true改为draft：false

**3.在本地预览效果**
- 回到Git Bash终端，运行：
  >hugo server
- 终端会启动一个本地服务器，并显示网址 http://localhost:1313/,打开网址浏览即可
- 预览完毕后，回到终端，按 Ctrl + C 组合键，停止本地服务器。

**4.提交并推送到GitHub (自动发布)**
- 添加更改到暂存区
>git add .
- 提交更改到本地仓库 (创建一个新的提交记录)
>git commit -m "feat: Add my first post 'Hello World'"

feat: 是一种规范的提交信息前缀，意思是“增加新功能”，这里指增加了新文章。
>>写了一篇新文章？用 feat。

>>修改了一篇文章的内容或错别字？用 docs。

>>修正了一个导致网站显示不正常的Bug？用 fix。

>>更新了 .gitignore 或者 deploy.yml？用 chore。

- 推送到GitHub (这是触发自动部署的扳机)

>git push origin main
  
**5.检查线上部署**
- 打开浏览器，进入您之前创建的源码仓 (Blog-Source) 的GitHub页面。
- 点击页面上方的 “Actions” 选项卡。
- 您会看到一个新的工作流（Workflow）正在运行，它前面会有一个黄色的小圆圈。这就是GitHub在后台帮您生成和部署网站。
- 等待1-2分钟，当黄色圆圈变成绿色的对勾 (✓) 时，就代表您的网站已经部署成功了。
- 现在，请在浏览器中访问您的线上博客地址：
>https://<您的GitHub用户名>.github.io