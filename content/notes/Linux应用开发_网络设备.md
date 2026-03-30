---
title: "Linux应用开发 网络设备" # <--- 修改这一行
date: "2026-3-16T17:15:29+08:00"
draft: false
tags: ["Linux", "应用开发"]
location: "西安"
---


参考链接：

<a href="https://wiki.dshanpi.org/docs/DshanPi-A1/AppBasic/Network" target="_blank" rel="noopener noreferrer">韦东山百问网网络通信 </a>

<a href="https://www.cnblogs.com/duzouzhe/archive/2009/06/19/1506699.html" target="_blank" rel="noopener noreferrer">Linux网络编程入门 (转载) </a>


## 为什么使用htons()等?
字节序 (Byte Order)：

- 电脑内部存数字的方式有“大端”和“小端”之分。

- 网络规定：在网线上传输数据必须用“大端”。

- htons 的意思是 host to network short（把主机里的数字转成网络通用的格式）。


| 函数名称 |功能描述 | 数据类型 | 转换方向 |
|----|---------|-------|-------|
|htonl|32位主机字节序网络字节序|uint32_t|主机网络|
|htons|16位主机字节序网络字节序|uint16 t|主机一网络|
|ntohl|32位网络字节序主机字节序|uint32 t|网络一主机|
|ntohs|16位网络字节序主机字节序|uint16 t|网络一主机|



## 为什么使用：inet_pton()？

地址转换函数
- 人类习惯看 127.0.0.1，但机器只认 4 个字节的二进制。192.168.1.1 （点分十进制串）在组包的时候IP地址需要转成整型数据
- inet_pton 的意思是 presentation to numeric（把字符串转成数字）。

int inet_pton(int af,const char *src,void *dst); 把ip地址转换成整型数据
const char *inet_ntop(int af,const void *src,char *dst,socklen_t size)；把整型数据转换成ip地址


## UDP

如果说 **TCP** 是在打**电话**（必须先拨号、接通、确保对方在听），那么 **UDP**（User Datagram Protocol）就像是在**寄明信片**。

---

### 1. 什么是 UDP？

UDP 是一种“不可靠”的传输协议。这里的“不可靠”不是贬义词，而是指它**只负责发，不负责查收**。

* **没有连接**：不需要 `connect` 和 `accept`，直接把数据往对方 IP 和端口上甩。
* **不保证顺序**：先发的明信片不一定先到。
* **不重传**：丢了就丢了，UDP 不会像 TCP 那样发现丢包了自动重发。
* **速度极快**：没有握手，没有确认机制，由于开销小，延迟极低。

常用于视频
---

### 2. UDP 的 C 语言实现

在 UDP 编程中，核心函数不再是 `read/write`，而是 `sendto` 和 `recvfrom`（因为没有连接，每次发信都得写上对方地址）。


### 客户端 (udp_client.c)
客户端写完之后可以使用网络调试助手检测一下是否能正常发送数据，首先验证Ubuntu和本机是否能ping通。协议类型选择UDP服务器，ip选择本机ip
```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(8888);
    servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");

    char *msg = "你好，UDP！";

    // 1. 发送数据 (必须指定目标地址)
    sendto(sockfd, msg, strlen(msg), 0, (const struct sockaddr *) &servaddr, sizeof(servaddr));
    printf("消息已发出\n");

    close(sockfd);
    return 0;
}
```

### 服务端 (udp_server.c)
将服务器代码在开发板上运行，客户端代码在Ubuntu上运行，看服务器上是否能接受到客户端发来的数据

```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    // 1. 创建 Socket (注意这里用的是 SOCK_DGRAM，表示数据报/UDP)
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in servaddr, cliaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = INADDR_ANY;
    servaddr.sin_port = htons(8888);

    // 2. 绑定端口
    bind(sockfd, (const struct sockaddr *)&servaddr, sizeof(servaddr));

    printf("UDP 服务端已启动，等待消息...\n");

    char buffer[1024];
    socklen_t len = sizeof(cliaddr);

    // 3. 接收数据 (recvfrom 会告诉你谁发给你的)
    int n = recvfrom(sockfd, buffer, 1024, 0, (struct sockaddr *) &cliaddr, &len);
    buffer[n] = '\0';
    
    printf("收到来自 %s 的消息: %s\n", inet_ntoa(cliaddr.sin_addr), buffer);

    // 4. 关闭
    close(sockfd);
    return 0;
}
```



## TCP
TCP 协议是最常用的，因为它像打电话，必须先接通（三次握手），保证数据不丢。

