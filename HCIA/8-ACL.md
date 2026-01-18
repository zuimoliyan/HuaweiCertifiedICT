## 一、ACL产生的背景
1. 希望控制（接受、拒绝）报文的收发
2. NAT网络地址转化希望某些网段能访问外网
3. 希望路由的过滤一控制业务的互访
4. QOS服务质量
5. route-policy-路由的引入
6. PBR policy-base-route-重定向

## 二、ACL的组成
1. 由一系列的rule组成，规则和规则之间是通过序号来进行区分匹配顺序
2. **按照rule的编号的从小到大进行匹配**，一旦匹配某条rule，就**跳出该ACL**
3. 如果没有匹配就往下面的rule的进行匹配，如果所有的都没有匹配，那么就执行默认的rule


## 三、ACL分类
### 1.基本ACL（2000-2999）
主要针对IP报文的源IP地址进行匹配，基本ACL的编号范围是2000-2999
用于：
1. 用于数据包过滤，但是**只能匹配源网段**，很少用户数据包的匹配
2. 可以用于匹配路由
3. 可以用于NAT策略
### 2.高级ACL（3000-3999）
可以根据IP报文中的源IP地址、目的IP地址、协议类型，TCP或UDP的源目端口号等元素进行匹配
	可以理解为：基本ACL是高级ACL的一个子集，高级ACL可以比基本ACL定义出更精确、更复杂、更灵活的规则
用于：
1. 匹配数据包
2. 用于Qos
3. 用于PBR
注意：==华为的设备不能使用高级的ACL来匹配路由==
4. 高级的ACL可以用于NAT设备


### 华为设备特性
出方向限制: 无法过滤设备**自身发出**的流量（如OSPF Hello报文）



## 四、具体实验场景
### 1.包过滤（ACL3000~3999）：默认会有一个隐含的permit


#### 案例1：AR2在入口禁止10.0.12.0、10.0.13.0、10.0.14.0、10.0.15.0网段数据包进入
![](DICM/Pasted%20image%2020251121171003.png)
1. `rule 5 deny icmp source 10.0.12.1 0.0.3.0 destination 10.0.100.2 0`：配置拒绝所有从 `10.0.12.0-10.0.15.255` 网段发往 `10.0.100.2` 的`ICMP`数据包
2. 设置在0端口启用

#### 案例2：AR3在案例一基础上允许10.0.12.0网段数据包进入
```
acl number 3000  
 rule 5 deny icmp source 10.0.12.1 0.0.3.0 destination 10.0.100.3 0 
 rule 10 permit icmp source 10.0.12.0 0.0.0.255 destination 10.0.100.3 0
```
注意：以上为==错误配置==，由于ACL找到匹配的规则就直接跳出选举规则，所以匹配到rule5之后就不会去看rule10了
下图为正确配置
![](DICM/Pasted%20image%2020251121171631.png)
1. 使用`10.0.12.1`ping`10.0.100.3`此时检测rule5匹配规则，允许通过并跳出选取
2. 如果使用`10.0.13.1`，此时rule5不会匹配，但rule10匹配，拒绝并跳出选取

#### 案例3：ACL match-order auto的使用
```
acl number 3000 match-order auto
 rule deny icmp source 10.0.12.1 0.0.3.0 destination 10.0.100.3 0 
 rule permit icmp source 10.0.12.0 0.0.0.255 destination 10.0.100.3 0
```
![](DICM/Pasted%20image%2020251121172359.png)
如上图，即使我们现配置拒绝，但是路由器还是会主动帮我们修改顺序
与案例2的不同是直接去掉rule后面的数据，路由器会自动分配
![](DICM/Pasted%20image%2020251121172300.png)


### 2.路由过滤（ACL2000~2999）：默认会有一个deny所有

#### 案例1：AR2在入口禁止10.0.12.0网段的路由
```
acl number 2000  match-order auto
 rule 5 deny source 10.0.12.0 0.0.0.0 //这里有个重点在---相关命令---中说
 rule 10 permit //默认会有一个deny过滤所有路由，所以我们需要手动全部允许
 
 ospf 1 
 filter-policy 2000 import  //结尾的import表示引入，export代表传出
```
![](DICM/Pasted%20image%2020251121174042.png)

### 解答
#### 对于 包过滤-案例1 中禁止多个网段：
`10.0.12.1 0.0.3.0`：其中`0.0.3.0`是标识符，转成二进制是：`0.0.0000,0011.0`这其中0表示不可变，1表示可变
	即`10.0.0000,1100.0`中，第三段可变成：`1100`、`1101`、`1110`、`1111`四个，即：12、13、14、15网段
#### 对于 路由过滤-案例1 中禁止单个网段
在OSPF中配置`rule 5 deny source 10.0.12.0 0.0.0.0`会使拒绝接收其他路由器`10.0.12.0`网段发过来的OSPF协议报文

## 五、相关命令

1. `[Huawei]acl number <数字>`：2000~2999是路由过滤，3000~3999是包过滤
2. `[Huawei]acl number <数字> match-order auto`：设置自动添加rule后的数字
3. `[Huawei-acl-basic-2000]rule <数字> deny icmp source 10.0.12.1 0.0.3.0 destination 10.0.100.3 0.0.0.0 `
	- 对于数字：如果 match-order 是 auto就加，反之不加
	- deny：拒绝接收，也可以是permit
	- icmp ：协议类型，除了这个还有别的类型
4. `[Huawei]display acl all`：显示ACL的过滤日志
5. `[Huawei-ospf-1]filter-policy 2000 import`：OSPF下引入路由过滤
6. `[Huawei-GigabitEthernet0/0/0] traffic-filter inbound acl 3000`：接口下引入包过滤
7. `ip address 10.0.15.1 24 sub`：从地址，加上sub 就是在这个接口下能起多个地址




