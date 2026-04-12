# task1_HongzeWu

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

5.1 打开wireshark并且打开ens33  
5.2 不关闭wireshark并且打开终端，输入 **ping -c 4 baidu.com**  
5.3 停止捕获  
5.4 使用过滤器输入icmp  
5.5 Info栏交替 **Echo (ping) request** / **Echo (ping) reply** 前者是我给百度发送的请求。点击Echo (ping) request之后，在Internet Portocol Version 4(IPV4)栏可以发现，Src:10.0.0.128,Dst:198.18.0.41，前者是虚拟机的IP，后者是百度的IP，说明是虚拟机向百度发送的请求，打开此栏后，可以在Protocol发现是ICMP(1)协议，与我们的筛选相同。同样打开Internet Control Message Protocol栏，可以找到Type 8 (Echo (ping) request)，与我最开始点击的相同。

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

### 9，TCP四次挥手

9.1 某一种标准情况如下（绝对不会有纯[FIN]）：  

```
[FIN,ACK] 客户端收到上一次消息，并且给服务器说自己要关通道了
[ACK] 服务器收到了请求
[FIN,ACK] 服务器确认并且提出关闭通道
[ACK] 客户端收到请求
```

9.2 会出现其他情况，例如第二次和第三次合并为[FIN,ACK]，第三次变为[FIN,PSH,ACK]等等  
9.3 也可能直接[RST]\[RST,ACK]当服务器过于繁忙时，为了节省资源，使用RST暴力切断连接

### 10，在NAT模式下实现两台虚拟机的ping通

10.1 克隆虚拟机Ubuntu_A,并且命名为Ubuntu_B  
10.2 通过命令 **ip a** 记录A，B的IP地址，分别为：10.0.0.128，10.0.0.129  
10.3 在A终端中输入命令 **ping 10.0.0.129**，发现已经ping通  
10.4 我们把A作为一个服务器，在9999端口等待连接，因此我们在A机的终端输入 **nc -l -p 9999** 表示A作为一个监听者在9999端口监听，并且同时在wireshark中筛选tcp.port === 9999      
10.5 在B终端输入 **nc 10.0.0.128 9999**，连接建立成功  
10.6 A的wireshark停止抓包，可以看到TCP三次握手  
10.7 重复上述步骤但不停止抓包，或者在wireshark中继续抓包但不保存  
10.8 B终端中输入Hello from B，可以在A的终端中收到，同时在A的wireshar左键点击有关包，可以在右下角看到Hello from B,或者右键有关包，点击追随流，可以直接看到Hello from B，如果之前还在A中输入Heloo from A，追随流中显示的二者的颜色还不通  
10.9 观察TCP三次握手的Source IP和Destination IP，这就可以用来明确哪个是客户端，哪个是服务器。观察Ports、Flags等等。以上观察方法均同理。
```
对一些命令的解释
nc Netcat，建立网络连接
-l 监听
-p 端口，注意，端口是可以随意设置的，但是有一些规则
0-1023（特权端口/知名端口）；1024-49151（注册端口/普通用户端口）；49152-565535（动态私有端口，留给客户端临时使用，可以知道客户端B的端口号很大，实际也是如此，B的端口号为59928，符合端口的使用规则）
```

### 11，使用namespace和veth pair实现ping通。其实原理和10是一样的，只不过11是在虚拟机上划出两片独立的网络空间，然后设置虚拟网卡、配置IP，然后设置虚拟网线连接两片空间，进而实现模拟两台虚拟机之间的ping通，此部分较难的点应该是大量的命令。以下主要以命令为主。

`sudo ip netns add nsA` 划出nsA的独立网络空间  
`sudo ip netns add nsB` 
`ip netns list` 查看这两片空间  
`sudo ip netns exec nsA ip a` 查看nsA的ip，发现没有网卡也没有IP  
`sudo ip link add veth-a type veth peer name veth-b` 创建新的网络设备，设备类型是veth（网线），网线的一头是veth-a，对等的另外一头是veth-b，同时二者被命令  
`sudo ip link set veth-a netns nsA` 把网线veth-a的一端接入网络空间nsA  
`sudo ip link set veth-b netns nsB`
`sudo ip netns exec nsA ip link set veth-a name eth0` 对网络空间A执行操作，操作为将veth-a重命名为eth0  
`sudo ip netns exec nsA ip link set eth0 up` 对网络空间A进行处理，操作为启用eth0   
`sudo ip netns exex nsA ip addr add 10.1.1.1/24 dev eth0` 对网络空间A做处理，操作为给它分配一个新的IP地址  
`sudo ip netns exec nsB ip link set veth-b name eth0`  
`sudo ip netns exec nsB ip link set eth0 up`  
`sudo ip netns exec nsB ip addr add 10.1.1.2/24 dev eth0`  
`sudo ip netns exec nsA tcpdump -i eth0 tcp port 9999 -w /tmp/nsA_capture.pcap` 对网络空间nsA执行操作，操作为让tcpdump在nsA中看住eth0网卡的9999端口，并且把抓到的包保存为文件，由于wireshark权限问题，不能进入nsA和nsB，因此采用tcpdump  
`sudo ip netns exec nsA nc -l -p 9999` 打开第二个终端，让nsA作为服务器监听9999端口  
`sudo ip netns exec nsB nc 10.1.1.1 9999`打开第三个终端，让nsB去连接nsA  
`wireshark /tmp/nsA_capture.pcap`  用wireshark打开文件，之后操作如10  

## 二，一些额外了解的知识

### 1.IP地址和MAC地址

IP地址：网络层使用，连接不同的WiFi会改变，帮助数据包找到网络
MAC地址：物理层
在第一次抓包百度时，Ethernet II那一层的Destination的MAC地址，是学校路由器的MAC地址或者VMware虚拟路由器的MAC地址，而不是百度的，至于如何抓包到百度，应该是不断的网络传输  

### 2.由ping -c 4 baidu.com和curl www.baidu.com的Dst IP不同引发有关IP的知识

1.由于百度过于庞大，是由无数个服务器组成的，也就是拥有无数个不同的IP地址   
2.DNS：网络电话本，隔一段时间curl www.baidu.com，DNS都有可能返回不同的IP地址  
3.利用 **nslookup baidu.com** 该命令，发现百度返回的IP是不同的

### 3.由curl baidu.com引发的有关重定向的知识

1.curl没有像谷歌浏览器一样的重定向功能，访问baidu.com时，实际上百度已经“搬家”  
2.通过 **curl -I baid.com** 查看百度搬家的信息，第一行为：**HTTP/1.1 301 Moved Permanently**，第二行为：**Location: http://www.baidu.com/**。   
3.通过 **curl -L baidu.com** 进行重定向时自动跟随新地址跳转，从而实现和 **curl www.baidu.com**同样的效果  

### 4.nc和ping的区别，nc（TCP协议）是直接上门，直接建立联系；而ping（ICMP协议）只是探路，测试连通性的工具