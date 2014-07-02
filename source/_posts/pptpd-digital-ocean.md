title: 搭建VPN在Digital Ocean机器上
date: 2014-07-02
tags: 
 - VPN
 - PPTP
 - Digital-Ocean
comment: true
categories:
 - tools
 - vpn
---

Digital Ocean是美国的虚拟主机提供商，$5一个月有1TB的流量

`这里讲的VPN是采用PPTP搭建`

## Digital Ocean机房的选择

我选择的是`San Francisco`的机房，一阵子用下来速度还是挺快的

曾经试用过`Singapore`的机房，通过`traceroute`发现会绕道美国，导致速度大大降低，不知联通的情况是什么样

```
$ traceroute 128.199.160.88
traceroute to 128.199.160.88 (128.199.160.88), 64 hops max, 52 byte packets
 1  broadcom.home (192.168.1.1)  3.848 ms  1.048 ms  1.009 ms
 2  124.74.56.15 (124.74.56.15)  3.397 ms  3.119 ms  3.383 ms
 3  124.74.57.153 (124.74.57.153)  6.609 ms  5.195 ms  3.989 ms
 4  124.74.209.13 (124.74.209.13)  6.366 ms  6.515 ms  8.199 ms
 5  202.101.63.234 (202.101.63.234)  8.541 ms  6.415 ms  8.109 ms
 6  202.97.33.74 (202.97.33.74)  5.659 ms  13.756 ms  6.561 ms
 7  202.97.33.254 (202.97.33.254)  8.264 ms  8.042 ms  7.182 ms
 8  202.97.33.5 (202.97.33.5)  7.568 ms  45.149 ms  65.558 ms
 9  202.97.5.66 (202.97.5.66)  71.700 ms *  76.652 ms
10  as-6.r21.sngpsi02.sg.bb.gin.ntt.net (129.250.5.157)  173.067 ms  172.575 ms  173.195 ms
11  ae-6.r00.sngpsi02.sg.bb.gin.ntt.net (129.250.6.105)  170.559 ms  168.120 ms  170.110 ms
12  * 116.51.27.190 (116.51.27.190)  292.490 ms  232.413 ms
13  ...
```

其中的`129.250.5.157`是美国的IP

所以，推荐大家选择`San Francisco`的机房

## 安装pptpd

``` bash
$ yum -y install pptpd
```


## 配置pptpd

### 配置pptpd.conf文件

``` bash
$ vi /etc/pptpd.conf
```

到文件的最后面，输入
```
localip 192.168.0.1
remoteip 192.168.1.1-238,192.168.1.245
```

保存退出即可

### 配置帐号信息

``` bash
$ vi /etc/ppp/chap-secrets
```

在最后一行添加帐号信息，如下示例

```
sekiyika    pptpd   123456  *
```

设置IP地址为通配符，能让这个帐号在任何IP下均可用，保存退出即可

`Caution: 不要用示例中如此简单的IP` 

### 添加DNS

``` bash
$ vi /etc/ppp/options.pptpd
```

找到`ms-dns`，在下面添加两行DNS，如下所示:

```
# Network and Routing

# If pppd is acting as a server for Microsoft Windows clients, this
# option allows pppd to supply one or two DNS (Domain Name Server)
# addresses to the clients.  The first instance of this option
# specifies the primary DNS address; the second instance (if given)
# specifies the secondary DNS address.
#ms-dns 10.0.0.1
#ms-dns 10.0.0.2

ms-dns 8.8.8.8
ms-dns 8.8.4.4
```
保存退出

### 启动PPTPD

``` bash
$ service pptpd restart
Shutting down pptpd:                                       [  OK  ]
Starting pptpd:                                            [  OK  ]
Warning: a pptpd restart does not terminate existing 
connections, so new connections may be assigned the same IP 
address and cause unexpected results.  Use restart-kill to 
destroy existing connections during a restart.
```

可以通过下面的命令观察pptpd是否启动成功
``` bash
$ netstat -nlp | grep pptpd
tcp        0      0 0.0.0.0:1723                0.0.0.0:*                   LISTEN      1222/pptpd   
```

## 服务器

在配置好PPTPD之后，还需要对服务器做一点点小小的修改，follow me

### forward

``` bash
$ vi /etc/sysctl.conf
```

修改`ip_forward = 0`为`1`，如下所示

```
net.ipv4.ip_forward = 1
```

保存退出之后，运行如下命令

``` bash
$ sysctl -p
```

### 创建NAT规则

运行如下命令

``` bash
$ iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE && iptables-save
```

到这里，PPTPD就已经配置好了

## 连接VPN

[Mac的用户请戳我](https://www.astrill.com/knowledge-base/69/PPTP---How-to-configure-PPTP-with-built-in-client-on-Mac-OS-X.html)
[windows的用户请戳我](http://blog.fens.me/vpn-pptp-client-win7/)


Thanks.

