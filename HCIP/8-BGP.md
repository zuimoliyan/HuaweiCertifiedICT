# 一、BGP 的基本概述
## 1、AS
`autonomous sysem` 自治系统，一个组织或一个机构管理的 IP 网络的集合
	例如：电信、联通、移动
- 为了区分不同的自治系统，使用 AS 号来进行区分。`AS1`~`AS65535`
## 2、BGP 的邻居的分类
### 1. EBGP 的邻居
建立 BGP 邻居的设备和自身<font color="#ff0000">不在同一个 AS</font>，建立 EBGP 邻居的时候，一般使用**直连接口**建立
### 2. IBGP 的邻居
建立 BGP 邻居的设备和自身<font color="#ff0000">在同一个 AS</font>，建立 EBGP 邻居的时候，一般使用**环回口**建立

## 3、BGP 的封装
1. 封装在TCP 之上，端口号是179，BGP 是单播建立邻居，从命令行体现需要指定对端的 IP 地址，`peer x.x.x.x as-number xx`
	TCP 是可靠的（三次握手、有确认和重传机制）
2. BGP 是基于属性开发的（属性的基本单元是 TLV），BGP 不仅可以传递传统意义上的路由，还可以传拓扑、链路状态、路径。

## 4、BGP 的报文--5种报文
1. `open` 报文：类似于 OSPF 的 Hello，用来协商参数，<font color="#ff0000">open 报文正常情况不会周期发送</font>。
	查看 `OPEN Message` 报文
	![|600](DICM/Pasted%20image%2020260110154634.png)
	协商内容：
	- RID
	- 版本：v4
	- HD （老化时间）：180秒
	- AS 号
	- 能力
1. `KA` 报文：类似于 OSPF 的 Hello，周期性发送，每60秒发送一次，180秒老化。用于邻居的保活。
`open` 报文 + `KA` 报文 = OSPF 的 `Hello`
2. `update` 报文：路由更新报文，用来传递或者撤销 BGP 的路由
3. `notification` 报文：当 `open` 报文协商错误，或者通告 BGP 路由错误的时候
4. `refresh` 报文：刷新报文，当在邻居的入方向使用 `router-policy`，会发送 `refresh`，为了加快路由策略生效

### 5. BGP 的简单配置***

#### 案例1：简单的 BGP 建立连接
![|750](DICM/Pasted%20image%2020260110153458.png)

AR1 的完整配置
```
interface GigabitEthernet0/0/0
 ip address 10.1.12.1 255.255.255.0
#
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255
#
ip route-static 2.2.2.2 255.255.255.255 10.1.12.2
#
bgp 100
 router-id 1.1.1.1
 peer 2.2.2.2 as-number 200
 peer 2.2.2.2 connect-interface LoopBack0 
 peer 2.2.2.2 ebgp-max-hop 2
```


##### 重点
1. `peer 2.2.2.2 as-number 200`：对端的 AS 号是200，IP 地址是`2.2.2.2`
2. `peer 2.2.2.2 connect-interface LoopBack0`
	- 如果是该设备去到对端设备，那么以 `lo0` 的地址为源，以 `2.2.2.2` 为目的
	- 如果是对端设备去到该设备，那么对端的源必须是 `2.2.2.2` ，目的地址必须是 `lo0` 的地址，<font color="#ff0000">不然邻居无法建立</font>
3. `peer 2.2.2.2 ebgp-max-hop 2`
	- EBGP 用 LoopBack 建邻居必须配置 `ebgp-max-hop N`（N≥2），突破 “直连” 限制
	- 即：当 `ebgp-max-hop 1` 时，路由器会触发直连检查，那么此时目的地址必须是直连接口，否则就会失败，但当 `ebgp-max-hop N`（N≥2）就不会触发检查
因为这个案例是用环回口配置的，如果是直连接口那么 `ebgp-max-hop 1` 就行

