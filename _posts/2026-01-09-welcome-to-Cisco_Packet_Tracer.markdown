---
layout:	post
title:	"Welcome to Cisco-Packet-Tracer!!!"
data:	2026-01-09 14:00:08 +0800
categories: jekyll update
---
[计算机网络实验概述 — 计算机网络实验指导书 - 2023春季 | 哈工大（深圳） v1.0 文档](https://comp-network-2023.readthedocs.io/zh/latest/lab0/index.html)

## 实验课题一

![ Vlan 和接口模式 ]({{ site.baseurl }}/assets/images/posts/Vlan_20260109.png)

| 设备 | IP地址       | 子网掩码      |
| ---- | ------------ | ------------- |
| PC0  | 192.168.2.10 | 255.255.255.0 |
| PC1  | 192.168.2.11 | 255.255.255.0 |
| PC2  | 192.168.3.10 | 255.255.255.0 |
| PC3  | 192.168.3.11 | 255.255.255.0 |

配置vlan2

```
# switch 0 
enable 
configure terminal
hostname S0
no ip domain-lookup
vlan 2
exit
interface f0/11
switchport access vlan 2
exit
interface f0/1
switchport access vlan 2
exit
exit
```

```
# switch 1
enable 
configure terminal
hostname S1
no ip domain-lookup
vlan 2
exit
interface f0/11
switchport access vlan 2
exit
interface f0/1
switchport access vlan 2
exit
exit
```

```
show vlan
```

配置vlan3

```
# S0
enable
configure terminal
vlan 3
exit
interface f0/13
switchport access vlan 3
exit
exit
```

```
# s1
enable
configure terminal
vlan 3
exit
interface f0/12
switchport access vlan 3
exit
exit
```

由于 vlan 技术的隔离，网络 vlan3 无法连通

```
# 配置 s0 和 s1 的 trunk 端口
# s0
interface f0/1
na switchport access vlan
switchport mode trunk
switchport trunk allowed valan 2,3
exit
exit
```

```
# s1
interface f0/1
na switchport access vlan
switchport mode trunk
switchport trunk allowed valan 2,3
exit
exit
```

```
show valan
此时 vlan2 和 vlan3 互相隔离且跨 vlan 可以访问
```

## 实验课题二

![ Rip 路由配置 ]({{ site.baseurl }}/assets/images/posts/Rip_20260109.png)

| 设备        | IP地址       | 子网掩码        |
| ----------- | ------------ | --------------- |
| R0-f0/0     | 192.168.1.1  | 255.255.255.0   |
| R0-f0/1     | 192.168.2.1  | 255.255.255.0   |
| R0-lookback | 1.1.1.1      | 255.255.255.255 |
| R1-f0/0     | 192.168.1.2  | 255.255.255.0   |
| R1-f0/1     | 192.168.3.1  | 255.255.255.0   |
| PC0         | 192.168.2.11 | 255.255.255.0   |
| PC1         | 192.168.3.11 | 255.255.255.0   |
| PC2         | 192.168.3.12 | 255.255.255.0   |

```
# 配置路由器 Route 0
enable
configure terminal
hostname R0
no ip domain-lookup
inerface 0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
exit
interface f0/1
ip address 192.168.2.1 255.255.255.0
no shutdown
exit
interface lookback 1
ip address 1.1.1.1 255.255.255.255
exit
exit

```

```
# 配置路由器 Route 1
enable
configure terminal
hostname R1
no ip domain-lookup
inerface 0/0
ip address 192.168.1.2 255.255.255.0
no shutdown
exit
interface f0/1
ip address 192.168.3.1 255.255.255.0
no shutdown
exit
exit
```

```
# 设置网关
```

由于没有设置 RIP 路由协议，PC0 和 PC1 之间还没办法连通

```
# R0
router rip
version 2
network 192.168.1.0
network 192.168.2.0
network 1.1.1.0
no auto-summary
exit
exit
```

```
# R1
router rip
version 2
network 192.168.1.0
network 192.168.3.0
no auto-summary
exit
exit

```

配置RIP路由协议后，PC0 就能访问到 PC1
仿真的时候抓包 RIP 、ARP 、ICMP 数据包
实验断开 lookback 1 的回环地址的情况

## 实验课题三

![ NAT组网 ]({{ site.baseurl }}/assets/images/posts/Nat_20260109.png)

| 设备    | IP地址        | 子网掩码      |
| ------- | ------------- | ------------- |
| G0-g0/0 | 202.169.10.2  | 255.255.255.0 |
| G0-g0/1 | 192.168.10.2  | 255.255.255.0 |
| PC0     | 192.168.10.11 | 255.255.255.0 |
| PC1     | 202.169.10.11 | 255.255.255.0 |
| SERVER0 | 192.168.10.12 | 255.255.255.0 |
| SERVER1 | 202.169.10.12 | 255.255.255.0 |

```
# 配置路由器 R0 的端口 ip 和内外网的通路
# 静态 NAT ，只能手动添加内网映射
enable
configure terminal
hostname 
no ip domain-lookup
interface g0/0
ip address 202.169.10.2 255.255.255.0
no shutdown
exit
interface g0/1
ip address 192.168.10.2 255.255.255.0
no shutdown
exit

interface g0/0
ip nat outside
exit
interface g0/1
ip nat inside
exit

ip nat inside source static 192.168.10.10 202.169.10.2
exit

```

```
# 添加 pool 动态 NAT
configure terminal
no ip nat inside source static 192.168.10.10 202.169.10.2
interface g0/0
ip nat outside
exit
interface g0/1
ip nat inside 
exit

access-list 1 permit 192.168.10.0 0.0.0.255
ip nat pool aaa 202.169.10.5 202.169.10.10 netmask 255.255.255.0
ip nat inside source list 1 pool aaa
exit
```

```
动态 NAT 的端口复用
configure terminal
no access-list 1 permit 192.168.10.0 0.0.0.255
no ip nat inside source list 1 pool aaa
no ip nat pool aaa 202.169.10.5 202.169.10.10 netmask 255.255.255.0

interface g0/0
ip nat outside
exit
interface g0/1
ip nat inside 
exit
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface g0/0 overload
exit
```

```
# 配置 NAT server 测试 FTP 服务
configure terminal
no access-list 1 permit 192.168.10.0 0.0.0.255
no ip nat inside sourse list 1 interface g0/0 overload

interface g0/0
ip nat outside 
exit
interface g0/1
ip nat inside 
exit

ip nat inside source static tcp 192.168.10.10 21 202.169.10.10 21
exit
```
