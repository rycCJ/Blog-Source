---
title: 'Linux驱动开发 Ioctl' # <--- 修改这一行
date: "2026-05-29T13:26:11+08:00"
draft: false
tags: ["", ""]
location: "西安"
---

对硬件，非数据的操作使用ioctl,read/write是对数据流的操作

应用程序：
ioctl系统的调用：`int ioctl(int fd, unsigned int cmd, unsigned long arg);`
ioctl 命令合成宏（参数cmd的选择）：
- 定义一个不需要参数的命令：
`#define _IO(type,nr) _IOC(_IOC_NONE,(type),(nr),0)`
- 定义一个应用程序从驱动程序读参数的命令：
`#define _IOR(type,nr,size) _IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))`
- 定义一个应用程序向驱动程序写参数命令：
`#define _IOW(type,nr,size) _IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))`
- 定义一个参数是双向传递的命令：
`#define _IOWR(type,nr,size) _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))`
- type 命令类型；nr 命令序列号；size:命令的大小

ioctl命令的构成：
|设备类型|序列号|方向|数据尺寸|
|8bit|8bit|8bit|2bit|13/14bit|
方向：无数据（00），只读（01），只写（10），读写（11）
ioctl命令的分解宏：
_IOC_TYPE(nr)：分解从驱动中读取数据的命令
_IOC_NR(nr)：分解向驱动中写数据的命令
_IOC_DIR(nr) :分解没有数据传递的命令
_IOC_SIZE(nr)：分解先写入数据再读取数据的命令


```c
#define CMD_TEST0 _IO('L',0);
#define CMD_TEST1 _IO('L',1);
#define CMD_TEST2 _IO('A',0);

#define CMD_TEST3 _IOW('A',1,int)
#define CMD_TEST4 _IOR('A',2,int)
#define CMD_TEST5 _IOWR('A',3,int)

int main(int argc, char *argv[]){
    printf("CMD_TEST3 type is %ld\n",_IOC_TYPE(CMD_TEST3));  //65 A的ASCII码
    printf("CMD_TEST3 nr is %ld\n",_IOC_NR(CMD_TEST3));  //1
    printf("CMD_TEST3 dir is %ld\n",_IOC_DIR(CMD_TEST3));  //1
    printf("CMD_TEST3 size is %ld\n",_IOC_SIZE(CMD_TEST3));
}
```
应用程序：
```c
#define CMD_TEST0 _IO('L',0)
#define CMD_TEST1 _IOW('L',1,int)
#define CMD_TEST2 _IOR('L',2,int)


int main(int argc,char *argv[]){
   //······
   int val;
    fd = open("/dev/test",O_RDWR);//打开 test 设备节点
    //······
    //iotrl(fd,CMD_TEST1,1); //向驱动程序写入数据
    ioctl(fd,CMD_TEST2,&val);//如果第二个参数为 read，则读取内核空间传递向用户空间传递的值
    //val 是一个 int 类型的变量，那么 &val 的数据类型就是 int *（整型指针）。
    printf("val is %d\n",val); //读取驱动中的val
}
```
驱动程序：
file_operation 中的ioctl体现：
函数原型：`long (*unlocked_ioctl) (struct file *file , unsigned int cmd, unsigned long arg);
```c
#define CMD_TEST0 _IO('L',0)
#define CMD_TEST1 _IOW('L',1,int)
#define CMD_TEST2 _IOR('L',2,int)

struct file_operations cdev_test_fops = {
.owner = THIS_MODULE, //将 owner 字段指向本模块，可以避免在模块的操作正在被使用时卸载该模块
.unlocked_ioctl = cdev_test_ioctl,
};

static long cdev_test_ioctl(struct file *file, unsigned int cmd, unsigned long arg) //
{
int val;//定义 int 类型向应用空间传递的变量 val
switch(cmd){
case CMD_TEST0:  //需要进行的操作
printk("this is CMD_TEST0\n");
break;
case CMD_TEST1:
printk("this is CMD_TEST1\n");
printk("arg is %ld\n",arg);//打印应用空间传递来的 arg 参数

break;
case CMD_TEST2:
val = 1;//将要传递的变量 val 赋值为 1
printk("this is CMD_TEST2\n");
if(copy_to_user((int *)arg,&val,sizeof(val)) != 0){//通过 copy_to_user 向用户空间传递数据
//把内核空间 &val 里的数据（比如数字 100），跨越时空结界，搬运到用户空间地址为 0x7FF001 的内存里去。
printk("copy_to_user error \n");
}
break;
default:
break;
}
return 0;
}
```

地址传参
```c 应用程序
//使用地址传递多个参数，可读可写
struct ioctl_arg{
    int a;
    int b;
    int c;
}
int main(int argc,char *argv[]){
    struct ioctl_arg test;
    test.a = 1;
    test.b = 2;
    test.c = 3;
    ioctl(fd,CMD_TEST1,&test);
}

```
```c 驱动程序
struct ioctl_arg{
    int a;
    int b;
    int c;
}
static long cdev_test_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct ioctl_arg test;
    switch(cmd){
    //传递多个参数--地址传参
        if(copy_from_user(&test,(int *)arg,sizeof(test))!=0){  ////将用户空间传递来的 arg 赋值给 test
        printk("copy_from_user error\n");
        }   
    }
}
```
细节：
```c
// &test 是个指针，但传入 ioctl 时，它被强行转成了一串纯数字：0x7ffeefbff5bc
ioctl(fd,CMD_TEST1,&test);
static long cdev_test_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    copy_from_user(&test,(int *)arg,sizeof(test)); //如果直接写 arg,译器会立刻大喊：  
    // “报告！arg 是个 long 类型的整数(0x7ffeefbff5bc)，你怎么能把整数直接当指针用呢？报错！”
    //copy_from_user 函数不认纯数字，它的第二个参数必须是一个明确的源地址指针（const void __user *from）。
    //无论你写 (int *)arg、(char *)arg 还是 (struct test_t *)arg，它们转化出来的目标内存起始地址都是一模一样的（都是 0x7ffeefbff5bc）。
}
```
