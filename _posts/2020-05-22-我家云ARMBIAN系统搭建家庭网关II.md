---
layout:     post
title:      我家云ARMBIAN系统搭建家庭网关II
subtitle:   科学上网
date:       2020-5-22
author:     Jet
header-img: img/post-bg-2015.jpg
catalog: true
tags: 
- ARMBIAN
- DNS
- CLASH
---
# 2020-05-22-我家云ARMBIAN系统搭建家庭网关II

## DNS 安装与配置

- DNSMASQ 分流
采用CHINADNS。
```
git clone https://github.com/felixonmars/dnsmasq-china-list

mkdir /opt/dnsmasq-china-list
cd /opt/dnsmasq-china-list

ln -s /opt/dnsmasq-china-list/accelerated-domains.china.conf /etc/dnsmasq.d/accelerated-domains.china.conf

#可选
ln -s /opt/dnsmasq-china-list/google.china.conf /etc/dnsmasq.d/google.china.conf

#可选
ln -s /opt/dnsmasq-china-list/apple.china.conf /etc/dnsmasq.d/apple.china.conf

#防止在使用无效域名解析时被运营商劫持
ln -s /opt/dnsmasq-china-list/bogus-nxdomain.china.conf /etc/dnsmasq.d/bogus-nxdomain.china.conf
```
- DNSCRYPT 代理

这里采用dnscrpy-proxy。
```
DNSCRYPT_PROXY_URL=https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.0.42/dnscrypt-proxy-linux_arm64-2.0.42.tar.gz
wget ${DNSCRYPT_PROXY_URL} -O dnscrypt-proxy-linux.tar.gz
tar xzf dnscrypt-proxy-linux.tar.gz
mv linux-arm64 /opt/dnscrypt-proxy
cd /opt/dnscrypt-proxy

cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml
```

修改dnscrypt-proxy.toml：

- 服务的监听端口为5311
> listen_addresses = ['127.0.0.1:5311']

- 启用 DNS OVER HTTPS 
> doh

将DNSCrypt-Proxy安装为服务并启动之：
```
./dnscrypt-proxy --service install
./dnscrypt-proxy --service start
```

将DNSMASQ的上游服务器由114.114.114.114等替换为127.0.0.1：5311，即修改/etc/dnsmasq.d/dns.conf：

> server=127.0.0.1#5311

重启服务后，试验：
```
systemctl dnsmasq restart
nslookup google.com
Server:         127.0.0.1
Address:        127.0.0.1#53

Non-authoritative answer:
Name:   google.com
Address: 172.217.19.238
```
可以看到已经无误的得到DNS地址了。


## CLASH 安装与配置

```
wget https://github.com/Dreamacro/clash/releases/download/v0.19.0/clash-linux-armv8-v0.19.0.gz

gzip -d clash-linux-armv8-v0.19.0.gz

install -Dm755 clash-linux-armv8-v0.19.0 /usr/local/bin/clash

# 把 clash 配置目录放在 /etc 目录下
mkdir /etc/clash/

# 下载Contry.mmdb文件
wget https://geolite.clash.dev/Country.mmdb
```
配置 CLASH
参考了其它P文:

https://github.com/V2RaySSR/Tools/blob/master/clash.yaml

/etc/clash/config.yaml修改如下：
- DNS 部分
禁用clash自带的DNS，因为上面采用了DNSMASQ监听53端口：
```
dns:
  enable: false
  #enable: true
  #ipv6: false
  #listen: 0.0.0.0:53
  ##enhanced-mode: fake-ip
  enhanced-mode: redir-host
```
- #配置开始部分

> - name: "V2ray节点的主机测试"
> - name: "Trojan节点的主机测试"

服务文件生成与配置：
```
cat << EOF > /etc/systemd/system/clash.service
[Unit]
Description=clash service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/clash -d /etc/clash/
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl enable clash.service
systemctl start clash.service
```

测试下 socks 代理是否可用:
```
curl --socks5 localhost:7891 google.com
```
出现
> 301 Moved
说明已经可以利用掉盘云的socks5代理科学上网了。

最后还要配置防火墙：

```
cat << EOF > /opt/clash-ipt.sh
#!/bin/sh

iptables -t nat -N CLASH

# 私有 ip 流量不转发
# 设置的 fake-ip 请注意检查这里
iptables -t nat -A CLASH -d 0.0.0.0/8 -j RETURN
iptables -t nat -A CLASH -d 10.0.0.0/8 -j RETURN
iptables -t nat -A CLASH -d 127.0.0.0/8 -j RETURN
iptables -t nat -A CLASH -d 169.254.0.0/16 -j RETURN
iptables -t nat -A CLASH -d 172.16.0.0/12 -j RETURN
#iptables -t nat -A CLASH -d 192.168.0.0/16 -j RETURN
#iptables -t nat -A CLASH -d 192.168.8.0/24 -j RETURN
iptables -t nat -A CLASH -d 192.168.6.0/24 -j RETURN
iptables -t nat -A CLASH -d 224.0.0.0/4 -j RETURN
iptables -t nat -A CLASH -d 240.0.0.0/4 -j RETURN

iptables -t nat -A CLASH -p tcp -j REDIRECT --to-ports 7892
iptables -t nat -A PREROUTING -p tcp -j CLASH

EOF

chmod +x /opt/clash-ipt.sh

# 加入自动启用

cat << EOF >> /etc/ppp/ip-up
/opt/up.d/ipt-rules.sh > /var/log/ipt-rules.sh 2>&1
EOF

```
这样内网主机也可以科学上网了。



## 参考资料
https://ksana410.github.io/2019/08/22/linux-router-05-Dnsmasq-advanced-configuration/
https://a-wing.top/network/2020/02/22/bypass_gateway-1_clash.html
https://www.v2rayssr.com/clashxx.html