#### 案例2：IBGP 与 EBGP 的建立
流程：先在 IBGP 内部配置 OSPF、BGP，再在 IBGP 与 EBGP 的连接配置 BGP 路由、并配置 `next-hop-local`
![|800](DICM/Pasted%20image%2020260112175813.png)
AR1
```
interface GigabitEthernet0/0/0            
 ip address 12.1.1.1 255.255.255.0        
#                                                                               
interface GigabitEthernet0/0/2            
 ip address 13.1.1.1 255.255.255.0  
#
interface LoopBack0                       
 ip address 1.1.1.1 255.255.255.255       
#                                         
interface LoopBack100                     
 ip address 100.1.1.1 255.255.255.255     
#
bgp 100                                   
 router-id 1.1.1.1                        
 peer 2.2.2.2 as-number 100               
 peer 2.2.2.2 connect-interface LoopBack0 
 peer 3.3.3.3 as-number 100               
 peer 3.3.3.3 connect-interface LoopBack0 
#
ospf 1 router-id 1.1.1.1                  
 area 0.0.0.0                             
  network 1.1.1.1 0.0.0.0                 
  network 12.1.1.1 0.0.0.0                
  network 13.1.1.1 0.0.0.0                
```

AR2：只写重点
```
bgp 100                                   
 router-id 2.2.2.2                        
 peer 1.1.1.1 as-number 100               
 peer 1.1.1.1 connect-interface LoopBack0 
 peer 24.1.1.4 as-number 200                
 peer 1.1.1.1 next-hop-local                                    
```
AR3：不写了
AR4：环回口宣告 `4.4.4.4`，只有在 AR2和 AR3上配置 `next-hop-local`，才能在 AR1有对应的路由
```
interface LoopBack0                       
 ip address 4.4.4.4 255.255.255.255       
# 
bgp 200                                   
 router-id 4.4.4.4                        
 peer 24.1.1.2 as-number 100              
 peer 34.1.1.3 as-number 100                                                                                      
 network 4.4.4.4 255.255.255.255         
```
# 二、BGP 的三张表

## 1、邻居表

**BGP Notification（通知报文）** 是 BGP 协议的四种核心报文之一，作用是当 BGP 对等体之间检测到**错误**时，向对方发送该报文来通告错误原因，并**关闭 BGP 连接**
![](DICM/Pasted%20image%2020260112153329.png)
![|600](DICM/Pasted%20image%2020260110164755.png)
1. `Idle` 状态：空闲状态，等 start 事件（默认等待32秒），等待对端设备预配的时间。
	1. 在 `start` 结束之前，自己不会朝着对方建立 TCP 连接
	2. 如果收到对端的 TCP 连接不会响应
	3. 如果没有 peer 的 IP 地址的路由时，就会一直卡在 `Idle` 状态
	可以通过命令查看
	![|400](DICM/Pasted%20image%2020260110165041.png)
2. `connect` 状态：`start` 结束后，朝着对端发起 TCP 连接
	- 如果 TCP 三次握手连接成功，状态迁移到 `opensend` 状态
	- 如果 TCP 连接失败，迁移到 `active` 状态
	- 如果对端没有运行 BGP，或者中间设备没有路由（去程或者回程路由）或者中间的设备做了包过滤，那么就一直停留在 <font color="#ff0000">connect</font>
3. `active` 状态：TCP 连接失败时，会尝试多次链接，如果连接成功，状态迁移至 `opensend` 状态；如果连接失败迁移到 `connect` 状态
	- 对端的 BGP 的 peer 配置错误或者没有配置，那么就会卡在 `connect` 和 `active` 之间来回切换
4. `opensend` 状态：TCP 三次握手成功，发送 open 报文，病等待对端的open 报文
	- 如果收到了对端的 open 报文，参数协商成功，那么状态迁移到 `openconfirm` 状态
	- 如果参数协商失败，状态回到 `idle`
5. `openconfirm` 状态：收到对端的 open 报文，并且参数协商成功，等待对端的 `KA`。<font color="#ff0000">如果收到对端的 notification，那么回到 idle 状态</font>
6. `establish` 状态：BGP 的最终状态，接收 KA 报文或者 refresh，<font color="#ff0000">也有可能收到 notification，如果收到就回到 idle 状态</font>
## 2、BGP 的 RIB
### 1. 路由的生成
1. network
2. import（引入）
3. 路由的汇总
### 2. BGP 的属性
1. 公认必遵属性：所有的 BGP 的对等体都能够识别并遵守该属性，如果没有携带该属性，那么对端不会接收该路由，并发送 notification
	- as-path
	- 起源属性
	- next-hop
