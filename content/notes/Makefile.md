---
title: "Makefile"  # <--- 修改这一行
date: "2026-04-08T17:18:00+08:00"
draft: false
tags: ["Makefile" ]
location: "西安"
---
.c程序得到可执行程序它们之间要经过四个步骤：
1. 预处理
2. 编译：将.c文件转换为.o文件
3. 汇编：将.o文件转换为汇编语言   (前三个步骤统称为编译)
4. 链接：将多个.o文件合并为一个可执行文件

```bash 
gcc -o test a.c b.c
```

1）对于a.c：执行：编译 的过程，a.c -> xxx.s -> xxx.o 文件。
2）对于b.c：执行：编译 的过程，b.c -> yyy.s -> yyy.o 文件。
3）最后：xxx.o和yyy.o链接在一起得到一个test应用程序。

当“依赖”比“目标”新，执行它们下面的命令
```bash
test ：a.o b.o //test是目标，它依赖于a.o b.o文件，一旦a.o或者b.o比test新的时候，
就需要执行下面的命令，重新生成test可执行程序。
    gcc -o test a.o b.o
a.o : a.c //a.o依赖于a.c，当a.c更加新的话，执行下面的命令来生成a.o
    gcc -c -o a.o a.c
b.o : b.c //b.o依赖于b.c,当b.c更加新的话，执行下面的命令，来生成b.o
    gcc -c -o b.o b.c
```

通配符，用于依赖文件多的时候，例如：
```bash
test: a.o b.o
    gcc -o test $^  
%.o : %.c  //%.o：表示所用的.o文件，%.c：表示所用的.c文件
    gcc -c -o $@ $<  //$@：表示目标文件，$<：表示第一个依赖文件
```
清除文件：
```bash
clean:
    rm  *.o test
``` 
如果有clean文件(即有目标文件，没有依赖文件)，执行make clean，系统寻找：是否有依赖文件比目标文件新，发现没有办法判断依赖文件的时间，故无法执行make clean，于是显示：```make: \`clean' is up to date.```
解决办法：
```bash
.PHONY: clean //把clean定义为假象目标。他就不会判断名为“clean”的文件是否存在
```
执行：make clean，就会执行删除操作。
Makefile函数
### 函数foreach
```bash
$(foreach var, list, text)
```
eg：
```bash
A = a b c
B = $(foreach f, &(A), $(f).o)
all:
    @echo B = $(B)  
```
输出：
```bash
B = a.o b.o c.o
```

### 函数filter/filter-out
```bash
$(filter pattern...,text) # 在text中取出符合patten格式的值
$(filter-out pattern...,text) # 在text中取出不符合patten格式的值
```
eg：
```bash 
C = a b c d/
D = $(filter %/, $(C))
E = $(filter-out %/, $(C))
all:
    @echo D = $(D)
    @echo E = $(E)
```
输出：
```bash
D = d/
E = a b c
```

### 函数Wildcard
```bash
$(wildcard pattern,...) # pattern定义了文件名的格式, wildcard取出其中存在的文件。 会以 pattern 这个格式，去寻找存在的文件，返回存在文件的名字。 该、目录下创建三个文件：a.c b.c c.c
```
eg：
```bash
F = $(wildcard *.c)
all:
    @echo F = $(F)
```
输出：
```bash
F = a.c b.c c.c
```

### 函数subst
```bash     
$(subst from,to,text) # 将text中的from替换成to
```
eg：
```bash
F = a b c
G = $(subst b,x, $(F))
all:
    @echo G = $(G)
```
输出：
```bash
G = a x c
```
### 函数patsubst
```bash
$(patsubst pattern,replace,text) # 将text中符合patten格式的值，替换成replace
```     
eg：
```bash
F = a.c b.c c.c
G = $(patsubst %.c,%.o, $(F))
all:
    @echo G = $(G)
```
输出：
```bash
G = a.o b.o c.o
``` 
