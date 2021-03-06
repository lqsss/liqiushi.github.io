﻿---
title: 网络 
tag: summary
categories: network
---

网络知识总结
<!--more-->

## TCP

### 三次握手
![](http://upload-images.jianshu.io/upload_images/5753761-a86c0d6ac857c3bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- A:发送SYN请求 告诉B "我要连接了"
- B:SYN、ACK ack = x+1，告诉A "哦 我知道了，你能听到我的话"
- A:ACK 告诉B "可以听到！我要发数据了！！"
- A在发送ACK后，进入ESTABLISHED，B在接受到ACK后进入ESTABLISHED

### 为什么要这样设计
>保持通信双方的信息对称，使通信双方处于同步状态

保证双方都能发数据，也能收数据。
1. 如果当初的TCP设计的是两次握手，
- A发送SYN，B接收到 返回对A请求的确认，自身进入ESTABLISHED状态，A收到，进入ESTABLISHED （ok）
- A发送SYN，B接收到返回对A请求的确认，自身进入ESTABLISHED状态，[SYN ACK]由于网络原因，而停滞。A以为SYN发送失败，则重传一个SYN，此时B再返回[SYN ACK]，A收到，双方都进入ESTABLISHED ，进行传输数据。假设此连接在通信结束后，释放连接资源，而此时第一次在网络停滞但未被丢弃的SYN包抵达了，B又进入ESTABLISHED，而此时A已经CLOSE，B一直没有释放连接资源，苦苦等待，因此浪费了资源。

2. 假设四次握手的过程是这样：
- A->B SYN
- B->A ACK
- B->A SYN
- A->B ACK

中间二三步可以合并一起，B既确认A（表示能收），发ACK给B，试试能不能发，四次没有必要，三步就可以确保双方的收发，建立连接。

### 终止
![](http://upload-images.jianshu.io/upload_images/5753761-3a73f69eb7f83808.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 过程
- A主动关闭方，B被动关闭方
- A放送[FIN] 一个中断请求给B，"我要关闭啦！！！"
- B收到FIN，发送ACK，告诉A，"我知道了你要关闭了，你还需要等我下，可能还有要发送的数据"，通知应用程序。
- A收到 进入FIN_WAIT_1
- B进入CLOSE_WAIT状态（被动关闭），已经没有要发送的工作了，发送一个[FIN ACK]，"我没什么要发的，可以关闭啦"
- A收到进入FIN_WAIT_2，此时也得发送一个ACK确认，"收到了你的关闭请求"，进入TIME_WAIT状态，等待一个2MSL时间
- B收到了ACK，进入CLOSED，A在2MSL的时间段内没有收到FIN，确保最后一次ACK发送到B，这下放心地关闭了

### 为什么要这样设计
>用来保障通讯双方可以安全的回收TCP通信的系统资源(全双工关闭)

关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送。

**为什么有TIME_WAIT状态**
A发送对于B发送的FIN数据的一个ACK，A并不知道B是否接到自己的ACK，如果B没有接受到ACK，超时重传会再次接受到B的FIN。所有A需要等待；
而B作为被动关闭方，在收到ACK确认后，表明了双方达成了同步，所以B可以安心释放资源，无需等待。
**为什么要2msl**
>maximum segment lifetime(最大分节生命期）,这是一个IP数据包能在互联网上生存的最长时间，超过这个时间将在网络中消失。

要取这两种情况等待时间的最大值，以应对最坏的情况发生，这个最坏情况是：去向ACK消息最大存活时间（MSL) + 来向FIN消息的最大存活时间(MSL)。


### 可靠
TCP的可靠传输在于：

- 解决了乱序
- 超时重传
- 拥塞控制
- 流量控制

1. 乱序和丢失：Sequence Number是包的序号，解决乱序，ACK解决丢包。
2. 超时重传：对每一个包设置有超时重传定时器，在一定时间内没有收到ACK，则重新发送
3. 拥塞控制：滑动窗口协议解决


#### 滑动窗口协议
- "窗口"对应的是一段可以被发送者发送的字节序列，其连续的范围称之为"窗口"；
- "滑动"则是指这段“允许发送的范围”是可以随着发送的过程而变化的，方式就是按顺序"滑动"。
- 发送缓冲区：应用层需要发送的所有数据都被放进了发送者的发送缓冲区，发送窗口是发送缓存中的一部分

已发送并收到确认的数据：不在发送窗口中，缓冲区中删除
已发送未收到确认的数据：在发送窗口中
未发送：在发送窗口中
不允许发送：在窗口外

每次成功发送数据之后，发送窗口就会在发送缓冲区中按顺序移动，将新的数据包含到窗口中准备发送；接收窗口乱序的包先暂存下来。


#### 拥塞控制
&nbsp;&nbsp;&nbsp;&nbsp;网络中的链路容量和交换结点中的缓存和处理机都有着工作的极限，当网络的需求超过它们的工作极限时，就出现了拥塞。**拥塞控制就是防止过多的数据注入到网络中**，这样可以使网络中的路由器或链路不致过载。常用的方法就是：

- 慢开始、拥塞避免

- 快重传、快恢复

一切的基础还是慢开始，这种方法的思路是这样的：

1. 发送方维持一个叫做“拥塞窗口”的变量，**该变量和接收端口共同决定了发送者的发送窗口；**

2. 当主机开始发送数据时，避免一下子将大量字节注入到网络，造成或者增加拥塞，选择发送一个1字节的试探报文；

3. 当收到第一个字节的数据的确认后，就发送2个字节的报文；

4. 若再次收到2个字节的确认，则发送4个字节，依次**递增2的指数级；**

5. 最后会达到一个提前预设的“慢开始门限”，比如24，即一次发送了24个分组，此时遵循下面的条件判定：

- cwnd < ssthresh， 继续使用慢开始算法；

- cwnd > ssthresh，停止使用慢开始算法，改用拥塞避免算法；

- cwnd = ssthresh，既可以使用慢开始算法，也可以使用拥塞避免算法；

6. 所谓拥塞避免算法就是：每经过一个往返时间RTT就把发送方的拥塞窗口+1，即让拥塞窗口缓慢地增大，按照线性规律增长；

7. 当出现网络拥塞，比如丢包时，将慢开始门限设为原先的一半，然后将cwnd设为1，执行慢开始算法（较低的起点，指数级增长）；

![](http://upload-images.jianshu.io/upload_images/5753761-2f8b817560c58f21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

拥塞：网络中的链路容量和交换结点中的缓存和处理机都有着工作的极限。网络的需求超过它们的工作极限时，就出现了拥塞。
拥塞窗口(发送方才有)：发送方取拥塞窗口与通告窗口中的最小值作为发送上限。
慢启动：维护一个拥塞窗口，开始时1、2、4...指数递增（慢启动算法），发送方开始时发送一个报文段，然后等待ACK。当收到该ACK时，拥塞窗口从1增加为2，即可以发送两个报文段。当收到这两个报文段的ACK时，拥塞窗口就增加为4。这是一种指数增加的关系。
拥塞避免算法：当慢启动一段时间后，拥塞窗口达到了"慢开启门限",拥塞窗口在一个RTT进行线性缓慢+1的增长。
出现拥塞之后：慢开始门限设置为此时拥塞窗口的一半，然后将cwnd设为1，执行慢开始算法（较低的起点，指数级增长）。

---

>如果数据包丢失了，TCP将会使用定时器来要求传输暂停,在暂停的这段时间内，没有新的或复制的数据包被发送

1. 接收方建立这样的机制，如果一个包丢失，则对后续的包继续发送针对该包的重传请求；

2. 一旦发送方接收到三个一样的确认，就知道该包之后出现了错误，立刻重传该包；

3. 此时发送方开始执行“快恢复”算法：

    3.1 慢开始门限减半；

    3.2 cwnd设为慢开始门限减半后的数值；

    3.3 执行拥塞避免算法（高起点，线性增长）；

![](http://upload-images.jianshu.io/upload_images/5753761-dc5cb2cd8d550945.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

快重传：因为拥塞情况而出现丢包的情况，为了不用等待超时重传时间，当收到3个dup ack，发送方立刻重传该包
快恢复：此时慢开始门限减半，拥塞窗口从减半后的数值开始，执行拥塞避免算法。

---

## IP
>面向无连接和不可靠的传输功能

### 功能
1. IP协议提供了IP地址，并将源目IP地址夹带在通信数据包里面，为路由器指明通信方向；
2. 数据包分片和组装功能
3. 防止数据包环路

### MSS
MSS就是TCP数据包每次能够传输的最大数据分段

### MTU
最大传输单元，最大传输单元这个参数通常与通信接口有关（网络接口卡、串口等）。

TCP/IP协议中除了IP分片还有TCP分段，并且协议栈是优先做TCP分段，因为IP分片的处理效率是很低的。
1、TCP：TCP协议下发包，**协议栈优先做TCP分段，保证ip层需要发送的数据不会因为超过MTU而做IP分片。**
2、UDP：UDP协议传输层没有分段的功能，只能依靠IP分片来发送较大的数据段。

### 路由器
ip协议选择适合的路线传送，ip切分的不同数据包经过不同的路线（路由器）到达最后的终点，其中路由器相当于一个快递公司在每个城市的中转站点。
路由器根据数据包的IP地址查找路由表（地图），然后以接力棒的方式逐跳转发直到目标服务器；

## 链路层
链路层-数据帧附加上了源mac地址，目的mac地址等信息


### ip地址和mac
- ip地址是ip协议提供的一种统一的地址格式，它为互联网上的每一个网络和每一台主机分配一个逻辑地址，可以随时变动。
- mac地址：物理地址，网卡芯片里的地址
>二层基于MAC地址转发数据帧，三层基于IP地址转发报文


## 当你在浏览器中输入Google.com并且按下回车之后发生了什么？

### DNS域名解析
1. 检查浏览器中是否缓存有此域名对应的ip地址
2. 检查本地host文件是否存有该域名和ip的映射关系
3. 如果上述都没有，就首先会找TCP/ip参数中设置的首选DNS服务器(本地服务器)查询，这中间会有ARP过程


#### ARP
我们知道本地DNS服务器的IP地址，但是需要知道它的mac地址
1. 通过子网掩码判断DNS服务器与本机是否处于同一子网
2. 查看本地ARP缓存表是否存有ip-mac的映射关系,如果没有进行下面
3. ARP
    3.1 处于同一子网：对目的IP进行APR请求

- 本机连接了交换机（mac-port表），通过交换机广播ARP请求{源ip、源mac、目的ip}
- 局域网中的机器收到请求发现ip不是自己的，则丢弃；如果发现目的ip是自己，则将mac封到帧单播ARP应答，并且ARP缓存表缓存对方的ip-mac
- 源主机收到回复，缓存到ARP缓存表里。

    3.2 不处于同一子网：需要默认网关中转，然后对目的IP进行APR请求

- 如果ARP缓存表里没有默认网关的mac，需要首先对网关进行ARP查询（路由器与交换机相连）
- 将帧{目的ip，路由器mac}转发给路由器，上面有路由表（端口和ip网段的对应）决定了下一跳，帧发生改变{目的ip，下一跳的mac地址}
- 当路由器不停地转发（解封、修改mac、包装），找到了目的ip的网段，如果路由器上有ARP缓存表，则直接返回，否则广播ARP请求。
**整个过程包含mac学习（交换机上的mac-port）**

4. 找到了本地DNS的mac地址后，对本地DNS进行DNS请求
5. 如果本地DNS没有映射存储的话，需要继续递归查询，向跟高级地进行查询

>查询分为两种：递归式查询：dns客户端最终要么得到结果，要么返回错误；迭代查询：dns客户端每次得到的不一定是最终结果。

例子：查询www.cs.cmu.edu
迭代查询：
- 向root服务器（edu）查询www.cmu.edu，返回cmu.edu
- 向cmu.edu服务器查询，得到cs.cmu.edu
- 最终向cs.cmu.edu服务器查询

>查询协议：向DNS 服务器发送 UDP 请求包，如果响应包太大，会使用 TCP 协议

6. 与目的ip进行TCP三次握手连接
    - 包装端口-> 包装ip->包装mac,如果mac没有需要ARP
7. http协议传输数据
8. 接下来就是浏览器的事情了，渲染dom树（html、css、js）