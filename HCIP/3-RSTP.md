## 一、加快收敛
<font color="#ff0000">写在开头：拓扑改变的唯一标准就是非边缘端口的Forwarding</font>
### 1. 边缘端口
接入终端设备的接口，或者连接三层网络设备的端口，可以设置为边缘端口。一旦设置为边缘端口，接口 UP 马上进入转发状态不需要等待30秒
- 配置命令（<font color="#ff0000">接口下配置</font>）
```
stp edged-port enable
```
- 边缘端口的特性：
	1. 边缘端口一旦 UP，那么马上从 discarding 迁移到 Forwarding 状态
	2. 边缘端口如果收到BPDU报文，将失去边缘端口特性，会进行RSTP的计算
	3. 如果交换机收到TC置位的BPDU报文，那么不会朝着边缘端口发送
	4. 边缘端口会周期性发送BPDU报文
	5. 边缘端口forwading不会产生TC置位的BPDU报文
	6. 交换机收到了TC置位的BPDU报文，不清除边缘端口的MAC

### 2. 根端口快速切换
当交换机的根端口故障，AP 马上转为 RP，并且进入到转发状态

### 3. AP 对次级 BPDU 的回应
RSTP 中，如果 AP 收到了对端发送的次级 BPDU，这个时候 AP 会对次级 BPDU 响应

## 二、RSTP 的状态和端口角色
### 1. 增加的角色：AP、BP、EP
1. AP（替代端口）：收到其他交换机更优的 BPDU 而被阻塞的端口
	![](DICM/Pasted%20image%2020251202162030.png)
2. BP（备份端口）：收到自己更优的 BPDU，而被阻塞的端口
	![](DICM/Pasted%20image%2020251202162158.png)
3. EP（边缘端口）

### 2. 端口状态
1. RSTP 状态简化
	1. Discarding：<font color="#ff0000">只能</font>收 BPDU 报文，不能发 BPDU、不能学习 MAC 地址、不能转发数据
	2. Learning：可以收发 BPDU 报文，可以学习 MAC 地址，<font color="#ff0000">不能</font>收发数据
	3. Forwarding：可以收发 BPDU 报文，可以学习 MAC 地址，<font color="#ff0000">可以</font>收发数据
2. STP 的状态
	1. DIsable：
	2. Blocking：
	3. Listening：
	4. Learning：
	5. Forwarding：


## 三、RSTP 和 STP 报文的对比
### 1. STP 报文
1. 配置 BPDU：就是用于选举根桥和计算端口角色
	- FLAG：TCA _ _ _ _ _ _ TC   (中间六位空余)
2. TCN 的 BPDU：当拓扑发生变化的时候（有接口进入到forwarding 状态）

### 2. RSTP 的报文
1. RSTP 的 BPDU
	- FLAG：![](DICM/Pasted%20image%2020251202105356.png)
	1. TCA：不考虑与 STP 兼容就没用
	2. Agreement：同意，和 P 位共同工作，<font color="#ff0000">进行 PA 机制</font>，提高收敛速度
	3. Forwarding 和  Learning 共同表示端口状态：（必须严格按照 F 和 L 的顺序看以下内容：00：`Discarding`、01：`Learning`、11：`Forwarding`。<font color="#ff0000">10没有用，不存在这种状态</font>）
		- Learning：置1-->端口可以学习 MAC 地址
		- Forwarding：置1-->端口进入转发状态
	4. Port Role：端口角色（11：`DP` 、10：`RP`、`01`：AP/BP。<font color="#ff0000">00保留没有用</font>）
	5. Prososal：协商，和 A 共同工作，<font color="#ff0000">进行 PA 机制</font>提高收敛速度
	6. TC：拓扑发生变化的时候，默认发送两次 TC 置位的 BPDU 报文。主要是为了清除 MAC 地址表，避免 MAC 地址表学习错误

## 四、PA 机制工作原理和条件
### 1. 条件（缺一不可）
1. 端口处于<font color="#ff0000">全双工</font>
2. DP 处于 `Dscording` 或者 `Learning` 状态才能发送 P 置位 `BPDU`
3. DP 对端一定是 RP，才能回 A 置位 BPDU
### 2. 工作原理
![](DICM/Pasted%20image%2020251202113636.png)

<font color="#ff0000">接下来的抓包均只看 BPDU FLAG 字段</font>
1. 当 `SW1` 和 `SW2` 的链路故障，`SW2` 会发送以自己为根桥的 BPDU 报文
	抓包截图：<font color="#ff0000">FLAG：0,1,0,0,11,1,0</font>
	![](DICM/Pasted%20image%2020251202113836.png)

2. 但 `SW3` 会马上回应 `SW2`
	抓包截图：<font color="#ff0000">FLAG：0,1,0,0,11,1,0</font>
	![](DICM/Pasted%20image%2020251202111823.png)

