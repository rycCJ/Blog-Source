---
date: '2025-09-02T22:55:18+08:00'
draft: false
tags: ["Git"]
title: 'A Folder to Git' # <--- 修改这一行
summary: "本地文件夹上传至GitHub"
---

**步骤 1: 在 GitHub 上创建新的远程仓库**
**步骤 2: 在本地电脑上操作**

```
# 打开 Git Bash进入你的项目文件夹
cd ~/Desktop/my-first-project
# 初始化 Git 仓库:
git init
#添加所有文件到暂存区:
git add .
# 提交文件到本地仓库:
git commit -m "Initial commit"
#重命名主分支为 main
git branch -m main
# 关联本地仓库和远程仓库:
git remote add origin https://github.com/YourUsername/YourProjectName.git        # 把下面的 URL 换成你自己的仓库 URL
# 推送代码到 GitHub:
git push -u origin main
```