# Task1中需要的Linux基础命令

## 一，文件与目录操作

### 1，基础导航

`pwd` 查看当前目录  
`ls -la` 列出文件（带细节）  
`cd /etc/netplan` 进入目录  
`cd ..` 返回上级没目录  
`cd ~` 回到用户主目录

### 2，文件编辑

`sudo nano /etc/netplan/00-install-config.yaml` 用nano编辑配置文件  
**快捷键**

    Ctrl + O  保存
    Ctrl + X  退出
    Ctrl + K  剪切行
    Ctrl + U  粘贴

### 3，文件管理

`sudo cp /etc/netplan/00-install-config.yaml ~/netplan-backup.yaml` 复制配置文件  
`mv old-file.txt new-file.txt` 移动/重命名  
`cat /etc/os-release` 显示系统信息（查看文件内容） 
`less /var/log/syslog` 分页查看日志（查看文件内容）

## 二，网络配置核心命令

### 1，查看网络状态

`ip addr show`  
`ip link show up` 仅查看活动接口  
`ip route show` 仅查看路由表  
`nslookup baidu.com` 测试DNS解析

### 2，网络连通性测试

`ping -c 4 8.8.8.8` 基础ping测试，发送4个包后停止  
`traceroute baidu.com` 跟踪路由  
`nc -zv 192.168.1.100 22` 测试SSH端口

### 3，应用网络配置

`sudo netplan apply` 应用Netplan配置  
`sudo systemctl restart NetWorkManager` 重启网络服务

## 抓包实验必备命令

### 1，tcpdump基础用法

`sudo tcpdump -i ens33 icmp` 抓取所有ICMP包  
`sudo tcpdump -3 ens33 port 53 -nn` 抓取DNS流量  
`sudo tcpdump -i ens33 -w dns_capture.pcap host baidu.com` 保存抓包到文件  

### 2，Wireshark操作

`wireshark` 启动图形界面（需先配置权限）  
`wireshark dns_capture.pcap` 从文件打开抓包

### 3，权限配置

`sudo usermod -aG wireshark $USER` 将用户加入wireshark组  
`sudo setcap 'CAP_NET_RAW+eip CAP_NET_ADMIN+eip' /user/bin/dumpcap` 设置抓包权限  
`getcap /user/bin/dumpcap` 验证权限

## 四，系统管理实用命令
