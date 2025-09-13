---
date: '{{ .Date }}'
draft: false
tags: ["",""]
title: '{{ .File.ContentBaseName | replaceRE "[_-]" " " | title }}' # <--- 修改这一行
summary: "在这里写下您的文章摘要..."
---