服务端（Server）：等电话的人
Socket: 买个电话机。

Bind: 固定一个电话号码（绑定 IP 和端口）。

Listen: 坐在电话边等它响。

Accept: 电话响了，按下接听键。

Read/Write: 说话、听话。

客户端 (Client)：拨号的人
Socket: 买个电话机。

Connect: 拨对方的号码。

Write/Read: 说话、听话。


总的来说网络程序是由两个部分组成的--客户端和服务器端.它们的建立步骤一般是:
服务器端
socket-->bind（把fd,ip和端口号绑定起来）-->listen（启动监测）-->accept（接受连接）  send/recv  发收数据

客户端
socket-->connect（建立连接）     send/recv  发收数据

## 客户端代码 (client.c)
客户端的任务很简单：找到地址，拨通电话。

```C
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    // 1. 创建 Socket
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 指定要连接的服务端地址
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(8888);
    
    // 将字符串形式的 IP 转换为二进制格式
    inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr);

    // 3. 连接服务端 (拨号)
    connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    // 4. 发送数据 (说话)
    char *hello = "Hello from C Client!";
    send(sock, hello, strlen(hello), 0);   //也可以使用send,sendto,recvfrom，recv不必非得是Write/Read
    printf("消息已发送\n");

    // 5. 关闭
    close(sock);
    return 0;
}
```


## 服务端代码 (server.c)
服务端的任务是：申请端口，死守阵地，等待连接。
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h> // 处理网络地址的库

int main() {
    // 1. 创建 Socket (买电话机)
    // AF_INET: 使用 IPv4; SOCK_STREAM: 使用 TCP 协议
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 准备地址和端口 (准备号码)
    struct sockaddr_in address;
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY; // 监听所有网卡的 IP
    address.sin_port = htons(8888);      // 监听 8888 端口 (htons 负责字节序转换)

    // 3. 绑定 (把号码安装到电话机上)
    bind(server_fd, (struct sockaddr *)&address, sizeof(address));

    // 4. 监听 (坐在电话边等它响)
    listen(server_fd, 3);
    printf("服务端正在 8888 端口等待连接...\n");

    // 5. 接受连接 (有人拨号进来了，接听)
    int addrlen = sizeof(address);
    int new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen);
    printf("连接成功！\n");

    // 6. 读取数据 (听对方说话)
    char buffer[1024] = {0};
    read(new_socket, buffer, 1024);
    printf("收到消息: %s\n", buffer);

    // 7. 关闭 (挂断)
    close(new_socket);
    close(server_fd);
    return 0;
}
```



---

## 3. TCP vs UDP：深度对比

为了帮你决定什么时候用哪个，我们看这张对比表：

| 特性 | TCP (传输控制协议) | UDP (用户数据报协议) |
| :--- | :--- | :--- |
| **连接性** | 面向连接（先握手） | 无连接（直接发） |
| **可靠性** | **可靠**（保证不丢、不重、有序） | **不可靠**（可能丢包、乱序） |
| **传输速度** | 较慢（有确认、拥塞控制机制） | **极快**（像冲锋枪扫射） |
| **资源消耗** | 大（需要维持连接状态） | 小 |
| **数据边界** | 流式（像自来水，没头没尾） | 数据报（像邮包，边界清晰） |
| **应用场景** | 网页(HTTP)、文件传输(FTP)、邮件 | **实时视频、直播、在线游戏、DNS** |

---

## 4. 通俗理解：如何选型？

* **选 TCP 的情况**：如果你发的数据**一个字节都不能错**。比如你下载一个软件，丢一个字节程序就运行不了；或者你发个红包，数据错了钱就对不上了。
* **选 UDP 的情况**：如果你追求**实时性**，且能容忍少量丢失。比如你在玩《王者荣耀》或者吃鸡，网速慢时你宁愿画面“瞬移”一下，也不希望它卡住 5 秒去等那个过时的数据包。或者你看直播，丢掉一两个像素点你根本发现不了。

---

## 5. 常见面试题：UDP 既然不可靠，为什么还要用它？

**回答：**
1.  **效率**：它省去了建立连接的 3 次握手和断开的 4 次挥手。
2.  **实时性**：在网络差的时候，TCP 会因为重传导致严重的排队延迟（队头阻塞），而 UDP 丢了就跳过，保证了最新的数据能最快到达。
3.  **支持广播/组播**：TCP 只能点对点，UDP 可以一个人发给全网段的人。


