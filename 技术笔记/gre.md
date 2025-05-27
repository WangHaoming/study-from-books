# 概念

GRE（Generic Routing Encapsulation，通用路由封装）是一种隧道协议，用于在两个网络设备之间封装任意网络层协议的数据包，并通过一个公用网络（通常是 IP 网络）进行传输。简单来说，GRE 隧道就是在两个设备之间建立一个虚拟的点对点连接，用于承载被封装的网络流量。

GRE是网络层协议，就是传统上我们说的IP层协议。


# GRE 的工作原理

- 封装
> 发送端设备将原始的网络层数据包（比如一个 IPv4 数据包）封装到 GRE 包中，GRE 包会添加一个 GRE 头，再由外层 IP 头封装，形成一个新的 IP 数据包。

- 传输
> 这个封装后的数据包通过互联网或其他 IP 网络传输，数据包的目的地址是对端 GRE 设备的公网 IP。

- 解封装
> 接收端收到封装后的 IP 包后，根据 GRE 协议头识别出里面被封装的原始数据包，然后将其解封装，还原出原始的网络层数据包，并将其发送到目标网络。


# 举例说明


比如你有一个需求，需要连接两个子网（局域网，subnet），它们内部的设备需要使用内网IP彼此通信，它们其实在不同的"局域网"，两个子网（不在同一个二层网络）必须要通过公共网络才能互相通信。这时就需要GRE出场了，我们可以把GRE隧道想象成一条"管道"，管道的两端分别是两个设备。管道内传输的是"包裹"（原始数据包，IP报文），在传输的过程中在外层有包裹了一层GRE封装，相当于内层就是一个完整的IP包，外层再包一层GRE的封装，然后然后把管道两端的公网IP写入。这样这个GRE包到到对端之后在拆开外包中，然后再对端内部当成一个普通的IP包传输。

# 配置

## 准备工作

需要两个Linux主机（HostA和HostB），各自有公网IP（或者可以互访的IP地址）。

- 假设：
    - HostA 公网IP： 192.0.2.1
    - HostB 公网IP： 198.51.100.1

- GRE隧道双方的对端接口内网IP地址是：
    - HostA的内网地址：10.0.0.1
    - HostB的内网地址：10.0.0.2





## 配置步骤

### １．创建GRE隧道接口

在HostA上执行

```
sudo ip tunnel add gre1 mode gre remote 34.44.203.186 local 10.128.0.11 ttl 255
sudo ip link set gre1 up
sudo ip addr add 10.0.0.1/30 dev gre1
```
在 Host B 上执行：

```
sudo ip tunnel add gre1 mode gre remote 34.27.138.140 local 10.128.0.12 ttl 255
sudo ip link set gre1 up
sudo ip addr add 10.0.0.2/30 dev gre1
```



## 防火墙阻止了包发出,这个时候 tcpdump也抓不到发出去的包? 为什么?

> 为什么抓不到发出的 ICMP 请求包？

抓包的通常位置和类型
本地抓包工具（如 tcpdump）默认是在 VM 的网络接口（例如 eth0）抓包。

ICMP Echo Request 包生成后，通常会经过本机网络协议栈，从应用层到内核网络层，最后发送到网卡。

但如果防火墙规则阻止 ICMP 流量
GCP 防火墙在虚拟网络层直接丢弃 ICMP 包，包不会被允许进入或离开虚拟机的虚拟网卡。

防火墙阻止的包可能在虚拟网络层被丢弃，导致包还没真正到达虚拟网卡驱动层，因此 tcpdump 无法捕获它。

### 简单理解：
发送 ICMP 包的流程被防火墙拦截了，包没有实际发往网卡，抓包工具在物理层看不到发出的包。

这就是为什么你“ping”命令执行时，实际上包被阻断了，但本机抓包抓不到发出的 ICMP 请求。


VPC flow log和firewall rule log都不在得捕获不到deny log


## Debug

- 抓取gre包
```
sudo tcpdump -i eth0 proto 47 -vvv
```

- 抓取指定IP的包
```
sudo tcpdump -i eth0 host 198.51.100.1 and tcp port 80
```

- 抓取指定源IP的包
```
sudo tcpdump -i eth0 src host 198.51.100.1
```

- 抓取指定目的IP的包
```
sudo tcpdump -i eth0 dst host 198.51.100.1
```






