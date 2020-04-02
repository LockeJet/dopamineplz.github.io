OpenWrt路由器上配置ocserv服务器

参考：

https://liuc.tk:88/blog/post/admin/d9be0ae00e0a

有一台IPAD，原来打算是看pdf书籍用（买前生产力），结果变变为小孩上油管看动画片专用（买后爱奇艺）。

原来IPAD上有安装Anyconnect客户端，直接连到VPS服务器。

可是有时候上B站看片，客户端只能走全局，很不爽。

后来参(PIAO)考(QIE)了上述的网文，在路由器上搭建ocserv服务器，IPAD通过客户端连接到ocserv服务器，通过路由器实现透明分流科学上网。

## ocserv安装与准备
在OpenWrt路由器上，安装ocserv：
```
opkg update
opkg install ocserv
```
准备一些链接：
```
cd /etc/ocserv
ln -s pki/ca.tmpl ca.tmpl
ln -s pki/server.tmpl server.tmpl
#ln -s ca.pem ca-cert.pem
#cp pki/client.tmpl client.tmpl
#cp ocserv.conf.template ocserv.conf
ln -s /etc/ssl/certs/${DOMAIN}/fullchain.pem /etc/ocserv/server-cert.pem
ln -s /etc/ssl/certs/${DOMAIN}/privkey.pem /etc/ocserv/server-key.pem
```
定义要用到的变量：

```
# CA证书
CA_CN="OpenWrt CA"
# 服务器证书
SRV_CN=${DOMAIN}
# 用户证书
USER=yy
CLI_CN=${USER}
CLI_UNIT=ow
USER_PREFIX=ow
USER_KEY=${USER}-key.pem
USER_CERT=${USER}-cert.pem
USER_P12=${USER_PREFIX}-${USER}.p12
```

生成模板文件：

- CA模板
```
cat << EOF > /etc/ocserv/pki/ca.tmpl
cn=${CA_CN}
expiration_days=-1
serial=1
ca
cert_signing_key
EOF
```
- 服务器模板
```
cat << EOF > /etc/ocserv/pki/server.tmpl
cn=${SRV_CN}
serial=2
expiration_days=-1
signing_key
encryption_key
EOF
```
- 用户模板
```
cat << EOF > /etc/ocserv/pki/client-${USER}.tmpl
cn = ${CLI_CN}
unit =${CLI_UNIT}
expiration_days = 3650
signing_key
tls_www_client
EOF
```

## 证书制作
- CA证书(ocserv安装后自带有ca证书，可以不重新制作; 若重新制作了，则用户证书相应地也要重新制作一遍)
```
certtool --generate-privkey --outfile /etc/ocserv/ca-key.pem
certtool --template /etc/ocserv/ca.tmpl --generate-self-signed --load-privkey /etc/ocserv/ca-key.pem  --outfile /etc/ocserv/ca-cert.pem
```

- 服务器证书
```
certtool --generate-privkey --outfile /etc/ocserv/server-key.pem
certtool --generate-certificate \
--template /etc/ocserv/pki/server.tmpl \
--load-privkey /etc/ocserv/server-key.pem \
--load-ca-certificate /etc/ocserv/ca-cert.pem \
--load-ca-privkey /etc/ocserv/ca-key.pem \
--outfile /etc/ocserv/server-cert.pem
```  

- 用户证书
生成cert和key：
```
certtool --generate-privkey --outfile ${USER_KEY}
certtool --generate-certificate --template /etc/ocserv/pki/client-${USER}.tmpl \
--load-privkey $USER_KEY \
--load-ca-certificate ca-cert.pem \
--load-ca-privkey ca-key.pem \
--outfile $USER_CERT
```
进行格式转换：
```
certtool --to-p12 --pkcs-cipher 3des-pkcs12 --load-privkey $USER_KEY --load-certificate $USER_CERT --outfile ${USER_P12} --outder
```

- 最后将证书传到WEB服务器，IPAD访问服务器安装描述文件。

## 修改服务文件
自带的服务文件/etc/init.d/ocserv似乎有问题，修改一下:

cat << 'EOF' > /etc/init.d/ocserv-alt.sh
#!/bin/sh /etc/rc.common

START=90
STOP=90

