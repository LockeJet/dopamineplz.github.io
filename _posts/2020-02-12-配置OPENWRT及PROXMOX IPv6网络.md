---
layout:     post
title:     配置 OPENWRT 及 PROXMOX IPv6 网络
subtitle:   
date:       2020-02-12
author:     Jet
header-img: img/post-bg-2015.jpg
catalog: true
tags: 
- openwrt
- proxmox
- ipv6
- ddns
---
# 配置 OPENWRT 及 PROXMOX IPv6 网络

## OpenWrt IPv6配置
OpenWrt版本：18.06。

广州电信现在默认是IPv4和IPv6双公网，其中分发了/64的前缀，貌似太小了（好像一般是下发/60前缀的？）。

WAN口没有240e开头的地址，但是LAN口有，去设置完善一下：

### Interfaces -> LAN
#### Common Configuration -> General Setup

>IPv6 assignment length = 64

>IPv6 assignment hint = 7777

>IPv6 suffix = random

后缀设置了随机，主要原因是地址不容易被人发现；eui64方式可以猜到MAC地址，::1之类话，知道分发的前缀可以很容易找到路由器LAN口地址。

#### DHCP Server -> IPv6 Settings

>Router Advertisement-Service = server Mode

>DHCPv6-Service = disabled

>NDP-Proxy = relay mode

>Always announce default router = [x]


RA和DHCPv6可以设置为hybrid模式，但是这样设置之后，PVE的一个接口可以得到8个240e开头的IPv6地址，太多了；

那就：

RA设置为server模式；

DHCPv6设置之后某一模式后，ANDROID手机得不到IPv6公网地址；既然SLAAC比较普遍，干脆禁用DHCPv6模式。

/etc/config/network总体配置如下：
```
config globals globals
        option ula_prefix fd00:db80::/48

config interface 'loopback'
        option ifname 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

config interface 'lan'
        option type 'bridge'
        option proto 'static'
        option ipaddr '192.168.8.1'
        option netmask '255.255.255.0'
        option ifname 'eth0 eth1 eth2'
        option ip6assign '64'
        option ip6ifaceid 'random'
        option ip6hint '7777'

config interface 'wan'
        option proto 'pppoe'
        option username '0@163.gd'
        option password 'P'
        option ipv6 'auto'
        option ifname 'eth3'
```
引用官网的说明解释一下：

>ip6assign: Prefix size used for assigned prefix to the interface (e.g. 64 will assign /64-prefixes)

>ip6hint: Subprefix ID to be used if available (e.g. 1234 with an ip6assign of 64 will assign prefixes of the form …:1234::/64)

>ip6class: Filter for prefix classes to accept on this interface (e.g. wan6 will only assign prefixes with class “wan6” but not e.g. “local”)



### 初步试验结果
上述配置完了之后，ANDROID手机和WINDOWS电脑都可以得到IPv6地址(以fd00:db80:0:7777开头的ULA地址和以240e::/64开头的GLA地址）。

WINDOWS PC上运行PING -6 IPV6.BAIDU.COM，正常；

用浏览器访问IPV6.BAIDU.COM也正常。吐槽一下，这个网站居然没有采用https！

但是Proxmox死活不能得到全球单播地址。

/etc/network/Interfaces改变了无数次，后来GOOLE之，终于找到原因，主要是sysctl配置的问题。
PVE配置如下文。

## Proxmox 配置
/etc/network/interfaces文件内容如下
```
auto lo
iface lo inet loopback
iface lo inet6 loopback

iface enp1s0 inet manual
iface enp1s0 inet6 manual

auto vmbr1
allow-hotplug vmbr1
iface vmbr1 inet manual
iface vmbr1 inet6 manual
        bridge-ports enp2s0
        bridge-stp off
        bridge-fd 0
iface vmbr1 inet6 static
        address  fd00::241
        netmask  128
        gateway fd00::1
iface vmbr1 inet static
        address  192.168.8.241
        netmask  24
        broadcast 192.168.8.255
        gateway  192.168.8.1
```
上述文件中，配置了vmbr1 inet6为manual模式，即意味着vmbr1的地址可以由上级路由器和宿主机“操作”或者“运算”得到合适合适的IPv6地址。

另外还配置了IPv4和IPv6的静态地址，以便于ssh登陆维护。

补充说明： 一个接口配置多个IP，可以用上面的方式；不必用落伍的的别名方式（如vmbr1:0之类）

/etc/sysctl.conf配置如下：
```
##
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
##
net.ipv6.conf.all.use_tempaddr = 2
net.ipv6.conf.default.use_tempaddr = 2
## IPv6 Support for docker
net.ipv6.conf.default.forwarding = 1
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.default.accept_ra=2
net.ipv6.conf.all.accept_ra = 2
net.ipv6.conf.default.proxy_ndp = 1
net.ipv6.conf.all.proxy_ndp = 1
## IPv4 support for proxies
net.ipv4.ip_forward = 1
```
主要是因为之前配置了转发（forwarding=1），所以现在路由广告的接受级别为2才行（accept_ra=2），这样转发和无状态地址自动配置（SLAAC）才能同时生效。

参见：
https://askubuntu.com/questions/997888/sysctl-p-return-net-ipv6-conf-all-accept-ra-2


### 试验结果

重启之后，PROXMOX 也可以得到合适的IPv6地址了。

有了IPv6地址，相当于内网主机就裸奔了，这就需要防火墙来保护一下。 这里借用网上配置好的/etc/ip6tables.rules文件修改一下。这里不提。

## DDNS 配置

手头有一个可用的域名，于是也设置了DDNS。

用了NewFuture的程序，参见：

https://github.com/NewFuture/DDNS

安装：
```
pip install --upgrade pip
pip install ddns
```
json文件（/opt/ddns/config-ipv6.json):
```
{
  "$schema": "https://ddns.newfuture.cc/schema/v2.json",
  "debug": false,
  "dns": "cloudflare",
  "id": "abc@outlook.com",
  "index4": "default",
  "index6": "default",
  "ipv6": [
    "nas.abc.xyz"
  ],
  "proxy": null,
  "token": "ff"
}
```
脚本文件（/opt/ddns/updateDDNS-ipv6.sh）
```
#!/bin/bash
/usr/local/bin/ddns -c /opt/ddns/config-ipv6.json
```
自动执行脚本/etc/network/if-up.d/ddns:
```
#!/bin/sh
#sleep 30s
#cd /opt/ddns
/opt/ddns/updateDDNS-ipv6.sh > /var/log/updateDDNS-ipv6.log 2>&1
```
记得上述sh文件都要设置为可执行格式。

这样，重启之后，ip -6 addr show vmbr1 显示的一个scope global temporary dynamic 类的IPv6地址在CF里面更新过来了；当然，按照说明，脚本也会自动每5分钟更新一次。





