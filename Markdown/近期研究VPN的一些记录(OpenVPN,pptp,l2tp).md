近期研究VPN的一些记录(OpenVPN,pptp,l2tp)
===================


近期由于一些需要（特别是上Google），研究了下在VPS上搭建VPN服务器的方法。其中遇到一些坑，顺带记下来以备下次使用。

其实在有VPS的情况下，还有另外一种替代方案。那就是在路由器上直接ssh隧道+sock5代理+使用[autossh](http://www.harding.motd.ca/autossh/)自动重连+使用[polipo](http://www.pps.univ-paris-diderot.fr/~jch/software/polipo/)作HTTP代理+[PAC文件](https://calomel.org/proxy_auto_config.html)自动代理切换。实现，最终我在家里就是这么搞得，而且这样对网络结构没有其他影响。

[我的PAC文件](https://github.com/owt5008137/OWenT.Articles/blob/master/Resource/2014/proxy.pac)

----------

以上那些都不重要，话不多说直接开始VPN的部分吧

OpenVPN
-------------
OpenVPN的话网上有很多教程啦，很容易配，过程挺繁琐。大致过程是

1. 如果使用tun（第三层协议）的话检查tun设备(/dev/net/tun)
2. 生成CA证书、服务器证书、客户端证书。（可以用**easy-rsa**生成，比较简单点）
2. 配置防火墙端口开放和**路由转发** （可以拷贝openvpn的sample里的**firewall.sh**来用，注意没有内网网络设备的话把eth1相关的东西注释掉）
3. OpenVPN配置

**要注意的一点是其实OpenVPN示例里有很多配好的带注释的配置，不需要照很多教程里的完全自己写iptables和server配置的**

```bash
#!/bin/sh
# CentOS 6 x86_64 下的命令，其他系统类似

# 检查tun （如果出错说内核不支持tun）
modinfo tun;

# EPEL源
rpm -ivh "http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm";

# 安装软件包
yum install -y easy-rsa openvpn;
mkdir -p /etc/openvpn;
cp -f /usr/share/doc/openvpn-*/sample/sample-config-files/* /etc/openvpn ;
cp -rf /usr/share/easy-rsa /etc/openvpn/ ;

# 手动生成证书 ...
# 配置firewall.sh(防火墙和路由转发，注意不要把你开放了的端口又屏蔽了)
# 配置openvpn-startup.sh里要启动的VPN配置文件（最后几行）

# 启动openvpn
cd /etc/openvpn && ./openvpn-startup.sh
# 关闭openvpn
cd /etc/openvpn && ./openvpn-shutdown.sh
```

**建议尝试配置的过程中先使用虚拟机试一下**，因为GFW灰常牛逼的可以按协议把握手包和丢掉。我就是卡在这非常久，UDP连接提示验证失败，TCP连接客户端和服务器都收到错误码为-1的断开连接报文。死活没连上，最后我换用国内的一个VPS同样的搭建方法就直接正常连上了。



PPP和PPTPD
-------------
由于OpenVPN各种墙，所以我就想换一个支持度比较高的解决方案，使用pptp协议。CentOS 6 下大致过程如下：
```bash
#!/bin/sh
# 1. 安装软件包
# EPEL源
rpm -ivh "http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm";
yum install ppp pptp pptpd pptp-setup -y;

# 2. 配置
vim /etc/pptpd.conf;
# 2.1.1 去除 localip 前注释
# 2.1.2 去除 remoteip 前注释并把内容改为 192.168.10.100-200
vim /etc/ppp/option.pptpd
# 2.2 ms-dns 8.8.8.8 和 ms-dns 8.8.4.4
# 2.3 配置账户 
vim /etc/ppp/chap/secrets
# 添加 [用户名] pptpd [密码] * （pptpd应该可以改成*，表示匹配所有名称，不过我没尝试过）

# 3. 启动模块和添加初始化启动模块
modprobe ppp_mppe
modprobe ip_gre
echo "#!/bin/sh" > /etc/sysconfig/modules/pptpd.modules
echo "modprobe ppp_mppe" >> /etc/sysconfig/modules/pptpd.modules
echo "modprobe ip_gre" >> /etc/sysconfig/modules/pptpd.modules

# 4. pptpd启动和开机启动
service pptpd start
chkconfig --add pptpd
chkconfig --level 5 pptpd on
chkconfig --level 6 pptpd on

# 5. 开启防火墙策略
iptables -A INPUT -p tcp --dport 1723 -j ACCEPT
iptables -A INPUT -p tcp --dport 47 -j ACCEPT
iptables -A INPUT -p gre -j ACCEPT
iptables -A POSTROUTING -t nat -s 192.168.10.0/24 -o eth0 -j MASQUERADE
iptables -A INPUT -p UDP --dport 53 -j ACCEPT
iptables -A INPUT -p tcp --dport 53 -j ACCEPT
service iptables save
```

最后客户端连接的时候的配置里要注意
1. 关闭EAP
2. 打开 使用点到点加密(MPPE)
3. 放心地使用MS-Chapv2吧
4. 另外貌似要内核支持某个功能，可以通过dkms提供（忘记哪个模块了但是我的VPS里自带）

我按这个配置成功连上了，但是后来配置l2tp的时候给搞乱了配置，又不知道为什么连不上了，好麻烦。

IPSec和l2tp协议
-------------
这个协议最麻烦，而且我没连成功过。不过记录一下操作流程。
```bash
# 1. 安装 yum install openswan xl2tpd openswan-doc lsof libpcap-devel
# 2. 配置
vim  /etc/ipsec.conf
# 编辑 dumpdir=/var/run/pluto/
# 编辑 virtual_private=%v4:10.0.0.0/8,%v4:192.168.11.0/24,%v4:172.16.0.0/12,%v4:25.0.0.0/8,%v6:fd00::/8,%v6:fe80::/10

# /etc/ipsec.d/xl2tpd.conf
cat > /etc/ipsec.d/xl2tpd.conf <<EOF
# Add connections here

# sample VPN connection
# for more examples, see /etc/ipsec.d/examples/
#conn sample
#               # Left security gateway, subnet behind it, nexthop toward right.
#               left=10.0.0.1
#               leftsubnet=172.16.0.0/24
#               leftnexthop=10.22.33.44
#               # Right security gateway, subnet behind it, nexthop toward left.
#               right=10.12.12.1
#               rightsubnet=192.168.0.0/24
#               rightnexthop=10.101.102.103
#               # To authorize this connection, but not actually start it,
#               # at startup, uncomment this.
#               #auto=add

conn L2TP-PSK-NAT
    rightsubnet=vhost:%priv
    also=L2TP-PSK-noNAT

conn L2TP-PSK-noNAT
    authby=secret
    pfs=no
    auto=add
    keyingtries=3
    rekey=no
    ikelifetime=8h
    keylife=1h
    type=transport
    left=[本机IP或域名] #这里写公网IP，没固定IP的就到花生壳弄个动态域名解析。
    leftid=[本机IP或域名]
    leftprotoport=17/1701
    right=%any
EOF;

# 设置预共享密钥
vim  /etc/ipsec.d/xl2tpd.secrets
echo ': PSK "l2tpd.owent.net"' > /etc/ipsec.d/xl2tpd.secrets;
# 3. 网络设置
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.default.rp_filter=0
sysctl -w net.ipv4.conf.default.send_redirects=0
sysctl -w net.ipv4.conf.default.accept_redirects=0
# 建议把以上内容写进 /etc/sysctl.conf 后 执行 sysctl -p
# 3. 启动和测试
service ipsec start
ipsec verify
# 全部通过或N/A就可以了

# 4. xl2tpd设置 
vim /etc/ppp/options.xl2tpd
# 去除require-mschap-v2前注释
# name l2tpd
vim /etc/xl2tpd/xl2tpd.conf
# 改写以下内容
# [global]
# listen-addr = [服务器IP]
# ipsec saref = yes # 如果 ipsec verify 返回 SAref kernel support                                  [N/A] 则改成 no
# [lns default]
# ip range = 192.168.11.128-192.168.11.254
# local ip = 192.168.11.1
# name = l2tpd
vim /etc/ppp/chap-secrets 
# 设置用户名密码 [用户名] l2tpd [密码] *

# 5. iptables 规则
iptables -A INPUT -p tcp --dport 1194 -j ACCEPT
iptables -A INPUT -p udp --dport 1701 -j ACCEPT
iptables -A INPUT -p udp --dport 500 -j ACCEPT
iptables -A INPUT -p udp --dport 4500 -j ACCEPT
iptables -t nat -A POSTROUTING -s 192.168.11.0/24 -o eth0 -j MASQUERADE 
iptables -A FORWARD -s 192.168.11.0/24 -j ACCEPT
iptables -A FORWARD -d 192.168.11.0/24 -j ACCEPT
service iptables save
# 6. 启动xl2tpd和自动运行
chkconfig --level 2345 ipsec on
chkconfig --level 2345 xl2tpd on
```

不过这个我没连接成功过，不知道什么原因。
另外据说也可以使用[**strongswan**](https://www.strongswan.org/)取代openswan，而且strongswan还可以用来配置**IKEv1和IKEv2**协议。

简化VPN服务器安装Softether VPN
-------------
痛苦了两周之后，发现其实有简单暴力的VPN方案，就是日本的开源软件[Softether VPN](https://www.softether.org/)

Source列表: https://www.softether.org/5-download/src
Github地址: https://github.com/SoftEtherVPN/SoftEtherVPN/
Google Code地址: https://code.google.com/p/softether/source/browse/
Source Forge地址: http://sourceforge.net/p/softethervpn/code/ci/master/tree/src/

这玩意简化了VPN配置，可以再Linux上部署，然后用Windows管理程序连上去管理。并且支持很多协议，OpenVPN,l2tp,IKEv1,IKEv2,IKEv3,sstp等。（不够我值尝试过openvpn和l2tp，很好用）

这货安装很简单，直接按官网的文档即可。使用上有几个注意点
1. 加密算法最好选RC4-SHA，选其他的我的Android设备都有很大概率连不上
2. 需要开启虚拟HUB里的DHCP，否则不设置符合规范的地址会连不上
3. 注意设置left地址和right的地址范围，默认好像是192.168.1.1，和默认局域网网段冲突
4. 默认会监听443端口，建议关闭掉，否则和HTTPS冲突（我的VPS的Web服务器监听了443端口）。
5. 建议换掉OPenVPN协议的默认端口1194，原因嘛，嘿嘿
6. 需要开放使用的端口

```bash
#!/bin/sh
# 我开放的端口如下
iptables -A INPUT -p tcp --dport 47 -j ACCEPT
iptables -A INPUT -p tcp --dport 53 -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j ACCEPT
iptables -A INPUT -p tcp --dport 992 -j ACCEPT
iptables -A INPUT -p udp --dport 992 -j ACCEPT
iptables -A INPUT -p udp --dport 500 -j ACCEPT
iptables -A INPUT -p tcp --dport 1723 -j ACCEPT
iptables -A INPUT -p udp --dport 4500 -j ACCEPT
iptables -A INPUT -p tcp --dport 5555 -j ACCEPT
service iptables save
```

通用要注意的地方
-------------
1. 最后检查一下/etc/sysconfig/iptables里的重复项，去掉保存
2. **CentOS 7下默认使用firewall**而不是iptables，需要相应地修改配置才行
3. 一些系统，比如CentOS 7下默认**使用systemd的systemctl命令**而不是传统的chkconfig来控制服务，也要做相应得变更 
4. 注意CentOS里的selinux(可通过sestatus查看状态，建议关掉就好，没啥作用)
5. 注意Ubuntu下的防火墙ufw

