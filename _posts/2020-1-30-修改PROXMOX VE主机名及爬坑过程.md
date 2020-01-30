---
layout:     post
title:      修改PROXMOX主机名及其爬坑过程
subtitle:   
date:       2020-1-30
author:     Jet
header-img: img/post-bg-2015.jpg
catalog: true
tags: 
- Proxmox
- node
- hostname
---

# 修改PROXMOX VE主机名及其爬坑过程

有两台PROXMOX VE主机，一台作为软路由，另外一台作为NAS。这里台都是默认的pve主机名，迫于强迫症，决定修改一下主机名。结果跳进大坑了，搞了一个晚上+一个早上才搞完。

参见 https://pve.proxmox.com/wiki/Renaming_a_PVE_node

## 修改相关文件
```
root@nas:~# cat /etc/hostname
nas
root@nas:~# cat /etc/hosts
127.0.0.1 localhost.localdomain localhost
192.168.8.254 nas.lan nas
<...>
```
WEBUI里面出现这个：
>hostname lookup 'pve' failed - failed to get address info for: pve: No address associated with hostname (500)

因此还需要以下步骤：
```
root@nas:~# cat /etc/postfix/main.cf
# See /usr/share/postfix/main.cf.dist for a commented, more complete version

#myhostname=pve.lan
myhostname=nas.lan

<...>

```

## 移动相关文件
1. nodes文件
```
cd /etc/pve/nodes
mv pve/qemu-server/*.conf nas/qemu-server/
```
2. 移动/var/lib/rrdcached/db下的nodes/storage文件/目录
```
mv /var/lib/rrdcached/db/pve2-nodes/pve  /var/lib/rrdcached/db/pve2-nodes/nas
mv /var/lib/rrdcached/db/pve2-storage/pve  /var/lib/rrdcached/db/pve2-storage/nas
```
变成这样：

```
root@nas:/var/lib/rrdcached/db# tree
.
├── pve2-node
│   ├── nas
│   └── pve
├── pve2-storage
│   ├── nas
│   │   ├── local
│   │   └── local-lvm
│   └── pve
└── pve2-vm
    ├── 100
 <...>

5 directories, 29 files
```

VM多了个2011，不知道怎么来的。


## 占用空间太大
执行期间发现占用空间太大，无法编辑、复制或移动文件，以下是排查过程。
```
root@nas:/var/log# du -h .
48K     ./vzdump
4.0K    ./ceph
<...>
4.0K    ./apt
22G     .
```
很奇怪，有一个22G的文件，但就是不显示。
上面的命令输出省略了一部分。为了确认当时没有漏看，重新做一下：
```
root@nas:/var/lib/rrdcached/db# du -h /var/log | grep ss-redir.log
root@nas:/var/lib/rrdcached/db# ls -l /var/log | grep ssr-redir.log
-rw-r--r-- 1 root     root         223 Jan 30 11:09 ssr-redir.log
```
很神奇。BUG？后来再试，原来不是：
```
root@pve:/var/lib/rrdcached/db/pve2-storage# du -ah /var/log | grep ss-redir.log
153M    /var/log/ss-redir.log
```
原来要加-a选项！！！

当时这样把大文件找出来了：
```
root@nas:/var/log# find ./ -size +1000M
./ssr-redir.log
root@nas:/var/log# rm ssr-redir.log
root@nas:/var/log# df -hl
Filesystem            Size  Used Avail Use% Mounted on
udev                  3.9G     0  3.9G   0% /dev
tmpfs                 793M   79M  714M  10% /run
/dev/mapper/pve-root   28G   28G     0 100% /
<...>
root@nas:/var/log# hostnamectl set-hostname pve
Could not set property: Failed to set static hostname: No space left on device

```
重启后才正常，但不一会又产生20G的ssr-redir.log文件：
```
root@nas:/etc/pve/nodes# cat /var/log/ssr-redir.log
ERROR: accept: Too many open files
<...Endless Errors...>
```
临时禁用之
```
root@nas:/etc/pve/nodes# systemctl disable ss-tproxy
```
这样才不会产生大的日志文件。ss-redicr问题暂时不处理，临时禁用。