2. 公认任意：所有的 BGP 对等体都能识别，但是如果不携带也不会报错
	- local-preference
	- 原子聚合属性
3. 可选过渡：BGP 的对等体可以识别也可以不识别的属性，但是不管是否识别都只会传递给自己的邻居
	- 团体属性
4. 可选非过渡：BGP 的对等体可以识别也可以不识别的属性，如果识别才会传递，不识别不传递
	- MED
	- cluster-list
	- originatorID

#### 补充：

##### 团体属性
用来描述相同特征的 BGP 路由的属性，类似于 IGP 的 tag
- 格式是：`AA:NN(AS Number)`
- 例如 AS 是100，设备的编号是3，那么这个设备发出的 BGP 路由，带上团体属性 `100:3`，便于对路由的控制（路由过滤、属性修改等）
##### 团体属性分类
###### 1.预定义（公认团体属性）：可以控制路由传递
1. no advertise：如果收到一条携带该属性的路由，那么<font color="#ff0000">不会将该路由传递给自己的任何 BGP 邻居</font>
		![|750](DICM/Pasted%20image%2020260112170058.png)
2. no export：如果收到一条携带该属性的路由，那么<font color="#ff0000">不会将该路由传递给自己的 EBGP 邻居</font>，但是会传给 IBGP 邻居
		![|750](DICM/Pasted%20image%2020260112170314.png)
3. no subconfed export
	
4. internet

配置案例
![|700](DICM/Pasted%20image%2020260112175008.png)
AR4上配置
```
route-policy community permit node 10 
 apply community no-export 
#
BGP 100
 peer 24.1.1.2 route-policy community export
 peer 24.1.1.2 advertise-community
```
<font color="#ff0000">AR2上配置</font>：如果不配置，那么 `4.4.4.4` 从 AR4传到 AR2再到 AR1，此时 AR1会把 `4.4.4.4` 传给 AR3
```
[AR2-bgp]peer 1.1.1.1 advertise-community
```

<font color="#ff0000">配置之前</font>
	![|600](DICM/Pasted%20image%2020260112174929.png)
<font color="#ff0000">AR2不配置</font>
	![|600](DICM/Pasted%20image%2020260112175324.png)
<font color="#ff0000">全都按规定配置</font>
	![|600](DICM/Pasted%20image%2020260112174954.png)
###### 2.自定义（管理员自己定义）

##### 添加团体标记、基于团体做路由过滤实操
###### 案例1：给指定路由添加团体属性
![|700](DICM/Pasted%20image%2020260113155736.png)
AR1：
```
ip ip-prefix 11-PREFIX index 10 permit 11.11.11.11 32
#
route-policy COMMUNITY_ADD permit node 10 
 if-match ip-prefix 11-PREFIX 
 apply community 4000:1 5:1
#
bgp 100
 router-id 1.1.1.1
 peer 12.1.1.2 as-number 200 
 #
  network 1.1.1.1 255.255.255.255 
  import-route static route-policy COMMUNITY_ADD
  peer 12.1.1.2 enable
  peer 12.1.1.2 advertise-community
  peer 12.1.1.2 advertise-ext-community
```

在 AR2上查询：`dis bgp routing-table community 4000:1 5:1`
	![|750](DICM/Pasted%20image%2020260113155944.png)

###### 案例2：基于团体属性过滤路由
![|750](DICM/Pasted%20image%2020260113162923.png)
```
ip community-filter 10 permit 4000:1
#
route-policy COMMUNITY_FILTER permit node 10 
 if-match community-filter 10 whole-match 
#
route-policy COMMUNITY_FILTER deny node 20
#
bgp200
  peer 12.1.1.1 route-policy COMMUNITY_FILTER import
```
配置后 AR1向 AR2发送的路由就被拦截
	![](DICM/Pasted%20image%2020260113162847.png)


补充：`if-match community-filter 10 whole-match` 和 `ip community-filter 10 permit 4000:1` 的知识点
1. `community-filter` 后必须跟**过滤列表编号**，而不能直接跟团体值
2. `whole-match` 代表的是严格匹配，例如 AR1传给 AR2是 `4000:1  5:1` 但 AR2设置严格匹配，所以拒绝接收
### 3. 路由的发送***（<font color="#ff0000">11条选路规则</font>）
1. `*`：`valid` 有效的
	- 条件是 BGP 路由的下一跳可达，判断下一跳是否可达就是<font color="#ff0000">看路由表中有没有这个下一跳对应的路由（默认路由除外）</font>
	- 如果 BGP 路由是无效的，那么肯定不可能最优
