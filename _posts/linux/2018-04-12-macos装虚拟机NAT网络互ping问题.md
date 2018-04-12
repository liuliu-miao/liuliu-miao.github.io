---
layout: post
title: macos装虚拟机NAT网络互ping问题
categories: [Linux]
description: 虚拟机NAT网络
keywords: macos装虚拟机, 虚拟机NAT网络
---


### 问题描述
* macOS系统装虚拟机,虚拟机与宿主机互ping不通


### 解决方案
* 修改宿主机的VMware网络设置
* 编辑文件目录 
> /Library/Preferences/VMware Fusion/networking
> 内容如下
``` bash
VERSION=1,0
# 此处禁用DHCP模式
answer VNET_1_DHCP no
answer VNET_1_DHCP_CFG_HASH A1C3DC05C0F343C380B049B0A45A95DD63494961
answer VNET_1_HOSTONLY_NETMASK 255.255.255.0
answer VNET_1_HOSTONLY_SUBNET 192.168.181.0
# 宿主机的ip地址
answer VNET_1_VIRTUAL_ADAPTER yes
answer VNET_1_VIRTUAL_ADAPTER_ADDR 10.10.1.67
# 禁用DHCP模式
answer VNET_8_DHCP no
answer VNET_8_DHCP_CFG_HASH 5197E3A254D370D2E0B1CCD8B3F59319D3A67453
# 子网掩码
answer VNET_8_HOSTONLY_NETMASK 255.255.255.0
# 子网网段 192.168.1.0-192.168.255.0
answer VNET_8_HOSTONLY_SUBNET 192.168.1.0
answer VNET_8_NAT yes
answer VNET_8_VIRTUAL_ADAPTER yes
``` 


