## 一、单臂路由
### 1.示意图
![](DICM/Pasted%20image%2020251112203552.png)
### 2.工作原理
1、PC1访问PC2，发现PC2和自己不在同一个网段，PC1就会发送ARP报文去解析网关IP的MAC
	该广播报文到达交换机以后，根据G0/0/1口打上PVID对应的VLAN10
2、交换机SW1收到ARP广播，在VLAN10中采取泛洪的处理
3、路由器根据VLAN10，将数据包送入子接口100，子接口100解封装，发现targetIP是自己的IP192.168.1.254所以先将PC1的IP和MAC放入到ARP缓存表，回复PC1的ARP reply，告诉PC1 自己的MAC
4、PC1收到路由器回复的ARP reply，将192.168.1.254和MAC放入到ARP缓存表中。
5、PC1根据ARP表项，对访问192.168.10.1的数据进行封装
6、路由器收到报文后，发现目的MAC是自己的MAC，解封装，露出IP头，查路由表发现192.168.20.1是自己的直连路由，直连路由的接口是200号子接口
7、路由器朝着200子接口发送ARP报文，请求192.168.20.1的MAC
8、PC2收到路由器发送的ARP报文，发现ARP载荷中target IP是自己，将路由器的IP和MAc放入自己的ARP缓存中
9、PC2回复ARP replay.
10、路由器接收到ARP replay后重新封装发给PC1
### 3.配置命令
`int g0/0/0.100`                     进入路由器端口0的子接口
`dot1q termination vid 10`
`ip addres 192.168.10.254 24 `配置IP地址和子网掩码
`arp broadcast enable`            开启ARP广播
## 二、VLAN if
### 1.示意图
![](DICM/Pasted%20image%2020251114160836.png)

### 2.配置命令
`int vlanif 10`进入vlanif 10的接口

## 三、ARP代理
![](DICM/Pasted%20image%2020251114163412.png)