2. `>`：最优的----对于网络号和掩码完全相同的路由，如果存在多条 BGP 的该路由，一定永远只有一条最优
	- 首先 BGP 路由需要有效
	- 必须满足IBGP同步-----<font color="#ff0000">华为默认关闭了IBGP同步，永远打不开</font>
		解释：一个路由器从IBGP 邻居学到的路由信息，除非这条路由也能在本地的 IGP（内部网关协议）路由表中找到，否则不会传递给任何 EBGP 邻居
		![|700](DICM/Pasted%20image%2020260111161041.png)
#### 1、BGP 的下一跳
![|750](DICM/Pasted%20image%2020260111170221.png)
1. 从 IBGP 邻居学习来的路由，传递给自己的 EBGP 邻居的时候，下一跳变成自己（更新源地址）
	如图在 AR4上查看 `1.1.1.1` 和 `100.1.1.1`
		![](DICM/Pasted%20image%2020260111170349.png)
2. 从 EBGP 邻居学习来的路由，传递给自己的 IBGP 邻居的时候，下一跳不变
	如图在 AR1上查看 `4.4.4.4`
		![](DICM/Pasted%20image%2020260111170435.png)
3. 从 EBGP 邻居学习来的路由，传递给自己的 EBGP 邻居的时候，下一跳变成自己
#### 2、11条选路规则
以下内容均以此拓扑为例
![|800](DICM/Pasted%20image%2020260112114359.png)

##### 0.BGP 路由必须是有效的
BGP 的下一跳可达

##### 1.协议首选值 `pref-val`
<font color="#245bdb">华为私有属性</font>，用于离开本设备该属性设备内部生效，不能通过 BGP 的 `update` 携带该属性。<font color="#ff0000">默认是0，越大越优先</font>

- 使用 `router-policy` 修改该属性，配置的时候只能用在邻居的入方向，<font color="#ff0000">不能在出方向</font>
	配置命令如下
	![](DICM/Pasted%20image%2020260111162956.png)
	配置之后，配置的设备会向目标设备发送 `refresh` 报文
拓展：
- 通过 `dis bgp routing-table x.x.x.x` 查看
- 如下图中的 `pref-val 0`，没有优选的原因是 `not preferred for router ID`
	![](DICM/Pasted%20image%2020260111162043.png)



##### 2. `local-preference`
  本地优先级，用于离开本 AS 的，该属性是公认任意属性，只能在一个 AS 内传递不能传出本 AS，<font color="#ff0000">默认值100，越大越优</font>
  
该属性用在 IBGP 邻居的出和入方向，或者是 EBGP 邻居的入方向
- 在 AR2上的配置示例
	![](DICM/Pasted%20image%2020260111185027.png)
如图所示：
- `4.4.4.4`是 AR4通过 EBGP 传入 AR2的，所以它没有 `local-preference`，（虽然没有，但是自己默认是100）
- 但是`8.8.8.8`和 `100.1.1.1` 是通过 IBGP 传来的，所以默认就有
![|600](DICM/Pasted%20image%2020260111164428.png)

##### 3.自己始发的路由，优于 `EBGP` 和 `IBGP` 邻居学习来的路由
同时<font color="#6425d0">对于从 IBGP 和 EBGP 同时学到的路由，优选 IBGP</font>

- 对应自己始发的路由的优先级：`aggregate(手动汇总)` > `auto summary` > `network` > `import`
1. 对于<font color="#ff0000">自身路由优于引入路由</font>的实验配置：
	- 在 AR1上引入 `ip route-static 4.4.4.4 32 NULL 0`，并在 AR1的 `bgp100` 引入静态路由
	- AR4在 `LookBack 0` 配置 `ip address 4.4.4.4 255.255.255.255`
	此时在 AR2和 AR3上
		![|500](DICM/Pasted%20image%2020260112111052.png)
	此时在 AR4上
		![|500](DICM/Pasted%20image%2020260112111147.png)
