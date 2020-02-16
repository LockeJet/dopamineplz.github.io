# Debian VM上搭建v2ray透明网关

水文一篇，仅当记录备查。

## 基本网络配置
/etc/network/interfaces
```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
address 192.168.8.119
netmask 255.255.255.0
gateway 192.168.8.1
dns-nameserver 192.168.8.1
iface ens18 inet6 auto
```
/etc/resolv.conf
```
nameserver 192.168.8.1
nameserver 127.0.0.1
```
/etc/sysctl.conf
```
##
net.ipv4.ip_forward=1
##

##
net.ipv6.conf.default.forwarding = 1
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.default.accept_ra=2
net.ipv6.conf.all.accept_ra = 2
net.ipv6.conf.default.proxy_ndp = 1
net.ipv6.conf.all.proxy_ndp = 1
##

##
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
##
```
虚拟机网卡，貌似没有什么隐私，直接用SLAAC就行了。

## v2ray
下载并安装：
```
bash <(curl -L -s https://install.direct/go.sh)
```
配置/etc/v2ray/config.json
```
{
  "log": {
    "error": "/var/log/v2ray/error.log",
    "access": "/var/log/v2ray/access.log",
    "loglevel": "warning"
        }, //log */

  "inbounds": [
    {
      "tag":"transparent",
      "port": 12345,
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp,udp",
        "followRedirect": true
      }, //settings */
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      }, //sniffing */
      "streamSettings": {
        "sockopt": {
          "tproxy": "tproxy" // 透明代理使用 TPROXY 方式
        }
      } //streamSettings */
    }, // inbounds[0] */
    {
      "port": 1089,
      "protocol": "socks", // 入口协议为 SOCKS 5
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth"
      }
    } // inbounds[1] */
  ], // inbounds */

  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vmess", // 代理服务器
      "settings": {
       "vnext": [
          {
            "address": "abc.xyz",
            "port": 443,
            "users": [
              {
                "id": "<>",
                "alterId": 16
              } //users */
            ] //users */
          } //vnext[0] */
          ] //vnext */
      }, //settings */
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "wsSettings": {
          "path": "/v-ws"
        }, //wsSettings */
         "sockopt": {
          "mark": 255
          } //sockopt */
        }, //streamSettings */
      "mux": {
        "enabled": true
      } //mux */
    }, // outbounds[0]: tproxy
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "UseIP"
      },
      "streamSettings": {
        "sockopt": {
          "mark": 255
        }
      }
    }, // outbounds[1]: direct
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "http"
        }
      }
    }, // outbounds[2]: blackhole
    {
      "tag": "dns-out",
      "protocol": "dns",
      "streamSettings": {
        "sockopt": {
          "mark": 255
        }
      } //streamSettings
    } //outbounds[3]: dns-out
  ], //outbounds */

  "dns": {
    "servers": [
      "8.8.8.8", // 非中中国大陆域名使用 Google 的 DNS
      "1.1.1.1", // 非中中国大陆域名使用 Cloudflare 的 DNS(备用)
      "114.114.114.114", // 114 的 DNS (备用)
      {
        "address": "223.5.5.5", //中国大陆域名使用阿里的 DNS
        "port": 53,
        "domains": [
          "geosite:cn",
          "ntp.org",   // NTP 服务器
          "abc.xyz" // VPS 的域名
        ]
      }
    ]
  }, //dns */

  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [
      { // 劫持 53 端口 UDP 流量，使用 V2Ray 的 DNS
        "type": "field",
        "inboundTag": [
          "transparent"
        ],
        "port": 53,
        "network": "udp",
        "outboundTag": "dns-out"
      },
      { // 直连 123 端口 UDP 流量（NTP 协议）
        "type": "field",
        "inboundTag": [
          "transparent"
        ],
        "port": 123,
        "network": "udp",
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": [
          // 设置 DNS 配置中的国内 DNS 服务器地址直连，以达到 DNS 分流目的
          "223.5.5.5",
          "114.114.114.114"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": [
          // 设置 DNS 配置中的国内 DNS 服务器地址走代理，以达到 DNS 分流目的
          "8.8.8.8",
          "1.1.1.1"
        ],
        "outboundTag": "proxy" // 改为你自己代理的出站 tag
      },
      { // 广告拦截
        "type": "field",
        "domain": [
          "geosite:category-ads-all"
        ],
        "outboundTag": "block"
      },
      { // BT 流量直连
        "type": "field",
        "protocol":["bittorrent"],
        "outboundTag": "direct"
      },
      { // 直连中国大陆主流网站 ip 和 保留 ip
        "type": "field",
        "ip": [
          "geoip:private",
          "geoip:cn"
        ],
        "outboundTag": "direct"
      },
      { // 直连中国大陆主流网站域名
        "type": "field",
        "domain": [
          "geosite:cn"
        ],
        "outboundTag": "direct"
      }
    ] //rules */
  } //routing */
}
```

## 路由和防火墙
/opt/v2ray_tproxy-rules.sh：
```
# 设置策略路由
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100

# 代理局域网设备
iptables -t mangle -N V2RAY
iptables -t mangle -A V2RAY -d 127.0.0.1/32 -j RETURN
iptables -t mangle -A V2RAY -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A V2RAY -d 192.168.8.0/24 -p tcp -j RETURN # 直连局域网，避免 V2Ray 无法启动时无法连网关的 SSH，如果你配置的是其他网段（如 10.x.x.x 等），则修改成自己的
iptables -t mangle -A V2RAY -d 192.168.8.0/24 -p udp ! --dport 53 -j RETURN # 直连局域网，53 端口除外（因为要使用 V2Ray 的
iptables -t mangle -A V2RAY -p udp -j TPROXY --on-port 12345 --tproxy-mark 1 # 给 UDP 打标记 1，转发至 12345 端口
iptables -t mangle -A V2RAY -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1 # 给 TCP 打标记 1，转发至 12345 端口
iptables -t mangle -A PREROUTING -j V2RAY # 应用规则

# 代理网关本机
iptables -t mangle -N V2RAY_MASK
iptables -t mangle -A V2RAY_MASK -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_MASK -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A V2RAY_MASK -d 192.168.8.0/24 -p tcp -j RETURN # 直连局域网
iptables -t mangle -A V2RAY_MASK -d 192.168.8.0/24 -p udp ! --dport 53 -j RETURN # 直连局域网，53 端口除外（因为要使用 V2Ray 的 DNS）
iptables -t mangle -A V2RAY_MASK -j RETURN -m mark --mark 0xff    # 直连 SO_MARK 为 0xff 的流量(0xff 是 16 进制数，数值上等同与上面V2Ray 配置的 255)，此规则目的是避免代理本机(网关)流量出现回环问题
iptables -t mangle -A V2RAY_MASK -p udp -j MARK --set-mark 1   # 给 UDP 打标记,重路由
iptables -t mangle -A V2RAY_MASK -p tcp -j MARK --set-mark 1   # 给 TCP 打标记，重路由
iptables -t mangle -A OUTPUT -j V2RAY_MASK # 应用规则
```
/etc/rc.local:
```
#!/bin/bash
/opt/v2ray_tproxy-rules.sh
```

## 试验结果
Debian VM本机和OpenWrt VM客户端均可以得到公网IPv6地址；curl -4 google.com正常。



