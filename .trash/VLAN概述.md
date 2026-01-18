### 一、 产生的背景
电脑刚开启的时候和 DHCP 获取 ip 地址，会发送 ARP广播报文
#### 1. 安全问题
ARP的攻击
#### 2. 消耗链路带宽
一个交换机就是一个广播域，那么交换机收到广播报文，才去的行为是泛洪漫画，线网中部署了组播业务，交换机收到的组比报文也是泛洪
### 二、 什么是VLAN？
虚拟局域网，是用于交换机的广播域隔离技术，将交换网络逻辑地分成多个网络
1. ==VLAN只能在同一网段下通信，不同网段不能通信==
2. VLAN 需要 ID 来区分相应的广播域
3. 如果从一个 VLAN 接口收到的 BUM 帧，只能在该 VLAN 对应的 up 接口泛洪
4. 如果从 VLAN 收到一个单播帧，该 只会查找该 VLAN 对应的 mac 地址表，如果命中则转发，如果没有命中，则以未知单播帧泛洪处理
## 三、802.1Q 的封装
当交换机接收到数据时，会根据接口配置 VLANID，给数据帧打上一个 802.1Q 的标记

802.1Q 的组成
1. TPID：端口 ID，0X8100（固定值） 16 位长
2. Priority：用于 QOS，只有三位，描述优先级 0～7
3. CFI：在以太网中，这个值为 0
4. VLANID：12 位
	交换机可配置的 VLAN 数量为 0～4095，0 和 4095 保留没有使用，能使用的是 1～4094

## 四、 简单配置
1. 如果访问显示 Destination  host  unreachable 那么说明 arp 没有解析到 MAC
2. 如果访问显示 Requsest  timeout! 那么说明 arp 解析到了 MAC 但是没法通信
## 五、交换机接口的类型
==注意：进入交换机后的数据帧一定是带有VLAN TAG的==
==PVID 本质上就是给一个 untag 帧划分 一个 VLAN==
### 1.Access 口：
1. 定义：交换机上用来连接用户PC、服务器等终端设备的接口，Access接口通常只收发无标记帧
2. Access口对数据帧的处理：
	1. 收数据帧：
		-  有VLAN ID的数据帧：VLAN ID=PVID，进入交换机
		- 无VLAN ID的数据帧：数据帧打上PVID对应VLAN ID
	2. 发数据帧：
		- VLAN ID=10：剥离VLAN 10，从Access发出没有TAG的数据帧
		- VLAN ID≠10：丢弃
	- 案例1：同VLAN跨交换机的Access口通信
		每个与终端相连的接口下都配置
			`interface GigabitEthernet0/0/2`
			`port link-type access`
			`port default vlan 10(或者vlan 20)`
			![](DICM/Pasted%20image%2020251109200032.png)
			1. 首先，PC1发送了一个不带标记的数据帧到交换机，但是G0/0/2的Access口的VLAN ID是10（因为我们已经创建了VLAN10，并将G0/0/2划入），给这个数据帧打上标记后进入交换机，通过G0/0/1口发出后将数据帧的VLAN ID剥离
				 ==注意，这里当第一次发出的时候会ARP广播到PC2，但是因为ARP的目标不是自己所以会丢弃==
			2. 进入右侧交换机重复同样的情况，但是这个数据帧只会从G0/0/4的口发出，因为G0/0/2和G0/0/3口的VLAN ID是20，交换机只会把VLAN10的帧转发到VLAN10上
	- 案例2：不同VLAN跨交换机的Access口通信
		每个与终端相连的接口下都配置
			`interface GigabitEthernet0/0/2`
			`port link-type access`
			`port default vlan 10(或者vlan 20)`
			![](DICM/Pasted%20image%2020251109200438.png)
		- 首先，PC1发送了一个不带标记的数据帧到交换机，但是G0/0/2的Access口的VLAN是10（因为我们已经创建了VLAN10，并将G0/0/2划入），给这个数据帧打上标记后进入交换机，通过G0/0/1口发出后将数据帧的TAG剥离
			 ==注意，这里当第一次发出的时候会ARP广播到PC2，但是因为ARP的目标不是自己所以会丢弃==
		- 进入右侧交换机，因为UNTAG帧进入交换机的Access口的VLAN ID是20，所以这个数据会从所有VLAN是20的区域发出，但只会在目标终端回复（也就是PC5）
### 2.Trunk 口（PVID 有且只有一个）
1. 定义：Trunk 一般是用于交换机之间互联的，可以允许带有标签的数据帧在该链路上进行传递。==Trunk 接口的默认 PVID 等于 1==
2. Trunk 接口对数据帧的处理：
	1. 收数据帧:
		- 有VLAN ID的数据帧：看接口VLAN的允许通过列表，如果命中就放行，反之丢弃
		- 无VLAN ID的数据帧：根据Trunk的PVID打上VLAN ID，再看VLAN的允许通过列表，命中就放行，反之丢弃
	2. 发数据帧：（必定带VLAN ID）
		- 若帧的VLAN ID和端口的PVID一致，就会去掉标签（UNTAG）发送
		- 若不一致，则带标签（VLAN ID）发送
	- 案例一：配置 Trunk 口并分别让 VLAN 10 和 VLAN 20 下的两个终端通信
		==每个与终端相连的接口下都配置==
		`interface GigabitEthernet0/0/2`
		`port link-type access`
		`port default vlan 10(或者vlan 20)`
		![](DICM/Pasted%20image%2020251109201043.png)
		- 首先，PC1发送一个UNTAG数据帧，进入左侧交换机，因为自身所在是Access口的VLAN10，所以这个帧被打上了标记，从G0/0/1出来的时候，==帧的VLAN ID和端口的PVID不一致==，所以也是以VLAN ID=10发送过来
		- 右侧交换机收到VLAN ID是10的数据帧后，根据自己的放行列表判断将这个数据帧转发到VLAN ID为10的VLAN区域下，对应的目标主机会回复
