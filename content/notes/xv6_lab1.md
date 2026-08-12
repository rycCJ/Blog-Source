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
*s ：如果输入atoi("123") 则第一次进while 时候 *s为 '1' 故值为 '1' 的ASCII 49;
 *s++：后置自增 先拿当前指针内容，指针再加一。
 *(++s)：指针先 + 1，再取新位置字符。

 ```c kernel/sysproc.c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;
  // 刚开始：进程：RUNNING  锁：UNLOCKED
  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);  //从用户态获取系统调用参数。执行此操作之后：tickslock变为LOCKED（已上锁），除非release（释放锁）,否则其他进程就不能直接拿到锁
  // 此时：进程：RUNNING   锁：LOCKED
  // 锁的目的就是:同一时刻只能让一个执行流进入临界区。
  ticks0 = ticks;
  while(ticks - ticks0 < n){  
    if(myproc()->killed){         // myproc()获取当前进程
      release(&tickslock);  //发现进程被杀死，就要立即释放锁
      return -1;
    }
    sleep(&ticks, &tickslock);  //执行xv6 内核内部的进程睡眠函数。叫做：阻塞等待 / 睡眠
    // 作用：让当前进程进入 Sleeping 状态（不再占用 CPU），并等待 ticks 这个等待通道被唤醒。
    // 睡眠之前需要正确释放、唤醒后需要重新获取的锁。如果睡眠的时候不释放锁，其他进程想要锁的话 就得一直等待。
    // 此时：进程：SLEEPING   锁：UNLOCKED
    // 不使用while检查的原因（  while (ticks - ticks0 < n) {    // 什么都不做}  ）：会一直检查，占用CPU。使用while也叫忙等待

  }
  release(&tickslock);
  return 0;
}
 ```
 程序流程：用户程序调用 sleep(5) -->进入 sys_sleep() -->argint(0, &n) -->n = 5 -->acquire(tickslock) --> ticks0 = ticks -->ticks0 = 100 --> ticks - ticks0 < 5 ? --yes--> 检查 killed 状态 -->  sleep(&ticks,...) 当前进程 SLEEPING --CPU 执行其他进程--> 时钟中断发生（每当一个 Tick 到来时，时钟中断会唤醒等待 &ticks 的进程） -->  ticks++ --> wakeup(&ticks) -->当前进程变成 RUNNABLE (等待调度器)-->  以后重新获得 CPU(进程变为RUNNING) --> 回到 while 判断 --> 已经到 5 Tick --> release(tickslock) --> return 0

 sys_sleep() → sleep() → sched() → scheduler() → 定时器中断 → wakeup() → 进程从 SLEEPING 变成 RUNNABLE
其中：sched()，当前进程 A 已经不能继续运行了，请把 CPU 交给其他进程。当前进程放弃 CPU，进行上下文切换
scheduler()调度器：寻找并切换到下一个可以运行的进程
不同对象有不同的状态机。锁通常有“已占用/未占用”两种核心状态；进程则有 RUNNING、RUNNABLE、SLEEPING 等多种状态。对象的状态由对应的操作和系统事件改变，例如 acquire() / release() 改变锁状态，scheduler()、sleep()、wakeup()、exit() 等改变进程状态。

