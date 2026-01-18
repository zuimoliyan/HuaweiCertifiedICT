# 一、PPP 
## 1、定义
PPP 定义了一套协议，包含了 LCP、NCP（IPCP）、authentication（PAP（密码认证协议）、CHAP（挑战握手认证协议））

## 2、PPP 的三个协议
### 1. LCP---链路控制协议
1. 用来建立、监控、维护链路
2. 用来协商 <font color="#ff0000">MRU</font>(**最大接收单元**)、<font color="#ff0000">认证方式</font>、<font color="#ff0000">魔术字</font> (**控制带宽**)、绑定

### 2. 认证协议
#### 1. PAP--密码认证协议
通过输入的用户名和密码来进行认证
	<font color="#ff0000">如果不配置就不认证</font>

例如 PPPOE 就是用的 PAP 认证
### 3. NCP
用来建立和配置不同的网络层协议，常见的有 IPCP、IPv6CP，检查 IP 地址是否有冲突，或者分配 IP 地址 

## 3. PPP 的工作原理
![](DICM/Pasted%20image%2020251215204219.png)

## 4. PPP 的交互过程

### 1. 首先进行 LCP
![](DICM/Pasted%20image%2020260118163918.png)

### 2. 认证
#### PAP 认证的操作
![](DICM/Pasted%20image%2020251214204035.png)
#### CHAP 认证的操作
![](DICM/Pasted%20image%2020260118163929.png)
- 在第二步：被认证点将**认证点**发送的<font color="#ff0000">挑战字+ID 与自己本地的密码做 HASH 认证</font>，并连同**用户名**和认证点发来的 **ID** 一起发送给认证点
	认证点将被认证点发来的 MD5与<font color="#ff0000">自己存储的当前被认证点的密码同样与挑战字和 ID 一起做 HASH 认证</font>，如果相同就回复成功


### 3.NCP 阶段
#### 1. 有地址的情况
![](DICM/Pasted%20image%2020260118163934.png)

#### 2. 没有地址的情况
![](DICM/Pasted%20image%2020260118163944.png)


# 二、PPPOE
PPPoE将PPP协议扩展到**以太网**环境，常用于家庭宽带接入（如ADSL），通过虚拟链路实现用户认证与管理
## 1、PPPoE的四步发现阶段
1. 客户端广播请求（PADI）
	客户端发送广播报文，寻找可用的PPPoE服务器。
2. ​服务端响应（PADO）​​
	服务器回应可提供服务，携带唯一Session ID。
3. ​​客户端确认（PADR） 
	客户端选择最优服务器并请求建立会话。
4. ​会话建立（PADS）
	服务器确认会话，虚拟链路创建完成。
## 2、PPPOE 终结
客户端或服务器发送​**​PADT报文​**​终止会话，释放Session ID

# 三、实验配置
## 案例1：无认证基础案例
![](DICM/Pasted%20image%2020260118163952.png)

LCP 阶段：
1. 首先两端互相发 `Configuration Request`，主要包含 `MRU`、魔术字
	- AR1
		![](DICM/Pasted%20image%2020260118164000.png)
	- AR2
		![](DICM/Pasted%20image%2020251214212000.png)

2. 两端互相检查对端 MRU
	- AR1：知道对端 `MRU = 1500`，于是自己后续发送<font color="#ff0000">必须≤1500</font> 
		![](DICM/Pasted%20image%2020260118164004.png)
	- AR2 知道对端 MRU=1300，于是自己后续发送<font color="#ff0000">必须≤1300</font>
		![](DICM/Pasted%20image%2020260118164008.png)
	双方回 Configure-Ack（Code=2）把对方选项原样打回 → LCP 进入 Opened 状态

3. <font color="#ff0000">由于没有认证所以直接进行 NCP</font>

NCP 阶段：两端互发 `IPCP Configure-Request`
1. AR1：
	![](DICM/Pasted%20image%2020260118164013.png)
2. AR2：
	![](DICM/Pasted%20image%2020251214212826.png)

<font color="#ff0000">3. 检查地址无冲突 → 回 Configure-Ack</font>


## 案例2.1：PAP认证案例
AR1：
```
[Huawei-Serial4/0/0]ppp pap local-user admin password simple Huawei@123
```
AR2：
```
[Huawei-Serial4/0/0]ppp authentication-mode pap
[Huawei-aaa]local-user admin password cipher Huawei@123
[Huawei-aaa]local-user admin service-type ppp
```

<font color="#ff0000">其他与上述案例1一致，仅在第三步把认证过程加上</font>
![](DICM/Pasted%20image%2020251214214311.png)
1. `AR1` ---> `AR2`
	![](DICM/Pasted%20image%2020251214214521.png)
2. `AR2-->AR1`
	![](DICM/Pasted%20image%2020260118164021.png)

## 案例2.2：CHAP 认证案例

AR1：
```
interface Serial4/0/0
 link-protocol ppp
 ppp chap user admin
 ppp chap password cipher Huawei@123
 ip address 10.1.12.1 255.255.255.0
```
AR2：
```
[Huawei-Serial4/0/0]ppp authentication-mode pap
[Huawei-aaa]local-user admin password cipher Huawei@123
[Huawei-aaa]local-user admin service-type ppp
```

1. 验证端向被认证端发挑战报文，包含 **ID** 和**挑战字**
	![](DICM/Pasted%20image%2020260118164026.png)

2. 被认证端做完 **HASH** 后将结果与自己的**用户名**和发过来的 **ID** 一起发回去
	![](DICM/Pasted%20image%2020260118164029.png)

3. 验证成功后，认证端回复确认 ACK
	![](DICM/Pasted%20image%2020251215204109.png)