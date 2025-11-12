---
title: "Linux系统编程" # <--- 修改这一行
date: "2025-11-06T22:22:40+08:00"
draft: false
tags: [Linux", "系统编程"]
location: "广州"
---

视频：(【北京迅为】嵌入式学习之 Linux 系统编程篇)[https://www.bilibili.com/video/BV1zV411e7Cy?vd_source=de9d0b5f46420ead2e7be875b9558767&spm_id_from=333.788.videopod.episodes]
文档：【北京迅为】itop-3568 开发板系统编程手册（新）.pdf

介于应用层和驱动层之间

可以在 Linux 系统下完成基本操作
完成入门教程（视频文档都可）
视频：(【北京迅为】嵌入式学习之 Linux 入门篇)[https://www.bilibili.com/video/BV1M7411m7wT/?spm_id_from=333.1387.upload.video_card.click&vd_source=de9d0b5f46420ead2e7be875b9558767]
文档：【北京迅为】itop-3568ubuntu 使用手册.pdf  
01*【北京迅为】itop-3568 开发板快速启动手册【底板 v1.7 版】v1.1.pdf  
03*【北京迅为】itop-3568 开发板快速使用编译环境 ubuntu18.04【底板 v1.7】 v1.1.pdf

main 函数传参

```C
int main(int argc, char *argv[])
{
/* 代码 */
}
```

argc 英文全称为 argument count，表示命令行参数的数量。
argv 英文全称为 argument value，是一个指向字符串数组的指针，每个字符串表示一个命
令行参数。

文件 IO/标准 IO
文件：直接
标准：间接

## 文件 IO

文件描述符
fd = open()

open-打开文件

```C
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

close-关闭文件
read-读文件

```C
- ssize_t read(int fd, void \*buf, size_t count);
```

- 返回实际读到的字节数 如果读到文件末尾，返回 0 如果读错，返回-1

write-写文件

```C
ssize_t write(int fd, const void *buf, size_t count);
```

- 如果要写一个文件的内容，count 可以是读文件的返回值（ret = read(),write(,,ret)）,因为读文件返回的就是字节大小，相当于写了 buf 中所有东西

在 Ubuntu 中运行：
gcc <文件名>.c #编译 生成.out 文件
./文件名.out

lseek

```C
off_t lseek(int fd, off_t offset, int whence);
// whence 参数
// SEEK_SET：相对于文件开头
// SEEK_CUR:相对于当前的文件读写指针位置
// SEEK_END:相对于文件末尾

lseek(fd,0,SEEK_END);  //指针在文件末尾 返回文件总字节大小
lseek(fd,0,SEEK_CUR)  //返回当前指针所在位置之前的字节大小
```

- 返回当前位移大小

## 目录 IO

创建目录

```C
int mkdir(const char *pathname, mode_t mode);
```

打开、关闭目录

```C
DIR *opendir(const char *name);      //失败返回 NULL  成功返回打开的目录流
int closedir(DIR *dirp);       //调用成功返回 0，调用失败返回-1
```

读取目录

```C
struct dirent *readdir(DIR *dirp);  // 成功返回指向 dirent 类型结构体的指针 失败返回 NULL
```

文件在目录下面是通过链表存放的，要查看所有需要循环读取

## 库

静态库：在程序编译的时候会被链接到目标代码里 程序运行不需要该静态库 编译出来体积大 静态库以 lib 开头 以.a 结尾 eg.移植
动态库：也叫共享库 编译的时候不会被链接到目标代码里，而是在运行的时候被载入的 程序运行需要该动态库 编译出来体积小 动态库以 lib 开头 以.so 结尾

静态库制作

- 编译或准备库的源代码
- 将源码.c 文件编译成.o 文件 （gcc -c 文件名.c）
- 使用 ar 命令创建静态库 （ar cr lib 文件名.a gcc wenjian .c -l 文件名 -L .）
  - lib 文件名.a：库文件名
  - 文件名：库名
- 测试库文件 (gcc test .c -l 文件名 -L . -l:指定静态库的库名 -L 查找位置 . :当前目录下面去查找 )

动态库制作

gcc -c -fpic 库名 .c //生成.o 文件
gcc -shared -o lib 库名.so 库名.o //生成 lib 库名.so

如果我们的程序代码用到了库文件里面的函数，我们在编译的时候需要链接库。系统默认会在/ib 或者/usr/lib 去找库文件。如果我们使用的库不在里面，会提示错误
解决方法：

1. 将生成的动态库拷贝到/ib 或者/usr/lib 里面去，因为系统会默认去这俩个路径下寻找。
2. 把我们的动态库所在的路径加到环境变量里面去（只对当前终端有效）
   比如我们动态库所在的路径为/home/test，我们就可以这样添加： export LD LIBRARY PATH=$LD LIBRARY PATH:/home/test
3. 修改 ubuntu 下的配置文件/etc/ld.so.conf，在这个配置文件里面加入动态库所在的位置，然后使用命令 ldconfig 更新目录。

gcc -c -fpic 库名 .c //生成.o 文件
gcc -shared -o lib 库名.so 库名.o //生成 lib 库名.so

## 进程

当程序在 Linux 操作系统中运行起来之后就变成了进程
进程 ID
每个进程都有唯一标识符，即进程 ID，简称 pid

进程间的几种通信方法
管道通信:有名管道，无名管道
信号通信:信号的发送，信号的接受，信号的处理
IPC 通信:共享内存，消息队列，信号灯
Socket 通信

创建进程函数

```C
pid_t fork(void);
```

- 返回值为 0，则说明当前进程是子进程
- 返回值大于 0，则说明当前进程是父进程，返回值为子进程的进程 ID
- 返回值小于 0，则说明 fork() 函数调用失败。
- getpid()函数的作用为获取当前进程 pid getppid()函数的作用为获取当前进程父进程的 pid

exec 函数族

在 Linux 中使用 exec 函数族主要有以下两种情况：

1. 当进程认为自己不能再为系统和用户做出任何贡献时，就可以调用任何 exec 函数族让自己重生。
2. 如果一个进程想执行另一个程序，那么它就可以调用 fork 函数新建一个进程，然后调用任何一个 exec 函数使子进程重生。

```C
int execl(const char *path, const char *arg,..)
// 将新程序加载到当前进程空间中（新创建的（fork）进程），替换原来的程序映像(执行path arg 程序，而非源程序)，并传递参数列表
int execv(const char *path, char *const argv[])
int execle(const cha *path,const char*arg,...,char *const envp[])
int execve(const char *path, char *const argv[], char *const envp[])
int execlp(const char  *file,const char *arg, ...)
int execvp(const char *file,char *const argv[])
```

ps 和 kill 命令

ps:查看进程信息

```C
ps [options]
// a 显示所有进程信息，包括其他用户的进程
// x 显示所有进程，包括没有控制终端的进程
// u 显示用户及资源使用情况
// e 显示所有进程信息，包括没有控制终端的进程，和 x 选项相比没有进程状态
// f 以进程树的形式显示进程信息
// w 使用宽输出格式，以便完整地显示进程信息
// h 不显示标题行
// o 自定义输出格式，可以指定输出的列和列之间的分隔符
```

kill 进程：向指定的进程发送信号，以达到控制进程行为的目的

```C
kill [signal] PID
```

孤儿进程
孤儿进程是指其父进程先于它结束，从而没有父进程来对它进行资源回收和管理的进程。当进程变为孤儿进程之后，会被进程 init 进程领养

僵尸进程

在操作系统中，每个进程都有一个唯一的进程 ID，以及父进程 ID。当一个进程终止时，它的进程 ID 和状态信息会被保留在系统中，从而方便父进程的查询。
而僵尸进程（zombie）是指已经执行完毕，但是父进程没有及时回收其资源的进程

wait 函数

```C
pid_t wait(int *status);
```

父进程可以通过 wait()函数等待子进程完成某些任务，然后再继续执行自己的任务。此外，子进程可以通过 exit()函数的返回值来向父进程传递处理结果或者状态

守护进程 daemon
运行再后台，不和任何控制终端关联
两个基本要求：

1. 作为 init 进程的子进程
2. 不和控制终端交互

3. 使用 fork 函数创建一个新的进程，让父进程直接退出（exit 函数） 使得子进程成为孤儿进程 （必须）
4. 调用 setsid 函数，让其摆脱控制终端 创建新的会话 （必须）
5. 调用 chdit 函数，将当前的工作目录改变为根目录，增强程序健壮性
6. 重设 umask 文件掩码，增强程序健壮性和灵活性
7. 关闭 0，1，2 文件描述符，节省资源 close(0)...
8. 执行需要执行的代码（必须）

无名管道
只能通过文件描述符在具有亲缘关系的进程之间使用，如父子进程、兄弟进程等
在 fork 函数之前创建管道 pipe(fd)

有名管道
实现两个互不相关的进程间的通信
知道特殊文件路径，可以读写

信号通信-信号发送

```C
int kill(pid_t pid, int sig);

int raise(int sig);
// raise函数等价于kill(getpid(),sig);

unsigned int alarm(unsigned int seconds);

```

信号通信-信号接收

```C
while(1)
sleep()
int pause(void);
// 函数说明：pause(会令目前的进程暂停（进入睡眠状态）直到被信号(signal)所中断。返回值：只返回-1.
```

信号通信-信号处理

```C
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);

// signal（参数1,参数2）；
// 参数1：我们要进行处理的信号，系统的信号我们可以再终端键入kill -l查看。
// 参数2：处理的方式（是系统默认还是忽略还是捕获）。
```

1. signal(SIGINT,SIG_IGN,); // SIG_IGN，代表忽略，也就是忽略 SIGINT 信号
2. signal(SIGINT,SIG_DFL); // SIG_DFL 代表执行系统默认操作，大多数信号的系统默认动作时终止该进程。
3. signal(SIGINT,myfun); //捕捉 SIGINT 这个信号，然后执行 myfun 函数里面的代码。myfun 由我们自己定义。

### 共享内存

创建或获取一个 IPC key

```C
key_t ftok(const char *pathname, int proj_id);
//文件路径   一个整数值

```

创建共享内存

```C
int shmget(key_t key, size_t size, int shmflg);
//返回值为共享内存标识符（shmid），用于标识创建或获取的共享内存对象。如果出错，则返回 -1 并设置 errno 错误码
// key:使用 ftok()函数生成的 IPC key 值 非零  使用宏：0
//size:共享内存大小，以字节为单位
```

将共享内存映射到进程的地址空间中(效率高，不用每次进入内核)

```C
void *shmat(int shmid, const void *shmaddr, int shmflg);
//shmid: 共享内存标识符，通过 shmget() 函数创建或获取。
//shmaddr:指定共享内存段的连接地址，通常设置为 NULL，让系统自动选择一个合适的地址。
//shmflg:连接共享内存的标志位 SHM_RDONLY   SHM_RND   SHM_REMAP
```

解除共享内存映射

```C
int shmdt(const void *shmaddr);
//共享内存段的首地址
```

删除共享内存

```C
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
// shimid:shmget() 函数创建的共享内存标识符
// cmd: IPC_STAT(获取对象属性)IPC_SET(设置对象属性）IPCRMID（删除对象）
// buf:指定IPC_STAT(获取对象属性)IPC_SET(设置对象属性）时用来保存或者设置的属性。
```

### 消息队列

创建或打开消息队列

```C
int msgget(key_t key, int msgflg);

//key:ftok()函数生成的 IPC key 值
//函数调用成功，会返回一个非负整数表示消息队列的标识符（即消息队列的唯一标识），调用失败返回-1
```

ipcs -q //共享内存  
ipcs -m //消息队列

发送消息

```C
int msgsnd(int msqid, const void *msgp, size_t msgsz,
int msgflg);
```

接收消息

```C
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long
msgtyp,int msgflg);
```

**......未完待续**
