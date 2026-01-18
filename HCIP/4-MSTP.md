## 一、RSTP 的缺点
### 1.单棵树
如果有环那么阻塞的接口是无法转发报文，就浪费链路和接口的资源。平时的时候是闲置的
### 2.次优路径
在 VRRP 场景，汇聚交换机作为网关，在两台交换机上进行网关的负载分担，会出现部分以 SW2为网关的主机访问外网的流量是次优路径
![](DICM/Pasted%20image%2020251203163703.png)


### 3.可能 trunk 链路的放行 VLAN 配置，导致链路不通
#### 案例1：根据下图回答四个问题
![](DICM/Pasted%20image%2020251203164357.png)
##### 1. 在不配置生成树 STP 的情况下，SW1和 SW2是否有环路？
在不配置生成树STP的情况下，SW1和SW2 **没有环路**​
##### 2.在不配置生成树 STP 的情况下，PC1是否能访问 PC3？PC2是否能访问 PC4？
都能访问
##### 3. 在 STP 情况下，会不会阻塞哪台交换机的哪个接口？
会阻塞 SW2的1端口，原因如下
- **Port1 (G0/0/1)**：`Designated Bridge/Port : 0.4c1f-ccce-0392 / 128.2`，这表示SW2从这个端口收到的BPDU，是由桥MAC地址为 `0.4c1f-ccce-0392`的交换机（即根桥SW1）从它的 **端口128.2**​ 发送出来的。
- **Port2 (G0/0/2)**：`Designated Bridge/Port : 0.4c1f-ccce-0392 / 128.1`，这表示SW2从这个端口收到的BPDU，是由同一个根桥SW1从它的 **端口128.1**​ 发送出来的。
所以，SW2选择自己的 **GigabitEthernet 0/0/2**​ 端口作为根端口，<font color="#ff0000">SW2的1端口被阻塞</font>
##### 4. 在STP 情况下，哪几台 PC 不能互访？
由上面的可知，VLAN20所在的线路被阻塞，所以 PC2和 PC4不能互访


## 二、MSTP
### 1. 单域
#### 满足四个条件才能在一个<font color="#ff0000">单域</font>
1. `VLANID` 和实例映射要一模一样：比对 HASH 值--MD5
2. 域名要一样--默认是自己的 MAC 地址
3. 修订版本要一样--默认是0，多增加一个验证维度，防止 MD5一样
4. 运行 MSTP

#### 案例1：单域的实操
![](DICM/Pasted%20image%2020251203175447.png)
- 所有交换机都要配置的命令
	```
	vlan batch 10 20 30 40
	interface GigabitEthernet0/0/1
	 port link-type trunk
	 port trunk allow-pass vlan 10 20 30 40
	interface GigabitEthernet0/0/2
	 port link-type trunk
	 port trunk allow-pass vlan 10 20 30 40
	 
	 stp region-configuration
	 region-name huawei
	 instance 1 vlan 10 20
	 instance 2 vlan 30 40
	 active region-configuration     //必须激活，不然不生效
	```

- SW1的配置
```
stp instance 1 priority 0
stp instance 2 priority 4096
```
- SW2的配置
```
stp instance 1 root secondary
stp instance 2 root primary 
```
- SW3不配置优先级的话
<font color="#ff0000">注意</font>：`stp instance 1 priority 0` 等同于 `stp instance 1 root primary `；`stp instance 2 priority 4096` 等同于 `stp instance 1 root secondary`


##### 最终结果
###### 1.展示 MST 域配置信息
```
[Huawei]display stp region-configuration
 Oper configuration
   Format selector    :0             
   Region name        :huawei      //区域名称        
   Revision level     :0           //修订级别

   Instance   VLANs Mapped         //VLAN与实例的映射关系
      0       1 to 9, 11 to 19, 21 to 29, 31 to 39, 41 to 4094
      1       10, 20
      2       30, 40
```

###### 2. 展示抓包结果
![](DICM/Pasted%20image%2020251203180334.png)
因为开了两个域分别是 SW1和 SW2当根桥，所以会有两个设备发 STP 报文

1. SW1发的 STP 包
	![](DICM/Pasted%20image%2020251203180525.png)

2. SW2发的 STP 包
	![](DICM/Pasted%20image%2020251203180807.png)

3. 通过抓包 SW1下的 STP 报文可知
	- 域1的优先级是0
	- 域2的优先级是1（**内部存储的优先级字段 = 配置的优先级值 / 4096**，即：4096/4096 = 1）
	![](DICM/Pasted%20image%2020251203181202.png)


### 2. 多域
#### 术语
1. 总根：CIST中的桥ID（Priority+MAC）最小的，各个域中所有的IST中（实例o）的BID最小的
2. IST：域内实例O的那棵树。每个域都有一个IST，IST=MST10
3. CST：域间形成那棵树
4. CIST：域内的IST和域间的CST形成的一棵大树
5. MSTI：域内的每个实例形成的那棵树
6. IST的域根：是非总根所在的域，到总根距离最近的交换机
7. 域边缘设备：连接其他域的交换机
8. 域边缘端口：域和域之间交换机互联的接口
9. Master：总根和IST的域根的集合
10. Master端口：IST的RP端口就是其它实例的MAST端口

#### MSTP 多域的 BPDU 报文 
![](DICM/Pasted%20image%2020251204120938.png)


#### 案例1：多域的基本认识
![](DICM/Pasted%20image%2020251204124520.png)




##### 配置命令：
###### LSW1：
```
SW1：
vlan batch 10 20 30 40
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
stp region-configuration
 region-name huawei
 instance 1 vlan 10 20
 instance 2 vlan 30 40
 active region-configuration
#
stp instance 1 priority 0
stp instance 2 priority 4096
```
###### LSW2：
```
vlan batch 10 20 30 40
#
stp instance 1 root secondary
stp instance 2 root primary
#
stp region-configuration
 region-name huawei
 instance 1 vlan 10 20
 instance 2 vlan 30 40
 active region-configuration
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/3
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
```

###### LSW3：
```
vlan batch 10 20 30 40
#
stp priority 0
#
stp instance 1 root secondary
stp instance 2 root primary
#
stp region-configuration
 region-name huawei
 instance 1 vlan 10 20
 instance 2 vlan 30 40
 active region-configuration
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/3
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
```

###### LSW4：
```
vlan batch 10 20 30 40
#
stp priority 8192
#
stp instance 10 root primary
stp instance 20 root secondary
stp instance 0 priority 8192
#
stp region-configuration
 region-name h3c
 instance 10 vlan 10 20
 instance 20 vlan 30 40
 active region-configuration
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/3
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
```

###### LSW5：
```
vlan batch 10 20 30 40
#
stp priority 4096
#
stp instance 0 priority 4096
#
stp region-configuration
 region-name h3c
 instance 10 vlan 10 20
 instance 20 vlan 30 40
 active region-configuration
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/3
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
```

###### LSW6：
```
vlan batch 10 20 30 40
#
stp instance 10 root secondary
stp instance 20 root primary
#
stp region-configuration
 region-name h3c
 instance 10 vlan 10 20
 instance 20 vlan 30 40
 active region-configuration
#
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
#
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 40
```
## 、相关命令
`display stp region-configuration`：显示完整的 MST 域配置信息，包括实例与 VLAN 的映射关系
