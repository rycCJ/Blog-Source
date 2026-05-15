---
title: 'Linux应用开发 命令' # <--- 修改这一行
date: "2026-04-09T21:41:23+08:00"
draft: false
tags: ["Linux", "应用开发"]
location: "西安"
---


## 网络配置命令
### ifconfig 命令
查看和配置网络接口的命令。
```bash
# 查看所有活动的网络接口
ifconfig

# 查看特定接口
ifconfig eth0

# 启用或禁用网络接口
sudo ifconfig eth0 up
sudo ifconfig eth0 down

# 配置IP地址和子网掩码
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0

# 配置广播地址
sudo ifconfig eth0 192.168.1.100 broadcast 192.168.1.255
```
### ip 命令
查看和配置网络接口的命令。
```bash
# 显示所有网络接口信息
ip addr show
# 或简写为
ip a

# 显示特定接口信息
ip addr show dev eth0

# 添加IP地址到接口
sudo ip addr add 192.168.1.100/24 dev eth0

# 删除接口上的IP地址
sudo ip addr del 192.168.1.100/24 dev eth0

# 启用或禁用网络接口
sudo ip link set eth0 up
sudo ip link set eth0 down

# 显示网络统计信息
ip -s link show eth0
```
### route 路由管理
路由表决定了数据包的转发路径。
route命令
```bash
# 显示路由表
route -n

# 添加默认网关
sudo route add default gw 192.168.1.1

# 添加到特定网络的路由
sudo route add -net 10.0.0.0/8 gw 192.168.1.254

# 删除路由
sudo route del -net 10.0.0.0/8
```
ip route 命令
```bash
# 显示路由表
ip route show
# 或简写为
ip r

# 添加默认路由
sudo ip route add default via 192.168.1.1

# 添加到特定网络的路由
sudo ip route add 10.0.0.0/8 via 192.168.1.254

# 删除路由
sudo ip route del 10.0.0.0/8

```

### netstat 命令
用于显示网络连接、路由表、接口等统计的信息。
```bash
# 显示所有活动的网络连接
netstat -a

# 显示监听端口
netstat -l

# 显示TCP连接
netstat -t

# 显示UDP连接
netstat -u

# 显示所有监听的TCP和UDP端口，并显示PID/程序名
sudo netstat -tulpn

# 显示路由表
netstat -r
```

## 网络配置文件
### 网络接口配置
networkmanager 命令
```bash
# 列出所有连接
nmcli connection show

# 添加新的静态IP连接
nmcli connection add con-name "static-eth0" ifname eth0 type ethernet ip4 192.168.1.100/24 gw4 192.168.1.1

# 修改连接的DNS服务器
nmcli connection modify "static-eth0" ipv4.dns "8.8.8.8 8.8.4.4"

# 启用连接
nmcli connection up "static-eth0"
```
### DNS 配置
### 主机名配置
### 
## 网络工具诊断
### ping连通信测试
### traceroute路由跟踪
### wget和curl下载工具   
### ssh 远程连接