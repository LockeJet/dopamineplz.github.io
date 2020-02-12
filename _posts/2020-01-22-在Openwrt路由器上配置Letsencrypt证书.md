---
layout:     post
title:     Openwrt 上配置 Letsencrypt 证书
subtitle:  复制证书方法
date:       2020-1-22
author:     Jet
header-img: img/post-bg-2015.jpg
catalog: true
tags: 
- Letsencrypt
- OPENWRT
---
# Openwrt 上配置 Letsencrypt 证书

0. 设置环境变量
```
export DOMAIN=example.com
```
1. 将证书从AWS复制到Openwrt路由器的/root/.acme.sh/${DOMAIN}下。
2. 设置证书文件
```
cd /root/.acme.sh/${DOMAIN}
ln -sf cert3.pem cert.pem
ln -sf privkey3.pem key.pem
```
3. 修改/etc/config/uhttpd文件
```
uci set uhttpd.main.redirect_https=1
uci set uhttpd.main.key="$(pwd)/key.pem"
uci set uhttpd.main.cert="$(pwd)/cert.pem"
uci commit uhttpd
```

4. 重启uhttpd即可
/etc/init.d/uhttpd restart
