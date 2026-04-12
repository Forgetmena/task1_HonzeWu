# task0_HongzeWu

## 一，任务实现流程

### 1，使用VMware安装Ubuntu操作系统（Ubuntu24.04.4）

### 2，配置linux主机可以连接到互联网

2.1 使用NAT模式，在物理机内部形成虚拟的局域网同时自动通过电脑连接到互联网，后续任务基于NAT模式  
2.2 使用桥接模式，能够直接连接到本电脑连接的WIFI，在后续任务中，能够实现连接不同WiFi的电脑之间的ping通  
2.3 用命令 **ping -c 4 baidu.com** 验证是否连上互联网   

### 3，认识网卡

用命令 **ip a** 查看机器上所有的网卡和它们的ip地址，其中包含的ens33就是用来上网的虚拟网卡，后续在用两台虚拟机抓包时就要监听ens33，记录下虚拟机Ubuntu_A的IP地址为，10.0.0.128

### 4，安装wireshark（在后续用namespace实现ping通也使用了tcpdump，此处不做过多赘述）

`sudo apt update` 获取应用商店最新的软件列表  
`sudo apt install wireshark`  需要允许non-superusers抓包  
`sudo usermod -aG wireshark $USER` 把当前用户加入到有资格抓包的群组中  

```
对一些命令的解释；
sudo:SuperUser DO
apt:Advanced Package Tool
```

### 5，抓包baidu.com

4.1 打开wireshark并且打开ens33  
4.2 不关闭wireshark并且打开终端，输入 **ping -c 4 baidu.com**
4.3 停止捕获  
4.4 使用过滤器输入icmp  
4.5 Info栏交替 **Echo (ping) request** / **Echo (ping) reply** 前者是我给百度发送的请求。点击Echo (ping) request之后，在Internet Portocol Version 4(IPV4)栏可以发现，Src:10.0.0.128,Dst:198.18.0.41，前者是虚拟机的IP，后者是百度的IP，说明是虚拟机向百度发送的请求，打开此栏后，可以在Protocol发现是ICMP(1)协议，与我们的筛选相同。同样打开Internet Control Message Protocol栏，可以找到Type 8 (Echo (ping) request)，与我最开始点击的相同。

### 6，TCP协议

6.1 TCP三次握手

```
整个过程其实很简单，首先要明白ACK，在每次信息的传输后，都要有ACK来确认收到了信息
[SYN] 客户端向服务器发送请求
[SYN,ACK] 服务器确认收到，并且向客户端发送请求
[ACK] 客户端确认收到信息
此时客户端和服务器才正式确认信息是可以互相传输的，因此才有之后的信息传输
```

6.2 使用curl命令访问百度网页（基本操作后续不再过多赘述）

`curl www.baidu.com`   

6.3 停止抓包并筛选 tcp.port == 80（80端口是HTTP网页浏览默认的端口）
6.4 前三行即是TCP三次握手，同样点击[SYN]。打开Transmission Control Protocol，打开Flags，可以发现Syn被设置为1，并且set，且Acknowledgment: Not set (0)。打开[SYN,ACK]，发现两者都被设置为1。同样，通过Transmission Control Protocol还可以获得更多信息，某次任务进行时，打开[SYN]后，Src Port被设置为了44468，Dst Port为80，而在[SYN,ACK]中二者则是相反的，这也侧面反映了TCP三次握手的过程

### 7，查看GET / HTTP/1.1

打开Hypertext Transfer Protocol。注意到Host: www.baidu.com\r\n，说明找的时www.baidu.com这个网站。注意到User-Agent: curl/8.5.0\r\n，这是客户端自报家门。注意到Accept: */*\r\n，接受任何格式的数据。

### 8，HTTP/1.1 200 OK说明请求成功，成功拿到网页代码

打开后可展开Line-based text data: text/html看到网页代码

## 二，遇到的问题以及解决措施

## 三，一些额外了解的知识

### IP地址和MAC地址

IP地址：网络层使用，连接不同的WiFi会改变，帮助数据包找到网络
MAC地址：物理层
在第一次抓包百度时，Ethernet II那一层的Destination的MAC地址，是学校路由器的MAC地址或者VMware虚拟路由器的MAC地址，而不是百度的，至于如何抓包到百度，应该是不断的网络传输  

### 由ping -c 4 baidu.com和curl www.baidu.com的Dst IP不同引发有关IP的知识

1.由于百度过于庞大，是由无数个服务器组成的，也就是拥有无数个不同的IP地址   
2.DNS：网络电话本，隔一段时间curl www.baidu.com，DNS都有可能返回不同的IP地址  
3.利用 **nslookup baidu.com** 该命令，发现百度返回的IP是不同的

### 由curl baidu.com引发的有关重定向的知识

1.curl没有像谷歌浏览器一样的重定向功能，访问baidu.com时，实际上百度已经“搬家”  
2.通过 **curl -I baid.com** 查看百度搬家的信息，第一行为：**HTTP/1.1 301 Moved Permanently**，第二行为：**Location: http://www.baidu.com/**。   
3.通过 **curl -L baidu.com** 进行重定向时自动跟随新地址跳转，从而实现和 **curl www.baidu.com**同样的效果  