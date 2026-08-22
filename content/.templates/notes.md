---
title: '{{ .File.ContentBaseName | replaceRE "[_-]" " " | title }}' # <--- 修改这一行
date: "{{ .Date }}"
draft: false
tags: ["", ""]
location: ""
---