3. `SW2` 收到 `SW3` 发送的 P 和 A 同时置位的 BPDU 报文后，会回复 A 置位的 BPDU 报文，同时端口立马进入到 `Forwarding` 状态
	抓包截图：<font color="#ff0000">FLAG：0,1,1,1,10,0,0</font>
	![](DICM/Pasted%20image%2020251202113001.png)

4. `SW3` 的2端口本来是阻塞（AP）状态，此时转换到了 DP 状态，因此会发送一个 TC 置1的 BPDU，清除 MAC 表
	<font color="#ff0000">只有 DP 才会发送 TC 置1的报文</font>
	抓包截图：<font color="#ff0000">FLAG：0,1,1,1,11,0,1</font>
	![](DICM/Pasted%20image%2020251202113607.png)



### 3. 为什么必须是在全双工、点到点能有 PA 机制？
<font color="#ff0000">如果不是点到点全双工，可能会导致临时环路</font>
![](DICM/Pasted%20image%2020251202160000.png)
如图所示，一开始 hub 与 `SW1` 没连接，后面连接之后，发生以下过程
1. `SW1`（作为根桥）从其连接Hub的端口（GE0/0/3）向外发送一个**P置位（Proposal）的BPDU**
	- **关键点来了**：由于Hub是共享介质，这个BPDU不是点对点地发给某个特定交换机，而是**以广播的形式，被Hub连接的所有设备（在这个场景下是SW3和SW4）同时收到**。
2. 此时如果 `SW4` 抢先发出了 A 置位的报文，那么 `SW1` 的3端口将立刻进入转发状态，<font color="#ff0000">但我们的本意是让 SW3回应 SW1</font>
	- 按照本意：
		1. <font color="#ff0000">SW3回复 A 置位的报文之前会阻塞自己其他的所有端口，但被 SW4抢占了，所以环路</font>
		2. SW3阻塞其他端口（端口2）让它进入 Discarding 状态并发出 P 置位的报文给 SW2，再从 SW2再发，直到找到该被阻塞的端口（三台设备太少，无法描述好这个过程）
3. 此时左侧 `SW1、SW2、S3` 所在位置将出现临时环路环路

## 五、RSTP 的老化时间
1. STP 默认的老化时间是20秒
2. RSTP的BPDU的 `老化时间=3*he11o的时间*时间因子`，时间因子默认是3，所以RSTP的BPDU的老化时间默认是18秒，如果将时间因子设置为1，那么BPDU的老化时间就为6
	如何修改时间因子：`stp timer-factor <1-10>`：默认是 3


## RSTP 的保护机制
### 1. BPDU 保护
用在边缘端口，当边缘端口收到 BPDU 报文后，边缘端口会被 `shutdown`
#### 案例1：BPDU 保护的实操
##### 1. 使用关闭 STP 的交换机来模拟攻击设备
![](DICM/Pasted%20image%2020251203152843.png)
##### 2.在连接外部终端的交换机上配置边缘端口，并设置 bpdu 保护，设置超时自动恢复
```
[Huawei-GigabitEthernet0/0/1]stp edged-port enable
[Huawei]stp bpdu-protection
[Huawei]error-down auto-recovery cause bpdu-protection interval 30
```
此时，如果将 LSW2的 stp 打开，LSW1会触发 bpdu 保护，并将端口 `shutdown`，因为我们设置了30秒恢复，所以过了30秒端口可以自动启用，但再次收到 bpdu 报文会再次将端口 `shutdown`
![](DICM/Pasted%20image%2020251203152504.png)

### 2. 环路保护
交换网络中因为可能用光纤互联，出现单通。导致一个方向出现环路
**单向链路故障**：在某些物理介质（如光纤收发器）或设备故障的情况下，链路可能出现单向通信问题。即A能收到B的信号，但B收不到A的信号。
环路保护是一种在特定端口上启用的安全机制，用于检测该端口的对端是否依然正常工作。如果检测到对端不再接收本端的信号（即出现单向链路），环路保护会将本端端口阻塞，从而主动防止环路的产生。
#### 案例1：环路保护的实操

![](DICM/Pasted%20image%2020251203162332.png)

```
[Huawei-GigabitEthernet0/0/2]stp loop-protection  //开启环路保护
```

### 3. TC 保护
1. **TC保护**是RSTP提供的一种安全机制，**限制交换机在单位时间内能够处理的TC BPDU的数量，并对超出限制的TC BPDU来源进行惩罚**
	TC保护通常作用于**非边缘端口**

2. **设定阈值**：管理员可以配置一个时间窗口（例如每2秒）和一个在该时间窗口内允许处理的最大TC BPDU数量（例如1个）。
	1. **正常情况**：当交换机在设定的时间窗口内收到的合法TC BPDU数量没有超过阈值时，它会正常处理这些TC BPDU，即刷新MAC地址表的老化时间。
	2. **异常情况**：
		- 如果在单位时间内，交换设备在收到TCBPDU报文数量<font color="#ff0000">大于配置的间值</font>，那么设备<font color="#ff0000">只会处理阈值指定的次数</font>
		- 对于其他超出间值的TCBPDU报文，<font color="#ff0000">定时器到期后设备只对其统一处理一次</font>。这样可以避免频繁的删除MAC地址表项，从而达到保护设备的目的。
