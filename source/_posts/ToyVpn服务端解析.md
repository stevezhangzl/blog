---
title: ToyVpn服务端解析
date: 2023-06-22 05:29:54
tags:
---



> 有多种方法可以使用该程序。这里我们只给出一个最简单场景的示例。假设 Linux 机器有一个eth0 上的公共 IPv4 地址。请尝试以下步骤并调整必要时的参数。
> 
>
> `1.Enable IP forwarding `  
>
> ```shell
> echo 1 > /proc/sys/net/ipv4/ip_forward
> ```
>
> 注：让路由器能执行数据报的转发操作，主机一般只能发送和接收数据报，修改这个值，可以使主机能进行数据报转发功能
> 
>
> `2.Pick a range of private addresses and perform NAT over eth0.`
>
> ```shell
> iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth0 -j MASQUERADE
> ```
>
> - `iptables`: 是 Linux 系统中用于配置防火墙规则的命令行工具。
> - `-t nat`: 指定操作的表为 `nat` 表，该表用于处理网络地址转换（Network Address Translation，NAT）相关的规则。
> - `-A POSTROUTING`: 表示将规则添加到 `POSTROUTING` 链中，`POSTROUTING` 链是在数据包离开系统之前进行处理的链。
> - `-s 10.0.0.0/8`: 匹配源地址为 `10.0.0.0/8` 的数据包，这个地址范围表示私有IP地址段，通常用于内部网络。
> - `-o eth0`: 匹配出站接口为 `eth0` 的数据包，即数据包将通过 `eth0` 接口发送出去。
> - `-j MASQUERADE`: 将匹配到的数据包进行源地址转换（Source NAT），使用服务器的外部IP地址替换数据包的源IP地址。这样做的目的是让数据包在互联网上能够正常发送和返回，实现内部网络与外部网络的通信。
>
> 总体来说，这条 iptables 规则用于在 ToyVPN 服务器端进行网络地址转换，将来自 `10.0.0.0/8` 内部网络的数据包在发送到外部网络之前，将源IP地址修改为服务器的外部IP地址，以实现内部网络与外部网络的通信。
> 
>
> `3.Create a TUN interface.`
>
> ```shell
> ip tuntap add dev tun0 mode tun
> ```
>
> - `ip`: 是用于配置和管理网络参数的命令行工具。
> - `tuntap`: 是 `ip` 命令的子命令，用于创建和管理虚拟网络接口（TUN/TAP 接口）。
> - `add`: 表示添加一个新的虚拟网络接口。
> - `dev tun0`: 指定虚拟网络接口的名称为 `tun0`，你可以根据需要自行指定其他名称。
> - `mode tun`: 指定虚拟网络接口的模式为 `tun`。`tun` 是一种虚拟网络接口模式，它在网络层（第三层）工作，用于 IP 数据包的路由。
>
> 总结起来，这条命令用于在 Linux 系统上创建一个名为 `tun0` 的虚拟网络接口，并将其设置为 TUN 模式，以便在网络层上进行 IP 数据包的路由。创建虚拟网络接口可以用于实现各种网络隧道和 VPN 连接。
>
> 