OCSERV_PID_FILE=/var/run/ocserv.pid
OCSERV_CONF=/etc/ocserv/ocserv.conf
OCSERV_CONF_LINK=/var/etc/ocserv.conf

start() {
killall -0 ocserv 2> /dev/null
OCSERV_IS_NOT_RUNNING=$?
if [ $OCSERV_IS_NOT_RUNNING -eq 1 ]
then
rm -f ${OCSERV_CONF_LINK}
ln -s ${OCSERV_CONF} ${OCSERV_CONF_LINK}
logger \"$0: Starting ocserv ...\"
echo "starting ocserv ..."
/usr/sbin/ocserv -c ${OCSERV_CONF_LINK}
killall -0 -w ocserv 2> /dev/null
else
logger \"$0: ocserv already running: $(pidof ocserv)\"
echo "ocserv already running."
fi
}

stop() {
killall -0 ocserv 2> /dev/null
OCSERV_IS_NOT_RUNNING=$?
if [ $OCSERV_IS_NOT_RUNNING -eq 1 ]
then
logger  \"$0: ocserv not running.\"
echo "ocserv not running."
else
killall -9 ocserv 2> /dev/null
rm -f ${OCSERV_CONF_LINK}
rm -f ${OCSERV_PID_FILE}
logger  \"$0: ocserv stopped.\"
echo "ocserv stopped."
# delay needed to avoid restart failure
sleep 3
fi
}

reload() {
rm -f ${OCSERV_CONF_LINK}
ln -s ${OCSERV_CONF} ${OCSERV_CONF_LINK}
        /usr/bin/occtl show status >/dev/null 2>&1
        if test $? != 0;then
                start
        else
                /usr/bin/occtl reload
        fi
}
EOF

```
注意cat命令中，EOF要带单引号。如果EOF不带单引号，后面输入的所有变量都会被替换为变量取值（如${OCSERV_IS_NOT_RUNNING}都被替换为0，如果上一个命令不出错的话）。这个坑我爬了一个晚上。

完了之后，让服务自动启动(可以不用，因为后面配置了热插拔脚本)：
```
/etc/init.d/ocserv-alt.sh enable
```

## 配置防火墙

```
cat << EOF >> /etc/firewall.user
# ocserv
iptables -I INPUT -p tcp --dport 9443 -j ACCEPT
iptables -I INPUT -p udp --dport 9443 -j ACCEPT
iptables -t nat -I POSTROUTING -s 172.30.0.0/16 -o pppoe-wan -j MASQUERADE
#iptables -t nat -I POSTROUTING -s 172.30.0.0/16 -o eth0 -j MASQUERADE
#iptables -I FORWARD -i vpns+ -s 192.168.8.0/24 -j ACCEPT
#iptables -I INPUT -i vpns+ -s 192.168.8.0/24 -j ACCEPT
iptables -I FORWARD -i vpns+ -s 172.30.0.0/16 -j ACCEPT
iptables -I INPUT -i vpns+ -s 172.30.0.0/16 -j ACCEPT
EOF
```
172.30.0.0/16 是 ocserv VPN 网段，192.168.8.0/24 是我家局域网网段。

按参考博文，应该设置：

> iptables -I FORWARD -i vpns+ -s 192.168.8.0/24 -j ACCEPT
> iptables -I INPUT -i vpns+ -s 192.168.8.0/24 -j ACCEPT

一开始我是这样配置的，试验发现可以连接到家里的路由器，但是只能用用浏览器上国内网，无法上海外网，也无法上知乎、微信。

后来把 -s 后面的改成 VPN 网段，上海外网和国内外都没有问题。

## 配置 ocserv

配置文件：
```
cat << 'EOF' > /etc/ocserv/ocserv.conf
listen-host-is-dyndns = true
auth = "certificate"
max-clients = 16
max-same-clients = 10
tcp-port = 9443
udp-port = 9443
keepalive = 32400
dpd = 240
mobile-dpd = 1800
try-mtu-discovery = true
server-cert = /etc/ocserv/server-cert.pem
server-key = /etc/ocserv/server-key.pem
ca-cert = /etc/ocserv/ca-cert.pem
cert-user-oid = 2.5.4.3
#cert-group-oid = 2.5.4.11
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"
auth-timeout = 40
mobile-idle-timeout
cookie-timeout = 86400000
rekey-time = 86400000
rekey-method = ssl
connect-script = /etc/ocserv/connect-script
disconnect-script = /etc/ocserv/connect-script
use-utmp = true
use-occtl = true
pid-file = /var/run/ocserv.pid
socket-file = /var/run/ocserv-socket
run-as-user = ocserv
run-as-group = ocserv
net-priority = 5
cgroup = "cpuset,cpu:test"
device = vpns
default-domain = ${DOMAIN}
ipv4-network = 172.30.0.0/16
ipv4-netmask = 255.255.0.0
dns = 172.30.0.1
ping-leases = false
output-buffer = 10
route-add-cmd = "ip route add %{R} dev %{D}"
route-del-cmd = "ip route delete %{R} dev %{D}"
cisco-client-compat =true
custom-header = "X-DTLS-MTU: 1200"
custom-header = "X-CSTP-MTU: 1200"
EOF
```

主要修改:

> listen-host-is-dyndns = true

- 配置证书登录

> auth = "certificate"

- 指定端口（443端口被电信封了，只能修改一下）

> tcp-port = 9443
> udp-port = 9443

- cert-user-oid

> cert-user-oid = 2.5.4.3

- 接口名称：

> device = vpns

- 默认域名：

> default-domain = ${DOMAIN}

- VPN网段：
> ipv4-network = 172.30.0.0/16
> ipv4-netmask = 255.255.0.0

- DNS：

> dns = 172.30.0.1

连接脚本：

```
cat << 'EOF' >  /etc/ocserv/connect-script
#!/bin/sh
#echo $USERNAME : $REASON : $DEVICE
case "$REASON" in
  connect)
