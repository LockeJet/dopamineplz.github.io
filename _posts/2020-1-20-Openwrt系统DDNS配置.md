---
layout:     post
title:     Openwrt路由器DDNS配
subtitle:   
date:       2020-1-20
author:     Jet
header-img: img/post-bg-2015.jpg
catalog: true
tags: 
- DDNS
- IPv6
- OPENWRT
---
# Openwrt路由器DDNS配置

## 获取域名在CLOUDFLARE上的DNS_ID

``` 编辑get_cf_dns_id.sh文件：
#!/bin/sh
CF_API_KEY=f6*85
CF_ZONE_ID=66*f3
CF_DNS_ID=
EMAIL=abc@outlook.com
curl -X GET "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records" \
-H "X-Auth-Email:${EMAIL}" \
-H "X-Auth-Key:${CF_API_KEY}" \
-H "Content-Type: application/json" 
```
加上权限执行：
```
chmod +x get_cf_dns_id.sh
./get_cf_dns_id.sh
```
找到SHELL输出result {...}里面域名对应的id，即为CF_DNS_ID，将域名的两个id（一个A记录，一个AAAA记录）填入后文的/etc/updateDDNS.sh对应的CF_DNS_ID。

## 编辑/etc/updateDDNS.sh
```
#!/bin/sh
CF_API_KEY=f6*85
CF_ZONE_ID=6*f3
CF_DNS_ID=85*6a
EMAIL=abc@outlook.com

ROUTER_NETWORK_DEVICE=pppoe-wan
TEMP_FILE_PATH=/tmp/cloudflare-ddns/
DNS_RECORD=example.com

echo "================================================================================"
mkdir -p ${TEMP_FILE_PATH}

curl -k -X GET "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records/${CF_DNS_ID}" \
-H "X-Auth-Email:${EMAIL}" \
-H "X-Auth-Key:${CF_API_KEY}" \
-H "Content-Type: application/json"  | awk -F '"' '{print $18}'>${TEMP_FILE_PATH}/current_resolving.txt

ifconfig $ROUTER_NETWORK_DEVICE | awk -F'[ ]+|:' '/inet /{print $4}'>${TEMP_FILE_PATH}/current_ip.txt

if [ "$(cat ${TEMP_FILE_PATH}/current_ip.txt)" == "$(cat ${TEMP_FILE_PATH}/current_resolving.txt)" ];
then
echo "IPv4 not changed. DNS A record not changed."
#exit 1
else
CURRENT_IP="$(cat ${TEMP_FILE_PATH}/current_ip.txt)"
curl -k -X PUT "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records/${CF_DNS_ID}" \
-H "X-Auth-Email:${EMAIL}" \
-H "X-Auth-Key:${CF_API_KEY}" \
-H "Content-Type: application/json" \
--data '{"type":"A","name":"'$DNS_RECORD'","content":"'$CURRENT_IP'","ttl":1,"proxied":false}'
echo -e "\r\n--------\r\n"
echo "IPv4 changed. DNS A record updated."
fi

ROUTER_NETWORK_DEVICE=br-lan
#TEMP_FILE_PATH=/tmp/cloudflare-ddns/
DNS_RECORD=example.com
CF_DNS_ID=4a*4c
#mkdir -p ${TEMP_FILE_PATH}

echo "================================================================================"

curl -k -X GET "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records/${CF_DNS_ID}" \
-H "X-Auth-Email:${EMAIL}" \
-H "X-Auth-Key:${CF_API_KEY}" \
-H "Content-Type: application/json" \
 | awk -F '"' '{print $18}'>${TEMP_FILE_PATH}/current_resolving_ipv6.txt

ip -6 addr show dev br-lan  | awk '/inet6 / {print $2}' | head -n 1 | cut -d/ -f 1 > ${TEMP_FILE_PATH}/current_ipv6.txt

if [ "$(cat ${TEMP_FILE_PATH}/current_ipv6.txt)" == "$(cat ${TEMP_FILE_PATH}/current_resolving_ipv6.txt)" ];

```
上述脚本获取获取路由器pppoe-wan的IPv4地址和br-lan接口的IPv6地址，并检查是否有变化；如果有那么就调用CLOUDFLARE API更新域名对应的A记录和AAAA记录。

## 让DDNS自动执行
编辑//etc/hotplug.d/iface/99-wan
```
#!/bin/sh

if [ $ACTION = "ifup" -a $INTERFACE = "wan" ]
then
# Send a mail to inform the new WAN IP
/etc/sendMail.sh
# Update DDDS
/etc/updateDDNS.sh

# Restart zerotier
ifconfig down ztukus2z4l
ifconfig up ztukus2z4l

# Config eth1 to allow login to ONU
#ip addr add 192.168.1.2/24 dev eth1:1
ip addr add 192.168.1.2/24 dev eth1.2
#ip addr add 192.168.1.2/24 dev eth1
fi

#if [ $ACTION = "ifup" -a $INTERFACE = "wan2" ]
#then    /etc/sendMail.sh
#        /etc/updateDDNS.sh
#        ip addr add 192.168.2.2/24 dev eth0.3
#
#fi

```