2. 对于<font color="#ff0000">始发路由内的优先级</font>的实验配置
	- 直接以 AR1举例：
		1. `ip route-static 10.1.0.0 24 NULL 0`
		2. `ip route-static 10.0.0.0 8 NULL 0`
		注意：<font color="#ff0000">上面两条用于做手动、自动汇总</font>
		引入静态路由 `import-route static`
		3. `[AR1-bgp]summary automatic`        配置自动汇总
		4. `[AR1-bgp]network 10.0.0.0 8`      配置 `network`
		5. `[AR1-bgp]aggregate 10.0.0.0 8`    配置手工汇总
	最终结果
		![](DICM/Pasted%20image%2020260112112527.png)

##### 4. `AS-path`
该属性用于记录路由的传递过程中经历的 as 号的有序集合，用于选路和防环。`AS-path` 是<font color="#ff0000">公认必遵属性</font>

- `AS-path` 用于选路，那么<font color="#ff0000">短的优于长的，从右往左记录</font>。`AS-path` 可以用在<font color="#ff0000"> IBGP 和 EBGP 邻居的入和出方向</font>
配置案例：在 AR4的入方向配置 AR2进入带上 `AS-path`
![](DICM/Pasted%20image%2020260112114117.png)
此时查看 AR4的 bgp 路由表
	AR2传过来的路由被额外套上了 as-path 为789的标记，而 AR3则默认只有 bgp100的标记，所以<font color="#ff0000">优选 AR3的路径</font>
	![](DICM/Pasted%20image%2020260112114210.png)
`[AR4-bgp]bestroute as-path-ignore `： <font color="#ff0000">此时可以这条命令让选择最优路由时忽略 as-path</font>
	![|500](DICM/Pasted%20image%2020260112114715.png)

##### 5.起源属性
origin，表示该 BGP 路由是通过什么方式生成的。该属性是<font color="#ff0000">公认必遵属性</font>
生成方式有下面几种：
1. network（IGP---> `i`）
2. import-route （EGP---> `e`）
3. import（incomplete---> `?`）
优先级 `i` > `e` > `?`

配置案例：
在 AR1的入口配置将 AR4的 `4.4.4.4` 的路由修改起源属性
```
[AR1]ip ip-prefix 4 permit 4.4.4.4 32       # 定义前缀列表：仅匹配4.4.4.4/32路由
#
[AR1]route-policy inc permit node 10        # 创建inc策略，节点10允许通过
[AR1-route-policy]if-match ip-prefix 4      # 匹配前缀列表4（即4.4.4.4/32）
[AR1-route-policy]apply origin incomplete   # 把匹配路由的Origin改为?
#
[AR1]route-policy inc permit node 20        # 路由策略inc节点20（低优先级）：放行所有其他路由（防默认拒绝）
#
[AR1]bgp 100
[AR1-bgp100]peer 2.2.2.2 route-policy inc import     #引入策略
```
在 AR 的环回口：
```
interface LoopBack0
 ip address 4.4.4.4 255.255.255.255 
 ip address 44.44.44.44 255.255.255.255 sub
```

查看 AR1的 bgp 路由表------对于匹配的 `4.4.4.4`，把从 AR2传过来的，起源属性从 `i` 跟改为 `?`，因为 `i` 优先级大于 `?`，所以选 AR3
	![|650](DICM/Pasted%20image%2020260112144309.png)

补充：
```
[AR1-route-policy]apply origin ? 
 egp           对应 Origin 属性为 e（Exterior） 
 igp           对应 Origin 属性为 i（Interior） 
 incomplete    对应 Origin 属性为 ?（Incomplete）
```

##### 6.MED 属性
`multi exit discriminator`，类似于 IGP 的 COST，发布 BGP 路由的时候，会将 IGP 的开销携带在 BGP 的 MED 属性中
MED 可以在两个 `AS` 之间传递，但是
1. 从<font color="#ff0000"> IBGP 邻居学习来的路由传递给 EBGP 邻居时，MED 值无法携带</font>
2. 从 <font color="#7030a0">EBGP 邻居学习来的路由传递给 IBGP 邻居时，MED 值可以携带</font>

