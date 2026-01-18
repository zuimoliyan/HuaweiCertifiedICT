# 一、实验拓扑
![](DICM/Pasted%20image%2020260118164037.png)
# 二、IP 地址规划
1. AR1 与 ISP 互联 `101.1.1.0/24`，ISP的`g0/0/0`地址为 `101.1.1.1`
2. AR3 与 ISP 互联 `101.1.2.0/24`，ISP的 `g0/0/0` 地址为 `101.1.2.1`，AR3 的IP地址为 `101.1.2.1`
3. Server 属于`Vlan172`，ip 地址为 `172.16.1.1/24` 网关为 `172.16.1.254/24`
4. PC1 属于 `Vlan10`，ip 地址为：`192.168.10.1`，网关是 `192.168.10.254/24` 
5. Client1 属于 `Vlan20`，ip 地址为：`192.168.20.1`，网关是 `192.168.20.254/24`
6. PC2 属于 `Vlan30`，IP 地址自动获取，网关是 `192.168.30.254/24`
7. Server2 属于 `Vlan30`，ip 地址为：`192.168.30.253`，网关是 `192.168.30.254/24`
8. 打印机属于 `Vlan30`，IP 地址自动获取到 `192.168.30.123` ，网关是 `192.168.30.254/24`
9. AR3和SW4之间的互联地址为`192.168.4.0/24` 其中AR3的地址为`192.168.4.1`，SW4的地址为`192.168.4.2`
10. 运营商后方有一网段`8.8.8.8/32`，用loopback 0接口代替模拟

# 三、实验需求
## 实验需求一：A公司网络规划如下
1. `SW1/2/3` 组成了 A 公司的交换网络，SW1身上不可以有接口阻塞
2. 三台交换机上创建 `vlan10、20、172` 由于路由器接口不足，所以由交换机中转汇总，路由器做为终端的网关
3. 交换机之间配置 trunk 链路，仅允许以上三个 vlan 通过
4. 交换机LSW1和路由器AR1中间的链路捆绑成一条，需要能够检查错连，且路由器为主设备
5. 所有终端设备的网关均在R1身上，允许 A 公司网络主机访问外网
6. AR1 要对运营商拨号上网，用户名为`huawei`，密码是`huawei1` 
7. Server 模拟内网服务器，Client1 <font color="#ff0000">只能 </font>HTTP 服务，无法通过 ping 命令访问 
8. A 公司所有的网络设备console密码为 `Huawei123`
## 实验需求二：B 公司网络规划如下
1. SW4 组成了 B 公司的交换网络
2. 在 SW4 上创建 vlan30，终端设备从交换机获取地址，获取的地址租期为2天
3. 在 PC2 和打印机上确认可以获取 IP 地址和网关。打印机必须被分配`192.168.30.123`
4. R3 与 SW4 之间运行 OSPF，其中 R3 的接口要成为 `DR`
5. 查看配置的时候，要在进程中可以看到宣告情况
6. 允许 B 公司网络主机访问外网，可允许外网用户用 `101.1.2.111` 来访问Server2

## 四、实验分析及配置
### 公司 A
#### 1. 对于三台交换机
![](DICM/Pasted%20image%2020251221155810.png)
1. `SW1/2/3 ` 组成了 A 公司的交换网络，<font color="#ff0000">SW1身上不可以有接口阻塞</font>
	- `SW1` 在选举过程中的<font color="#ff0000">优先级不能是最低的</font>
2. 三台交换机上创建 `vlan10、20、172` 由于路由器接口不足，所以由交换机中转汇总，<font color="#ff0000">路由器做为终端的网关</font>
	- 与 SW1相连的 <font color="#ff0000">AR1作为网关</font>
3. 交换机之间配置 trunk 链路，仅允许以上三个 vlan 通过

SW1：
```
stp mode stp
stp priority 0
vlan batch 10 20 172
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 172
 undo port trunk allow-pass vlan 1
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 172
 undo port trunk allow-pass vlan 1
#
interface GigabitEthernet0/0/10
 port link-type access
 port default vlan 172
```
SW2：
```
vlan batch 10 20 172
stp mode stp
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 172
 undo port trunk allow-pass vlan 1
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 172
 undo port trunk allow-pass vlan 1
 #
 interface GigabitEthernet0/0/3
  port link-type access
  port default vlan 10
```
SW3：
```
vlan batch 10 20 172
stp mode stp
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 172
 undo port trunk allow-pass vlan 1
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 172
 undo port trunk allow-pass vlan 1
 #
interface GigabitEthernet0/0/3
 port link-type access
 port default vlan 20
```

#### 2. SW1与 AR1的连接
![](DICM/Pasted%20image%2020260118164051.png)

1. 交换机`SW1`和路由器 `AR1`中间的链路捆绑成一条，需要能够检查错连，且<font color="#ff0000">路由器为主设备</font>
	- `SW1` 和 `AR1` 通过链路聚合捆绑，并设置 `AR1` 的优先级高于 `SW1`
2. 所有终端设备的网关均在 R1身上
	- 正常把 `AR1` 的<font color="#ff0000">聚合接口</font>设置为网关
3. Server 模拟内网服务器，Client1 <font color="#ff0000">只能 </font>HTTP 服务，无法通过 ping 命令访问 
	- 在 `AR1` 的聚合接口设置 `ACL`

AR1：
1. **AR1 的 Eth-Trunk 必须配置为三层聚合口**
	- 二层 Eth-Trunk 仅能透传二层帧（如 VLAN），无法实现三层路由转发
	- 配置命令：<font color="#ff0000">undo portswitch</font>