```C user/ls.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"  //DIRSIZ 是 xv6 中文件名部分允许保存的长度，通常为 14。

char*
fmtname(char *path)
{
  static char buf[DIRSIZ+1];
  char *p;

  // Find first character after last slash.
  for(p=path+strlen(path); p >= path && *p != '/'; p--)            //指向末尾 \0,      p-- 不断向左移动，逐个判断字符是不是/       一直往左走到路径里最后一个 / 的位置停下
    ;
  p++;         //  跳过 /，此时 p 指向字符串 "a.txt"

  // Return blank-padded name.
  if(strlen(p) >= DIRSIZ)
    return p;
  memmove(buf, p, strlen(p));          //将 "a.txt" 拷贝进静态数组；
  memset(buf+strlen(p), ' ', DIRSIZ-strlen(p));         //后面填充空格，凑齐 14 个字符，保证打印对齐；
  return buf;         //返回填充好空格的 "a.txt "。
}

void
ls(char *path)
{               
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, 0)) < 0){         // open("./testdir", 0) 只读打开目录，拿到文件描述符 fd
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){                   // fstat 通过 fd 获取文件元数据存入 st
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){
  case T_FILE:  //普通文件
    printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);
    break;

  case T_DIR:    //目录
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("ls: path too long\n");
      break;
    }
    strcpy(buf, path);
    p = buf+strlen(buf);             //p 指向字符串末尾的 \0
    *p++ = '/';              //在路径末尾拼接 /                 此时 buf 内容："./testdir/"
    while(read(fd, &de, sizeof(de)) == sizeof(de)){             //循环读取目录内每一条目录项
      if(de.inum == 0)                 //代表空闲无效条目，直接跳过。
        continue;
      memmove(p, de.name, DIRSIZ);          //拿到：name = "a.txt"               把 "a.txt" 复制到 buf 里 / 的后方       
      p[DIRSIZ] = 0;    //p[DIRSIZ] = 0 手动补 \0，变成合法字符串；现在 buf = "./testdir/a.txt"

      if(stat(buf, &st) < 0){           //获取该文件属性：
        printf("ls: cannot stat %s\n", buf);       
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);             //st.type=T_FILE、inode 编号、文件大小
    }
    break;
  }
  close(fd);
}

int
main(int argc, char *argv[])  // ls ./testdir
{
  int i;

  if(argc < 2){
    ls(".");
    exit(0);
  }
  for(i=1; i<argc; i++)
    ls(argv[i]);      //ls("./testdir")          目录内部包含两个文件：a.txt（普通文件）、b（子目录）；
  exit(0);
}


```
函数
struct dirent	目录中保存的简要目录项，主要有文件名和 inode 编号
struct stat	文件的详细信息，包含类型、inode、大小等
fd	进程访问已打开文件或目录时使用的编号
stat(path, &st)	根据路径获取信息
fstat(fd, &st)	根据已打开的文件描述符获取信息
read(fd, &de, sizeof(de))	从目录文件中读取一个目录项
流程：
main 接收命令行路径，丢给 ls()；
ls 打开路径，判断是文件还是目录；
文件：直接格式化文件名打印属性；
目录：拼接父路径 +/，循环读取目录内所有条目，拼成完整路径获取属性；
fmtname 专门剥离路径最后的文件名、补齐空格对齐输出；
全部处理完成关闭文件描述符，程序结束。

int strcmp(const char *s1, const char *s2);
按ASCII 码值逐字节比较两个以 \0 结尾的 C 风格字符串：
从第 0 个字符开始一一对比，直到出现不一样的字符，或者其中一个字符串结束。

返回 0：s1 == s2，两个字符串完全一模一样；
返回正数：s1 > s2（第一个不同字符 s1 的 ASCII 更大）；
返回负数：s1 < s2（第一个不同字符 s1 的 ASCII 更小）。
判断两个字符串相等：
```c
if (strcmp(str1, str2) == 0)
```
错误写法（极其高频踩坑）：
```c
// 错误！直接比较指针地址，不是比较字符串内容 ,== 只会对比两个指针存的内存地址，不会挨个对比字符。
if (str1 == str2)
```


```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
//递归找
int 
find(char *path,char *filename){
  
}

int main(int argc,char *argv[]){
      if(argc<3){
          fprintf(2,"usage:find <path> <filename1,filename2...>");
        }
  for(i=2;i<argc;i++)
  {
    find(argv[1],argv[2]);
  }
  exit(0);

}
```
xargs:
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/param.h"
#include "user/user.h"
#include "kernel/fs.h"

    /*
    //对于|->left 输出当作 |->right的输入
// 对于：|->left
    // read(0,buf,1);这里的0是指从标准输入（stdin）读取一个字符。（1：stdout（标准输出）：屏幕）（2：stderr（标准错误）：屏幕）
        eg:
        char ch;
        read(0, &ch, 1);
        printf("%c\n", ch);
        运行
        $ test
        A
        A
        即：
        键盘
        │
        ▼
        stdin (fd = 0)
        │
        read(0, &ch, 1)
        │
        ▼
        ch = 'A'
        │
        printf(...)
        │
        stdout (fd = 1)
        │
        屏幕

        eg2:
        echo hello | xargs echo

        管道把 echo hello 的输出连接到了 xargs 的标准输入：

        因此：
        read(0, &ch, 1);
        读到的为：
            h
            e
            l
            l
            o
            \n

    //当这个输入是一行一行的时候（比如上述例子的hello,需要处理将其放入一个buffer中，遇到\n代表），没有getline()函数，所以只能使用read
        eg:
        abc
        def
        ghi
        真正想要的：第一次：abc；第二次：def；第三次：ghi
        只能：read(0,&ch,1)，放到buffer,遇到\n说明一行结束；

        不停 read()

        ↓

        不是 '\n'

        ↓

        放进 buffer

        ↓

        遇到 '\n'

        ↓

        buffer 加 '\0'

        ↓

        开始执行命令


//对于|->right :
    // xargs echo hello
    有：
    argc = 3

    argv[0] = "xargs"
    argv[1] = "echo"
    argv[2] = "hello"
    argv[3] = 0


//执行时候：
    想要把|->left输出加到|->right的输入，就要构造新的数组argv，这个数组是argv与buffer的组合。
    char *newargv[MAXARG];
    newargv[0]=argv[1];
    newargv[1]=argv[2];
    ...
    newargv[last]=buffer;
流程：
    while(读到一行)
{
    fork()

        child
            exec()

        parent
            wait()
}
    */
    #define MAXSTDIN 16
