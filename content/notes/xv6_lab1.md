---
title: 'Xv6 Lab1' # <--- 修改这一行
date: "2026-07-29T10:38:38+08:00"
draft: false
tags: ["xv6", ""]
location: "广州"
---
```c user/ulib.c
 int
atoi(const char *s)
{
  int n;

  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}
 ```
 功能：把数字字符串转成整数
 *s++：后置自增 先拿当前指针内容，指针再加一。
 *(++s)：指针先 + 1，再取新位置字符。