#### 配置命令
全局下配置
```
[Huawei]stp tc-protection threshold <1-255>  TC-BPDU 保护的阈值，默认为 1。
```


### 4. 根保护
- 保护根桥的地位不会改变，根保护配置在 <font color="#ff0000">DP</font> 端口，避免接口收到比根桥还好的 bpdu，导致逻辑拓扑发生变化。
- 如果开启根保护之后，收到了比根桥还好的 bpdu，接口仍然是 DP，但是状态处于 `discarding` 状态
#### 案例：根保护的实操
##### 1. 默认情况下，先将 LSW6优先级设置较低不破坏本网络的拓扑，模拟公司骨干内网与其他网络连接
![](DICM/Pasted%20image%2020251203150356.png)
##### 2. 不设置根保护，将 LSW6的优先级设置为0，来破坏左侧网络拓扑
![](DICM/Pasted%20image%2020251203150609.png)
<font color="#ff0000">此时，LSW6被选举为根桥，公司内部的网络拓扑被破坏了</font>

##### 3. 在恢复默认情况后，开启根保护，再将 LSW6的优先级设置为0
![](DICM/Pasted%20image%2020251203151433.png)


## 六、案例分析
### 案例1：RSTP 中断开一条 DP与 RP 的线路
- 在没关闭 HUB 的时候，LSW1的1端口是 `DP`, LSW2的1端口是 `RP`、2端口是 `DP`，`LSW3` 的1端口是 blocking
![](DICM/Pasted%20image%2020251201203121.png)
当 HUB 关闭后
1. `LSW2` 会造反，自身端口2向 `LSW3` 的端口1发出 BPDU，`4096  开销  4096+MAC`，此时 `LSW3` 收到后与 `LSW1` 发的 BPDU 对比，发现还是原本的优先级高（LSW2 的新 BPDU：根桥 ID=4096（LSW2）、根路径开销 = 0，但**根桥 ID（4096）比原根桥 ID（0）差**）
2. `LSW3`发现：虽然 `LSW2` 的 BPDU 不是最优，但**LSW2 的 GE 0/0/2 端口是其 “根端口候选”**，因此 LSW3 向 LSW2 回复**Agreement 类型的 BPDU（该 BPDU 携带的是根桥 ID的信息）**
3. `LSW2` 收到 `Agreement` 后，<font color="#ff0000">立即</font>将 **GE 0/0/2 从 Discarding 切换为 Forwarding**，变成 `RP`
4. `LSW3` 收到自己的根端口（GE 0/0/2）的 BPDU 仍有效，同时将 GE 0/0/1 从 `Discarding` 切换为 `Forwarding`（成为 `DP` 端口）。

### 案例2（探索 RSTP 的老化时间）：RSTP 中断开与根桥相连的 HUB，但 RP 端口的 HUB 还在
![](DICM/Pasted%20image%2020251202163700.png)
实验开始后会断开 `HUB2` 与 `SW1` 的连线
- 需要等待18秒的 RSTP 老化，SW2才会造反，造反过程与[案例1：RSTP 中断开一条 DP与 RP 的线路](#案例1：RSTP%20中断开一条%20DP与%20RP%20的线路)一样

### 案例3：深入理解 STP/RSTP 的选举规则
从右往左：首先 SW1和 SW4不相连，后面相连之后的端口变化过程
![](DICM/Pasted%20image%2020251202170129.png)
1. 连接之后，SW1的4端口变成 DP，并变成 discarding 状态，向 SW4的4端口发送 PA 报文，然后 <font color="#ff0000">SW4阻塞自己的其他所有端口</font>，再回复 A 置位的报文
2. 以此类推，SW4的3端口被置为 Discarding 状态并发送 PA 报文给 SW3，SW3收到后阻塞自己的所有端口并回复 A 置位的报文
3. 此时，SW3知道 SW4的 BID 远小于 SW2，<font color="#ff0000">所以整机优先从 SW2发送数据</font>，但 SW3的3端口开销大于2端口，所以3端口被阻塞
#### 这里为什么阻塞了 SW3的3端口而不是 SW4的3端口呢？
<font color="#ff0000">DP选举是优先看开销</font>，<font color="#7030a0">看完开销相同再比BID</font>，因此 `SW3` 的3端口到根桥从 `SW3-->SW2-->SW1` （cost = 20+20= 40）要大于 `SW4` 的 `SW4-->SW1` （cost= 20）的<font color="#ff0000">开销</font>，所以 `SW4` 的3端口是DP。图片上理解其实也没特别大的问题