2. `Eth-Trunk1.10`、`Eth-Trunk1.20`、`Eth-Trunk1.172` 分别配置为三个终端网络区域的网关
```
lacp priority 0
#
interface Eth-Trunk1
 undo portswitch
 mode lacp-static
#
#
interface GigabitEthernet0/0/1
 eth-trunk 1
#
interface GigabitEthernet0/0/2
 eth-trunk 1
#
interface Eth-Trunk1.10
 dot1q termination vid 10
 ip address 192.168.10.254 255.255.255.0 
 arp broadcast enable
#
interface Eth-Trunk1.20
 dot1q termination vid 20
 ip address 192.168.20.254 255.255.255.0 
 arp broadcast enable
#
interface Eth-Trunk1.172
 dot1q termination vid 172
 ip address 172.16.1.254 255.255.255.0 
 traffic-filter outbound acl 3000
 arp broadcast enable
#
acl number 3000  
 rule 5 deny icmp source 192.168.20.1 0 destination 172.16.1.1 0 
 rule 10 permit tcp source 192.168.20.1 0 destination 172.16.1.1 0 
 rule 20 deny ip source 192.168.20.1 0 destination 172.16.1.1 0 
#
```
SW1：
```
#
interface Eth-Trunk1
 mode lacp-static
 port link-type trunk
 port trunk allow-pass vlan 10 20 172
#
interface GigabitEthernet0/0/11
 eth-trunk 1
#
interface GigabitEthernet0/0/12
 eth-trunk 1
```
#### 3. AR1与 ISP 的连接
![](DICM/Pasted%20image%2020260118164058.png)

1. AR1 要对运营商拨号上网，用户名为 `huawei`，密码是 `huawei1` 
2. 允许 A 公司网络主机访问外网

AR1：
```
interface Dialer 1
 dialer user huawei 
 dialer bundle 1 
 PPP chap user huawei 
 ppp chap password cipher huawei1 
 ip address ppp-negotiate
#
interface g0/0/0
 pppoe-client dial-bundle-number 1
#
ip route-static 0.0.0.0 0 Dialer 1
#
acl 2000
 rule permit source any
#
interface Dialer 1
 nat outbound 2000
```
ISP：
```
interface LoopBack 0
 ip address 8.8.8.8 32
#
ip pool ISP
 network 101.1.1.0 mask 24
 gateway-list 101.1.1.1
#
int Virtual-Template 1
 ppp authentication-mode chap
 ip address 101.1.1.1 24
 remote address pool ISP
#
int g0/0/0
 pppoe-server bind virtual-template 1
#
aaa
 local-user huawei password cipher huawei1
 local-user huawei service-type ppp
```
#### 4.所有的网络设备 console 密码为 Huawei123
```
user-interface console 0
 set authentication password cipher Huawei123
```


### 公司 B
#### 1. 对于交换机和终端
![](DICM/Pasted%20image%2020251221162809.png)
1. 在 SW4 上创建 vlan30，终端设备从交换机获取地址，获取的地址租期为2天
	- 创建 vlan、将 SW4作为 DHCP 服务器，配置 vlanif
2. 在 PC2 和打印机上确认可以获取 IP 地址和网关。打印机必须被分配 `192.168.30.123`
	- 在 DHCP 服务器配置**静态绑定**、**排除**、**租期**

SW4：
```
vlan 30
#
dhcp enable
#
ip pool dhcp
 gateway-list 192.168.30.254
 network 192.168.30.0 mask 255.255.255.0
 static-bind ip-address 192.168.30.123 mac-address 5489-9819-5065
 excluded-ip-address 192.168.30.253
 lease day 2 hour 0 minute 0
#
interface Vlanif30
 ip address 192.168.30.254 255.255.255.0
 dhcp select global
#
interface GigabitEthernet0/0/11
 port link-type access
 port default vlan 30
#
interface GigabitEthernet0/0/12
 port link-type access
 port default vlan 30
#
interface GigabitEthernet0/0/13
 port link-type access
 port default vlan 30
#
```

#### 2. 交换机和 AR3的连接
![](DICM/Pasted%20image%2020251221163158.png)

R3 与 SW4 之间运行 OSPF，其中 R3 的接口要成为 `DR`
- 在 SW4配置 VLANIF，将 `AR3` 的 OSPF 优先级设置高于 `SW4`

SW4：
```
vlan 100
#
interface Vlanif100
 ip address 192.168.4.2 255.255.255.0
#
interface GigabitEthernet0/0/1
 port link-type access
 port default vlan 100
#
ospf 1 router-id 4.4.4.4
 area 0.0.0.0
  network 192.168.30.0 0.0.0.255
  network 192.168.4.0 0.0.0.255
#
ip route-static 0.0.0.0 0 192.168.4.1
```
AR3：
```
interface GigabitEthernet0/0/1
 ip address 192.168.4.1 255.255.255.0 
 ospf dr-priority 255
#
ospf 1 router-id 1.1.1.1 
 area 0.0.0.0 
  network 192.168.4.0 0.0.0.255
```

#### 对于访问外网和外网访问 
1. 允许 B 公司网络主机访问外网，可允许外网用户用 `101.1.2.111` 来访问 Server2
AR3：
```
acl number 2000  
 rule 5 permit source 192.168.30.0 0.0.0.255 
#
interface GigabitEthernet0/0/0
 ip address 101.1.2.2 255.255.255.0 
 nat static global 101.1.2.111 inside 192.168.30.253 netmask 255.255.255.255
 nat outbound 2000
#
ip route-static 0.0.0.0 0.0.0.0 101.1.2.1
```

ISP：
```
int GigabitEthernet0/0/1
 ip address 101.1.2.1 24
```
