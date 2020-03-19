---
layout:     post
title:      在服务器上搭建下载利器QBITTORRENT容器
subtitle:   ""
date:       2020-03-17
author:     Jet
header-img: img/post-bg-2015.jpg
catalog:    true
tags: 
- DOCKER
- QBITTORENT
---

# 服务器上搭建下载利器ARIA2容器及其WEBUI

前段时间配置ARIA2的容器老是失败，主要是配置https老是出错，经常说权限问题。一气之下不搞了。可是过几天手又痒痒，重新找了一个，这几天终搞定了。这里记录一下。

参考：
- https://p3terx.com/archives/docker-aria2-pro.html
- https://p3terx.com/archives/aria2-frontend-ariang-tutorial.html

## Openwrt 防火墙配置

在/etc/config/firewall增加以下重定向规则：

```
config redirect
        option target 'DNAT'
        option src 'wan'
        option dest 'lan'
        option proto 'tcp udp'
        option dest_ip '192.168.8.254'
        option name 'PVE-QBITTORRENT-DK'
        option src_dport '6881'
        option dest_port '6881'

config redirect
        option target 'DNAT'
        option src 'wan'
        option dest 'lan'
        option proto 'tcp udp'
        option dest_ip '192.168.8.254'
        option name 'PVE-NGINX'
        option src_dport '8080'
        option dest_port '8080'

config redirect
        option target 'DNAT'
        option src 'wan'
        option dest 'lan'
        option proto 'tcp udp'
        option dest_ip '192.168.8.254'
        option name 'PVE-QBITTORRENT-DK-WEB-SSL'
        option src_dport '44383'
        option dest_port '44383'

```

完成后重新启动防火墙。

## 内网服务器防火墙配置

放行 44383、6881端口：

```
# aria2...
iptables -A INPUT -p tcp -m multiport --dport 6800:6809,6810:6819,6880:6889,6860:6869 -m state --state NEW -j ACCEPT
iptables -A INPUT -p udp -m multiport --dport 6800:6809,6810:6819,6880:6889,6860:6869 -m state --state NEW -j ACCEPT
```

## DOCKER 配置

- 配置 docker-compose.yml

```
mkdir qbittorrent
cd qbittorrent

cat << EOF > docker-compose.yml

version: "2"

services:
  qbittorrent:
    image: linuxserver/qbittorrent
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - UMASK_SET=022
      #-  WEBUI_PORT=44373
      #-  WEBUI_PORT=8073
      #- WEBUI_PORT=44373
      - WEBUI_PORT=8080
      #- DOMAIN=https://${DOMAIN}

    volumes:
      - ./config:/config
      - ./downloads:/downloads

    network_mode: "host"

    ports:
      - "6881:6881"
      - "6881:6881/udp"
      #- "44373:8073"
      - "8080:8080"
      #- "44373:44373"
      #- "8073:8073"

    restart: unless-stopped


- 启动 

```
docker-compose up -d
```

可以看到启动成功了。



## NGINIX

- 配置 NGINX 服务

```
cat << 'EOF' > /etc/nginx/sites-available/docker-qbittorrent

server {
        listen 8083;
        server_name ${DOMAIN};
        #location / {
        #proxy_pass http://127.0.0.1:8073;
        #proxy_set_header   X-Forwarded-Host  $host:$server_port;
        #proxy_hide_header  Referer;
        #proxy_hide_header  Origin;
        #proxy_set_header   Referer      '';
        #proxy_set_header   Origin       '';
        #}
        return 301 https://$host$request_uri;

}

server {
    listen 44383 ssl http2;
    #listen 44383 ssl;
    server_name ${DOMAIN};
    #ssl on;
    ssl_certificate /etc/ssl/certs/${DOMAIN}.pem;
    ssl_certificate_key /etc/ssl/certs/${DOMAIN}.key;
    location / {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:8080;
        #proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Host $host:$server_port;
    }
}

```

- 启动服务

```
ln -s /etc/nginx/sites-available/docker-qbittorrent /etc/nginx/sites-enabled/docker-qbittorrent
servic nginx restart
```

## 使用

在浏览器输入：

> https://${DOMAIN}:44383

> https://$<IP>:44383

就可以进入QBITTORENT的前端，设置相关参数，就可以下载了。