echo $USERNAME "connected" >> /tmp/ocservinfo
date >> /tmp/testocserv
echo $REASON $USERNAME $DEVICE $IP_LOCAL $IP_REMOTE $IP_REAL >> /tmp/ocservinfo
	;;
  disconnect)
echo $USERNAME "disconnected" >> /tmp/ocservinfo
date >> /tmp/ocservinfo
	;;
esac
exit
EOF
```

更改权限：

```
chmod +x /etc/ocserv/connect-script
```

## 配置热插拔脚本

```
cat << 'EOF' > /etc/hotplug.d/iface/75-ocserv
#!/bin/sh

if [ $ACTION = "ifup" -a $INTERFACE = "wan" ];then
sleep 15s
ip -4 addr del 172.30.0.1/16 dev eth0
ip -4 addr add 172.30.0.1/16 dev eth0
/etc/init.d/ocserv-alt.sh restart
fi
EOF
```

这样如果WAN口断线或者启动时，ocserv会自动重启。


## 监控脚本

生成监控脚本，定期检测，保证ocserv意外退出后及时重启。

- 生成脚本
```
cat << 'EOF' > /etc/ocserv-watch.sh
#!/bin/sh
killall -0 ocserv 2> /dev/null
if [ $? != 0 ]
then
logger \"$0: Starting ocserv ...\"
if [ ! -e /var/etc/ocserv.conf ]
then
ln -s /etc/ocserv/ocserv.conf /var/etc/ocserv.conf
fi
/usr/sbin/ocserv -c /var/etc/ocserv.conf
else
logger \"$0: ocserv already running: \"
fi
EOF

- 修改权限
```
chmod +x /etc/ocserv-watch.sh
```

- 计划任务
```
cat << 'EOF' >> /etc/crontabs/root

*/5 * * * * /etc/ocserv-watch.sh
EOF
```

## 使用

执行：
```
/etc/init.d/ocserv-alt.sh start
/etc/init.d/firewall restart
```
在IPAD上导入证书，就可以愉快的上境内境外网了。

注：前提是路由器上设置了透明代理。

关于证书：

只要以下三则一致,VPN连接时就不会提示 “证书与服务器名称不符”:

- 服务器证书里的 cn
- VPN服务端配置文件ocserv.conf文件里的 default-domain
- VPN客户端设置里的 服务器地址

但是我的证书有点问题（多个域名的证书），暂时还没有时间去搞。凑合着用吧。