###### MED 的特性
1. MED 只能在两个 `AS` 之间传递，不能传递到第三个 `AS`
2. 从 IBGP 邻居学来的路由传递给 EBGP 邻居的时候，MED 属性<font color="#ff0000">不携带</font>
3. 从 EBGP 邻居学来的路由传递给 IBGP 邻居的时候，MED 属性<font color="#ff0000">可以携带</font>
4. 从不同的 `AS` 学来的 EBGP 路由默认不比较 MED 值
	`[AR4-bgp]compare-different-as-med` 执行这条命令就可以比较了
###### 案例1：在 EBGP 的入方向修改 MED 属性
配置命令：这里是默认匹配 AR2的所有路由
```
[AR4]route-policy med permit node 10   # 创建名为med的路由策略，节点10，匹配的路由允许通过
[AR4-route-policy]apply cost 10        # 对匹配此节点的路由，设置MED值为10（影响BGP选路优先级）
#
[AR4]bgp 200
[AR4-bgp]peer 24.1.1.2 route-policy med import
```
此时查看 AR4的 bgp 路由表
	![|600](DICM/Pasted%20image%2020260112150851.png)

补充：如果想匹配指定路由
	路由策略只匹配 4.4.4.4 这个路由前缀
```
# 1. 创建IP前缀列表，命名为prefix_4444，仅匹配4.4.4.4/32（主机路由） 
ip ip-prefix prefix_4444 index 10 permit 4.4.4.4 32 

# 2. 创建路由策略med，节点10，仅匹配前缀列表prefix_4444中的路由 
route-policy med permit node 10 
if-match ip-prefix prefix_4444     // 核心：只匹配4.4.4.4 
apply cost 10                      // 仅对4.4.4.4设置MED值为10 

# 3. （可选）添加拒绝所有其他路由的节点（防止默认放行） 
route-policy med deny node 20

#
[AR4]bgp 200
[AR4-bgp]peer 24.1.1.2 route-policy med import
```

###### 案例2：在 IBGP 的出方向修改 MED 属性
配置命令：这里是让 AR2在出接口方向配置将所有发往 AR4的路由都把 MED 修改
```
[AR2]route-policy med permit n 10
[AR2-route-policy]apply cost 20
#
[AR2]bgp 100
[AR2-bgp]peer 24.1.1.4 route-policy med export
```
在 AR4上查看 bgp 路由表
	![|600](DICM/Pasted%20image%2020260112151937.png)

抓包
	![|600](DICM/Pasted%20image%2020260112151635.png)

##### 7. EBGP 邻居学来的路由要优于 IBGP


##### 8.1 BGP 的 `next-hop` 开销小的

##### 8.2 BGP 的负载分担
1. 当前面八条规则都无法优选的时候
2. `AS-path` 完全一致（不仅仅指长度一致，还要 as 号的排序一模一样）
	- 但是如果配置 `bestroute as-path-ignore`（忽略 as-path），那么就<font color="#ff0000">仍然可以负载分担</font>

配置命令：`[AR1-bgp]maximum load-balancing 2`    最后的数字是 <font color="#ff0000">BGP 负载均衡的路径数量</font>
	配置之前
		![|600](DICM/Pasted%20image%2020260112163638.png)
	配置之后
		![|600](DICM/Pasted%20image%2020260112163707.png)

##### 9. cluster-list
路由器经过的簇 ID 的列表，越短越优，由 RR 添加，若长度相同，则比较 Cluster List 的字典序（越小越优）
##### 10. criginatorID/RID（始发者 ID / 路由器 ID）
- Originator ID：RR 环境中，标识**最初发布这条路由的路由器 ID**
	- 若收到的路由包含 Originator ID，且与本地 Router ID 相同，则丢弃（防环）；选路时，优先选择**Originator ID 数值更小**的路由
- Router ID（RID）：若没有 Originator ID，则比较**路由的 “始发路由器 ID”**（即发布这条路由的 BGP 设备的 Router ID），数值越小越优。
##### 11. PEER IP 地址（邻居 IP 地址）
当前面 10 条规则都无法区分路由优先级时，作为**最终的 “决胜条件”**。
	优先选择**发送这条路由的邻居 IP 地址更小**的路由

### 4. 路由反射器--PR

