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
	![](DICM/Pasted%20image%2020251110141937.png)
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
	![](DICM/Pasted%20image%2020251110142041.png)
	- 案例一：配置 Trunk 口并分别让 VLAN 10 和 VLAN 20 下的两个终端通信
		==每个与终端相连的接口下都配置==
		`interface GigabitEthernet0/0/2`
		`port link-type access`
		`port default vlan 10(或者vlan 20)`
		![](DICM/Pasted%20image%2020251109201043.png)
		- 首先，PC1发送一个UNTAG数据帧，进入左侧交换机，因为自身所在是Access口的VLAN10，所以这个帧被打上了标记，从G0/0/1出来的时候，==帧的VLAN ID和端口的PVID不一致==，所以也是以VLAN ID=10发送过来
		- 右侧交换机收到VLAN ID是10的数据帧后，根据自己的放行列表判断将这个数据帧转发到VLAN ID为10的VLAN区域下，对应的目标主机会回复
### 3.Hybrid-混杂端口
定义：既可以实现Access的功能，也可以实现Trunk的功能，灵活的携带或者不携带TAG在链路上传递
	华为交换机端口默认是Hybrid状态，PVID=1，离开本交换机的时候，只允许VLAN1，并且移除VLAN1的标签
`port hybrid pvid vlan 1` ==注意：PVID 必须是已经存在的 VLAN 值==
`port hybrid untagged vlan 1 //离开交换机` 
Hybrid的PVID只在进入交换机但没打标记的时候有用，此时如果放行列表有和PVID一样的VLAN标签，那么就可以放行此untagged数据帧

1. 进入交换机接口：
	- 带有TAG：如果有带TAG的数据帧过来了，看是否在VLAN的允许通过列表，如果在就放行（`port hybrid untagged/tagged XXX`）
	- 没有TAG：先根据Hybrid的PVID，打上相应的TAG，然后查看是否在VLAN的允许通过列表，在就放行，反之丢弃
2. 离开交换机接口：
	这里的tagged和untagged都是针对出交换机是否带标记，tagged就是带着标记发送，untagged同理
	- 带有TAG：数据帧离开交换机端口，先查看是否在VLAN允许通过列表，在就看执行动作，如果是tagged就带有标签离开交换机
	- 没有TAG：数据帧离开交换机端口，先查看是否在VLAN允许通过列表，如果在就看执行动作，如果是untagged就不携带标签离开交换机
- 案例 1：Hybrid 混合端口入门演示
	![](DICM/Pasted%20image%2020251110144305.png)
	1. 首先 PC 1 发出不带标记的帧，到达 1 端口因为配置的是 `port hybrid pvid vlan 10`，所以被打上标记为 `10`，又因为允许通过的 VLAN 的 PVID 是 10，所以允许通过
	2. 进入交换机后，从 3 号口出来，因为允许通过的 VLAN 是 10 和 20，并且允许带着标记，所以带着自己的 10 标记进入右侧交换机
	3. 从右侧交换机的允许端口（1 端口）出来，并剥离标签
- 案例 2: PC1、PC2和PC3在同一个网段，PC1能访问PC2和PC3，但PC2和PC3隔离
	![](DICM/Pasted%20image%2020251110150526.png)
	- 这里让 PC 1 所在的交换机端口 PVID 是 10，放行 `10、20、30`
	- 让 PC 2 所在的交换机端口 PVID 是 20，放行 `10、20`
	- 让 PC 3 所在的交换机端口 PVID 是 30，放行 `10、30`
	1. 当 PC 1 发出数据帧，它被打上的 PVID 是 10，然后无论是端口 2 还是端口 3 都可以通过
	2. 当 PC 2 和 PC 3 发出数据帧，它只能与 PC 1 通信，与对方放行的 PVID 不符

## 六、VLAN的划分方式
### 1.根据端口
1. 根据交换机的接口来划分VLAN，实现隔离广播域，提高网络安全性
2. 缺点：移动性不好（例如接口1是VLAN10的PC1，接口2是VLAN20的PC2，期望是PC1与PC2的位置接口互换，VLAN也互换，==但基于端口划分无法做到==）
### 2.基于MAC划分
1. 根据PC或终端的MAC来划分交换机接口对应的VLAN
2. 优点：移动性好，VLAN划分跟着MAC走
	注意：为了让借口根据MAC来划分相应的VLAN，需要在接口下配置untagged相应的VLAN（可能会接入这个设备这个端口的所有VLAN）
### 3.基于协议
IPv4或IPv6
### 4.基于子网
根据PC或者终端的ip网段来划分相应的VLAN