## 一、产生
IPv4地址匮乏，2的32次方个地址，42.9亿个地址，但是还要去掉组播地址、去掉网络地址和子网广播。

## 二、NAT的分类
### 1.源NAT
#### 1.no-pat：仅转换IP地址，不转换端口号
1对1转换，将一个私网地址转换成一个公网地址。no-pat技术是无法节约IP地址的

#### 2.NAPT：同时转换IP地址和端口号
- 实现n:1地址复用（n个私网IP共享1个公网IP）
- napt使用的是地址池（地址池中也许有多个，也许只有一个公网IP）

#### 3.EasyIP：是NAPT的特殊形式，仅使用外网接口的物理IP地址进行转换


### 2.目的NAT
#### 1.产生背景
主要是服务器的IP是私有的，但是需要给外网用户端来进行访问，那么就需要在**出口路由器**配置目的NAT



## 三、具体实现
### 1.源NAT
#### 1.no-pat实现
前提：
**不知道为什么防火墙的0/0/0端口无法被其他设备ping通，尽量不用**
```
1.
#配置安全区域分别将g1/0/1加入trust区域、g1/0/2加入untrust区域  
	[FW1]firewall zone trust   
	[FW1-zone-trust]add int g1/0/0
	[FW1-zone-untrust]add int g1/0/1
2.
#到防火墙与设备连接的端口下配置允许ping包（开放防火墙的 ICMP 管理权限）
	[FW1] interface GigabitEthernet 1/0/0
	[FW1-GigabitEthernet1/0/0] service-manage ping permit #仅允许该接口被ping
	[FW1] interface GigabitEthernet 1/0/1
	[FW1-GigabitEthernet1/0/1] service-manage ping permit # 仅允许该接口被ping

```
![](DICM/Pasted%20image%2020251122152348.png)
配置好前提和所有的端口下的IP地址后：
##### 1.配置访问公网的默认路由
```
[FW1]ip route-static 0.0.0.0 0 200.1.1.2
```
##### 2.创建NAT地址池
```
[FW1]nat address-group 1
[FW1-address-group-1]mode no-pat global //设置模式为no-pat
[FW1-address-group-1]section 200.1.1.1 200.1.1.1 //配置端口IP作为地址池的IP
[FW1-address-group-1]route enable //开启NAT路由，不开启可能会导致环路
```
##### 3.配置NAT策略
```
[FW1]nat-policy
[FW1-policy-nat]rule name nat
[FW1-policy-nat-rule-nat]source-address 192.168.1.0 mask 255.255.255.0
[FW1-policy-nat-rule-nat]action source-nat address-group 1
```
##### 4.实验结果
此时如果PC1去pingPC3可以ping通
![](DICM/Pasted%20image%2020251122153210.png)
但是PC2无法ping通：
- 地址池中的地址已经全部分配出去，则剩余内网主机访问外网时不会进行NAT转换，直到地址池中有空闲地址时才会进行NAT转换
- no-pat 地址池默认老化时间是1小时

#### 2.NAPT
![](DICM/Pasted%20image%2020251122205626.png)

##### 1.在`AR1`上配置企业内网的出口NAT
```
[Huawei]acl 2000
[Huawei-acl-basic-2000]rule 5 permit source 192.168.1.0 0.0.0.255 //创建路由过滤规则

[Huawei]nat address-group 1 10.12.0.10 10.12.0.10 //创建地址池

[Huawei]int g 0/0/0
[Huawei-GigabitEthernet0/0/0]nat outbound 2000 address-group 1 //为出口网关应用NAPT地址池

[Huawei]ip route-static 0.0.0.0 0 10.12.0.2 //设置静态路由
```
##### 2.在`AR2`上配置好充当`AR1`和`AR3`路由的通信桥梁
```
[Huawei]ip route-static 172.168.1.0 255.255.255.0 10.23.0.3
[Huawei]ip route-static 192.168.1.0 255.255.255.0 10.12.0.1
```
##### 3.在`AR3`上配置服务器的出口NAT
```
[Huawei]acl 2000
[Huawei-acl-basic-2000]rule 5 permit source 172.168.1.0 0.0.0.255 //创建路由过滤规则

[Huawei]nat address-group 1 10.12.0.10 10.12.0.10 //创建地址池

[Huawei]int g 0/0/0
[Huawei-GigabitEthernet0/0/0]nat outbound 2000 address-group 1 //为出口网关应用NAPT地址池

[Huawei]ip route-static 0.0.0.0 0 10.23.0.2 //设置静态路由
```