##### 簇--cluster
一个簇由 RR 和它的客户端构成。使用簇 ID 来标识这个簇，默认簇 ID 等于 RR 设备的 RID
##### 客户端
客户端是和 RR 建立反射的 IBGP 邻居关系
##### 非客户端
非客户端是和 RR 建立普通的 IBGP 邻居关系
##### RR
既不是客户端也不是非客户端的 IBGP。<font color="#ff0000">只要指定 IBGP 邻居是客户端，那么这个设备立刻称为 RR</font>

配置 `x.x.x.x` 作为 RR 的客户端
配置命令：`peer x.x.x.x reflect-client`


### 5. BGP 内部处理机制


### 6. BGP 路由不加表
1. BGP 不加 BGP 的 RIB 表
	- 从 EBGP 邻居学习到的路由，如果 as-path 中的 as 号和接收设备一致，那么不能接收
	- 从 IBGP 邻居学习到的路由 originatorID 和自己的 RID 相同
	- 从 IBGP 邻居学习到的路由 cluster-list 有自己的 clusterID 时
	- 路由条目的限制
		`[AR2-bgp]peer 12.1.1.1 route-limit <1-80000>  < alert-only | idle-forever | idle-timeout >`
	- as-path 超过限制
	- 配置路由过滤
2. BGP 不加 IP路由表
	- RR 配置了 `BGP RIB ONLY`，不加 IP RIB
		`[AR2-bgp]bgp-rib-only`，此时 BGP 表会有路由，但是路由表没有
		
	- 开启了 BGP 的抑制  `D 和 H 置位`
		当一条 BGP 路由的可达性频繁变化（比如 `up` 和 `down` 反复切换），会导致全网 BGP 设备反复更新路由、消耗大量资源，这种现象叫**路由震荡**
		- D 置位：路由因震荡被抑制，当前惩罚值高于抑制阈值，不进路由表
		- H 置位：路由曾被抑制，惩罚值正在衰减（低于抑制阈值，但未到重用阈值），不进路由表
		当惩罚值降到**重用阈值**（默认 750），路由会恢复正常，重新进入 `valid` 状态，加入路由表
		
	- BGP 下一跳不可达
	- 将自己建立邻居的源地址发布到 BGP
	- 通过其他协议学习到该路由
		只有自己的 BGP 路由表有这条路由，才会加 IP 表


## 3、IP路由表
除了以下情况
![](DICM/Pasted%20image%2020260113170723.png)

# 三、BGP 的特性

## 路由汇总

1. 目的：减少 BGP 的路由条目
2. 配置注意点
	- 汇总抑制明细---减少路由条目
	- 配置 as-set 防环


### 案例1：简单的路由汇总实验
![](DICM/Pasted%20image%2020260113183724.png)

配置汇总之前：
	![|750](DICM/Pasted%20image%2020260113183020.png)

<font color="#ff0000">配置汇总命令 </font>`[AR2-bgp]aggregate 8.8.8.0 24 detail-suppressed` 后
	![|750](DICM/Pasted%20image%2020260113183106.png)

<font color="#ff0000">此时可能会有环路风险</font>，我们可以配置 `[AR2-bgp]aggregate 8.8.8.0 24 detail-suppressed  as-set`
	![|750](DICM/Pasted%20image%2020260113183245.png)
#### 补充1：抑制策略
此时查看 AR1上是没有 AR3上的 `8.8.8.5和8.8.8.6` 的路由，那如果我们需要 `8.8.8.1` 和 `8.8.8.5` 两个互访怎么办？
	![](DICM/Pasted%20image%2020260113183921.png)

配置：抑制8.8.8.2和8.8.8.6，其他正常发布
```
ip ip-prefix 15 index 10 permit 8.8.8.6 32   #抑制的路由条目
ip ip-prefix 15 index 20 permit 8.8.8.2 32
#
route-policy sup permit node 10
 if-match ip-prefix 15
#
aggregate 8.8.8.0 255.255.255.0 as-set suppress-policy sup
```
查看 AR2的 BGP 路由表
	![|750](DICM/Pasted%20image%2020260113185114.png)
AR1查询 AR3的 `8.8.8.5` 路由
	![|750](DICM/Pasted%20image%2020260113185150.png)
成功