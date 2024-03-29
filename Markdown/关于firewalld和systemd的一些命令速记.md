关于firewalld和systemd的一些命令速记
======

[TOC]

前言
------

CentOS 7 已经用firewalld替换掉了iptables并用systemd来管理启动服务（之前是chkconfig）。而且下一个Ubuntu的长期支持版也要这么干了。

这两个工具在操作上和之前的系统有很多的变化，所以集中记录一下常用的命令，以免每次都要靠搜索引擎。

firewalld
------

关于firewalld：http://fedoraproject.org/wiki/FirewallD/zh-cn

图形化配置工具： firewall-config

命令行工具：firewall-cmd


默认配置位于： /usr/lib/firewalld

用户配置位于： /etc/firewalld

### 添加服务
在 /etc/firewalld/services 创建 [服务名称].xml
格式如下:
```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>服务名称</short>
  <description>服务名称 server port whitelist</description>
  <port protocol="协议" port="端口"/>
  <port protocol="tcp" port="8001"/>
</service>
```

### 添加区域规则

和添加服务类似，可以从/usr/lib/firewalld/zones拷贝到/etc/firewalld/zones然后改。

主要流程是控制某一个区域开启哪些服务或者端口

### 常用命令
```bash
# 重载
firewall-cmd --reload
# 状态
firewall-cmd --state

# 添加/移除/查询服务
firewall-cmd [--permanent] [--zone=<zone>] --add-service=<service> [--timeout=<seconds>]
firewall-cmd [--zone=<zone>] --remove-service=<service>
firewall-cmd [--zone=<zone>] --query-service=<service>

# 添加/移除/查询端口
firewall-cmd [--zone=<zone>] --add-port=<port>[-<port>]/<protocol> [--timeout=<seconds>]
firewall-cmd [--zone=<zone>] --remove-port=<port>[-<port>]/<protocol>
firewall-cmd [--zone=<zone>] --query-port=<port>[-<port>]/<protocol>

# 添加/移除/查询端口转发
firewall-cmd [--zone=<zone>] --add-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
firewall-cmd [--zone=<zone>] --remove-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
firewall-cmd [--zone=<zone>] --query-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
# 例: 将区域home的ssh转发到127.0.0.2
firewall-cmd --zone=home --add-forward-port=port=22:proto=tcp:toaddr=127.0.0.2

# 直接访问（类似iptable的操作）
firewall-cmd [--permanent] --direct --get-all-chains
firewall-cmd [--permanent] --direct --get-chains { ipv4 | ipv6 | eb } table
firewall-cmd [--permanent] --direct --add-chain { ipv4 | ipv6 | eb } table chain
firewall-cmd [--permanent] --direct --remove-chain { ipv4 | ipv6 | eb } table chain
firewall-cmd [--permanent] --direct --query-chain { ipv4 | ipv6 | eb } table chain

firewall-cmd [--permanent] --direct --get-all-rules
firewall-cmd [--permanent] --direct --get-rules{ ipv4 | ipv6 | eb } table
firewall-cmd [--permanent] --direct --add-rules{ ipv4 | ipv6 | eb } table chain priority args
firewall-cmd [--permanent] --direct --remove-rules{ ipv4 | ipv6 | eb } table chain priority args
firewall-cmd [--permanent] --direct --query-rules{ ipv4 | ipv6 | eb } table chain priority args

firewall-cmd [--permanent] --direct --get-all-passthroughs
firewall-cmd [--permanent] --direct --get-passthroughs{ ipv4 | ipv6 | eb }
firewall-cmd [--permanent] --direct --add-passthroughs{ ipv4 | ipv6 | eb } args
firewall-cmd [--permanent] --direct --remove-passthroughs{ ipv4 | ipv6 | eb } args
firewall-cmd [--permanent] --direct --query-passthroughs{ ipv4 | ipv6 | eb } args

# 直接访问例子
firewall-cmd --direct --add-rule ipv4 filter INPUT 0 -p tcp --dport 80 -j ACCEPT

# 获取所有可用配置集
firewall-cmd --get-zones
firewall-cmd --list-all-zones

# 获取所有可用服务
firewall-cmd --get-services
firewall-cmd --get-icmptypes

# 获取已经启用的服务
firewall-cmd [--zone=<zone>] --list-services
```

systemd
------

关于systemd： https://wiki.archlinux.org/index.php/Systemd_%28%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87%29

系统配置位置：/usr/lib/systemd/system/

用户配置位置：/etc/systemd/system/

其实systemd的enable操作就是把系统配置软链接到用户配置

### 添加服务

具体文档可以 *man 5 systemd.unit* 和 *man 5 systemd.service*

直接在 /usr/lib/systemd/system或者/usr/lib/systemd/user里添加 <单元>.service文件

```bash
systemctl enable <单元>.service
```
然后执行上面的命令即可


文件内容示例：
```
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target
 
[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
```

### 启动级别
SysV 启动级别      | Systemd 目标      | 注释
------------------|------------------|-------
0                 | runlevel0.target, poweroff.target | 中断系统（halt）
1, s, single      | runlevel1.target, rescue.target | 单用户模式
2, 4              | runlevel2.target, runlevel4.target, multi-user.target | 用户自定义启动级别，通常识别为级别3。
3                 | runlevel3.target, multi-user.target | 多用户，无图形界面。用户可以通过终端或网络登录。
5                 | runlevel5.target, graphical.target | 多用户，图形界面。继承级别3的服务，并启动图形界面服务。
6                 | runlevel6.target, reboot.target | 重启
emergency         | emergency.target | 急救模式（Emergency shell） 

### 常用命令

```bash
# systemd 重载配置
systemctl daemon-reload
systemctl restart <单元>

# 查看启动日志
journalctl -b

# 开启/关闭/查询自自动服务
systemctl enable/disable/enable <单元>

# 启动、重启、重载、关闭服务
systemctl start/restart/reload/stop <单元>

# 列举所有服务单元
systemctl list-units

# 关机
systemctl poweroff

# 重启
systemctl reboot
```

顺带记一下怎么开启coredump
------

```bash
# 
COREDUMP_DIR=/home/coredump

# 配置资源限制 
echo '#!/bin/sh

ulimit -S -c unlimited > /dev/null 2>&1
' > /etc/profile.d/coredump.sh;

# 配置文件模式 
echo "kernel.core_pattern = $COREDUMP_DIR/%e.%t.%p.coredump" > /etc/sysctl.d/99-sysctl.conf;

mkdir -p $COREDUMP_DIR;
chmod 666 $COREDUMP_DIR;

# 重载配置 
sysctl -p;

# 检查模式 
cat /proc/sys/kernel/core_pattern ;

# 资料 
# http://man7.org/linux/man-pages/man5/core.5.html 
# man 5 core 
```



> Written with [StackEdit](https://stackedit.io/).