int
main(int argc,char *argv[]){
    /*
    char buf[MAXSTDIN];
    read(0,buf,1);
    printf("|->left output:%s\n",buf);
    //echo hello | xargs grep aaa得到：|->left output:hello
    */


    /*
    for(int i = 0;i<argc;i++){
        printf("argv[%d]:%s\n",i,argv[i]);

    }
    //echo hellp | xargs echo a b c 得到：
    argv[0]:xargs
    argv[1]:echo
    argv[2]:a
    argv[3]:b
    argv[4]:c
    */
    // char *p = argv[2];
    // for(int i = 2;i<argc;i++){
    //     exec(argv[1],p);
    //     p++;

    // }
    // exec("echo",argv);  //echo hellp | xargs echo a b c 得到：echo a b c
    
    // echo hello | xargs echo a b c 希望得到：a b c hello

    char *newargv[MAXARG];
    char buffer[128];
    char ch;
    int n=0;
    //完成buffer构造，不是 '\n'，放进buffer,是\n，加\0,完成第一行
    
    while(read(0,&ch,1)>0)
    {
        if(ch!='\n'){
            if(n>=sizeof(buffer)-1){
                continue;
            }
            buffer[n] = ch;
            n++; 
        }
        else{
            buffer[n] = '\0';
            n = 0;
            // printf("buffer is : %s\n",buffer);
            // memset(newargv,0,sizeof(newargv));
            for(int i =1;i<argc;i++)
            {
                newargv[i-1] = argv[i];     //如果 echo hellp | xargs echo a b c 
                // 那么argv为：argv[0]:args argv[1]:echo argv[2]:a argv[3]:b argv[4]:c 
                // 那么exec("echo",argv);  //echo hellp | xargs echo a b c 得到：echo a b c
                // newargv为: newargv[0]:echo  ewargv[1]:a  ewargv[2]:b  ewargv[3]:c  
                // 那么exec("echo",newargv);  得到 a b c
            }
            newargv[argc-1] = buffer;   //argc=5 newargc=4 newargv[4]位置接（echo hellp | xargs echo a b c）hellp
            // newargv为: newargv[0]:echo  newargv[1]:a  newargv[2]:b  newargv[3]:c  newargv[4]:hellp  
            newargv[argc] = 0;
            int pid = fork();
            if(pid<0){
                fprintf(2,"error fork!\n");
                continue;
            }
            if(pid==0){
                //子
                exec(newargv[0],newargv); //得到 a b c hellp  exec成功不反悔，失败返回
                //当 exec 执行成功时，它不会返回到调用程序；相反，从文件中加载的指令会从 ELF 头中声明的入口点开始执行。
                fprintf(2, "exec %s failed\n", newargv[0]);
                exit(1);
            }
            else{
                //father
                wait(0);
            }
            
            
        }
    }
    //最后一行没有\n , EOF时如果buffer里还有内容，再执行一次
    if(n>0){
        buffer[n] ='\0';
        // memset(newargv,0,sizeof(newargv));
        //构造newargv
        for(int i =1;i<argc;i++){
            newargv[i-1] = argv[i]; //xargc echo a b c argc=5
            //new:echo a b c 
        }
        newargv[argc-1] = buffer;
        newargv[argc] = 0;
        int pid = fork();
        if(pid<0){
            fprintf(2,"fork error\n");
        }
        if(pid==0){
            //child
            exec(newargv[0],newargv);
            fprintf(2, "exec %s failed\n", newargv[0]);
            exit(1);
        }
        else{
            //father
            wait(0);
        }

    }

    exit(0);
}
```