# WINDOWS无法上GITHUB的解决办法

今天在公司在WINDOWS系统上，用VPN连接至家里，可以GOOGLE，但是无法上GIBHUB；手机却可以。很奇怪。

搜索了一圈，发现要把GITHUB相关的域名放到C:\Windows\System32\drivers\etc\host文件里面去，类似这样：
```
192.30.253.113 github.com
192.30.253.113 github.com
192.30.253.118 gist.github.com
192.30.253.119 gist.github.com
```
我看了一下电脑上的这个文件，发现有类似内容，随手把这些内容删除掉，发现可以上GITHUB了。这就更奇怪了。