#### 3.Easy-IP
##### 是NAPT的特殊形式，仅使用外网接口的物理IP地址进行转换

![](DICM/Pasted%20image%2020251122211050.png)

##### 1.在`AR1`上配置企业内网的出口NAT
```
[Huawei]acl 2000
[Huawei-acl-basic-2000]rule 5 permit source 192.168.1.0 0.0.0.255 //创建路由过滤规则

[Huawei]int g 0/0/0
[Huawei-GigabitEthernet0/0/0]nat outbound 2000 //为出口应用Easy IP模式

[Huawei]ip route-static 0.0.0.0 0 10.12.0.2 //设置静态路由
```
##### 2.在`AR2`上配置好充当`AR1`和`AR3`路由的通信桥梁
```
[Huawei]ip route-static 172.168.1.0 255.255.255.0 10.23.0.3
[Huawei]ip route-static 192.168.1.0 255.255.255.0 10.12.0.1
```
##### 3.在`AR3`上配置服务器的出口NAT
```
[Huawei]acl 2000
[Huawei-acl-basic-2000]rule 5 permit source 172.168.1.0 0.0.0.255 //创建路由过滤规则

[Huawei]int g 0/0/0
[Huawei-GigabitEthernet0/0/0]nat outbound 2000 //为出口应用Easy IP模式

[Huawei]ip route-static 0.0.0.0 0 10.23.0.2 //设置静态路由
```

### 2.目的NAT
前提：
**不知道为什么防火墙的0/0/0端口无法被其他设备ping通，尽量不用**
```
1.
#配置安全区域分别将g1/0/1加入trust区域、g1/0/2加入untrust区域  
	[FW1]firewall zone trust   
	[FW1-zone-trust]add int g1/0/0
	[FW1-zone-untrust]add int g1/0/1
2.
#到防火墙与设备连接的端口下配置允许ping包（开放防火墙的 ICMP 管理权限）
	[FW1] interface GigabitEthernet 1/0/0
	[FW1-GigabitEthernet1/0/0] service-manage ping permit #仅允许该接口被ping
	[FW1] interface GigabitEthernet 1/0/1
	[FW1-GigabitEthernet1/0/1] service-manage ping permit # 仅允许该接口被ping
```
![](DICM/Pasted%20image%2020251122165436.png)
##### 1.在防火墙配置NAT策略
```
[FW1-policy-nat-rule-nat]nat-policy # 进入全局NAT策略视图
[FW1-policy-nat]rule name nat # 创建一条名为“nat”的NAT规则
[FW1-policy-nat-rule-nat]source-address 192.168.1.0 0.0.0.255 # 指定NAT规则的匹配条件：仅对“192.168.1.0/24”网段的内网设备生效
[FW1-policy-nat-rule-nat]action source-nat easy-ip # 配置规则执行动作：对匹配的内网流量执行“源NAT转换”，采用Easy-IP模式

[FW1]nat server protocol tcp global 200.1.1.1 80 inside 192.168.1.3 80
	# 等价于原www写法，各字段含义：
	# protocol tcp：指定转发协议为TCP（HTTP服务的底层协议）
	# global 200.1.1.1 80：公网侧暴露的地址+端口——公网用户访问 200.1.1.1:80（HTTP默认端口）
	# inside 192.168.1.3 80：内网侧目标地址+端口——流量转发至内网192.168.1.3的80端口（内网HTTP服务器）
```
##### 2.在路由器配置Easy-IP
```
配置允许进行NAT转换的内网地址段
[AR1]acl number 2000  
[AR1-acl-basic-2000]rule 5 permit source 172.168.1.0 0.0.0.255 

#在出接口GE0/0/0上做Easy IP 方式的NAT
[AR1]int g 0/0/0
[AR1-GigabitEthernet0/0/0]nat outbound 2000

#配置默认路由，保证出接口到对端路由可达
[AR1]ip route-static 0.0.0.0 0.0.0.0 201.1.1.2
```
##### 3.中间的路由器只充当通信作用，配置好两端的IP地址就不动
##### 4.实验结果
![](DICM/Pasted%20image%2020251122170840.png)
成功




## 四、相关命令
1. `nat address-group 1 公网起始IP 公网结束IP`：配置NAT地址池
2. `nat outbound 2000 address-group 1`：绑定ACL和地址池
3. `nat server protocol tcp/udp global 公网IP 端口 inside 内网IP 端口`：Web服务器映射，常用在目的NAT

