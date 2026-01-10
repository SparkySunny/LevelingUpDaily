# netfilter
## netfilter简介
Netfilter框架由一系列钩子（hooks）组成，这些钩子位于Linux网络栈的关键路径上。当数据包经过网络栈时，它会触发这些钩子，从而使得注册到这些钩子上的函数（即内核模块）能够对数据包进行处理。
### netfilter五个关键钩子（hook）
这五个
>NF_INET_PRE_ROUTING (预路由)  
数据包由网卡驱动接收，经过简单的校验（如 IP 头部检查）后，进入网络层的第一站。  
此时内核还不知道这个包是要发给自己的，还是需要转发给别人的。  
这是进行 DNAT（目标地址转换） 的唯一机会。  

>NF_INET_LOCAL_IN (本地输入)  
内核查看 路由表 后，确认该包的目的 IP 地址属于本机（Loopback 或某个物理网卡）。  
数据包在这里通过最后的过滤检查，然后被剥离网络层头部，将载荷（如 TCP/UDP 数据）移交给传输层，最后送达用户态的 Socket 缓冲区。  

>NF_INET_FORWARD (转发)  
内核查看 路由表 后，发现目的地址不是本机，且系统开启了 ip_forward 功能。  
数据包不会被送往上层协议栈，而是直接进入“转发逻辑”。这里是实现路由器功能的防火墙核心。

>NF_INET_LOCAL_OUT (本地输出)  
本机进程（如你的浏览器）产生数据，经过传输层封装后，进入网络层的第一站。  
数据包在查找路由表之前/期间触发。这里可以对本机发出的包进行初步过滤或标记。  

>NF_INET_POST_ROUTING (路由后)  
无论包是转发的，还是本机产生的，在确定了出口网卡并准备将其传给链路层驱动之前。  
这是包离开协议栈的最后一站，也是执行 SNAT（源地址转换/伪装） 的最佳时机，因为此时已经确定了数据包将从哪个网卡（哪个 IP）发出去。  

数据包流向总结
>入站流量 (发给本机的程序)	PREROUTING -> INPUT  
转发流量 (经由本机发往别处)	PREROUTING -> FORWARD -> POSTROUTING  
出站流量 (本机程序发出)	OUTPUT -> POSTROUTING  
### netfilter优先级常量
#### 主要模块的回调函数
>连接跟踪模块（nf_conntrack）  
回调函数：nf_conntrack_in() 或 nf_conntrack_handle_packet()  
注册钩子：NF_INET_PRE_ROUTING，NF_INET_LOCAL_OUT（优先级为-200）

>NAT模块（nf_nat）  
回调函数：nf_nat_ipv4_in()，nf_nat_ipv4_out()，nf_nat_ipv4_local_fn()等  
注册钩子：根据NAT类型（SNAT/DNAT）注册到不同的钩子点

>iptables相关的内核模块
iptable_filter，其回调函数为iptable_filter_hook()，注册到INPUT、FORWARD和OUTPUT钩子，优先级为0（filter）。  

>iptable_mangle，回调函数为iptable_mangle_hook()，注册到多个钩子点，优先级为-150。

>iptable_raw，回调函数为iptable_raw_hook()，注册到PREROUTING和OUTPUT钩子，优先级为-300。
#### 优先级常量
在内核中，不同模块的回调函数按照优先级顺序执行。以下是以IPv4为例的典型优先级：

| 优先级常量 | 值 | 对应的内核功能 |
|-----------|----|---------------|
| `NF_IP_PRI_RAW` | -300 | 原始包处理（NOTRACK标记） |
| `NF_IP_PRI_CONNTRACK_DEFRAG` | -400 | 连接跟踪前的分片重组 |
| `NF_IP_PRI_CONNTRACK` | -200 | 连接跟踪 |
| `NF_IP_PRI_MANGLE` | -150 | 数据包头部修改 |
| `NF_IP_PRI_NAT_DST` | -100 | 目标地址转换（DNAT） |
| `NF_IP_PRI_FILTER` | 0 | 数据包过滤 |
| `NF_IP_PRI_SECURITY` | 50 | 安全模块处理 |
| `NF_IP_PRI_NAT_SRC` | 100 | 源地址转换（SNAT） |

### 钩子链表 (Hook List)
#### 钩子链表 (Hook List)的概念
在内核里，每一个 Netfilter 钩子（比如 NF_INET_PRE_ROUTING）本质上就是一个名为 nf_hook_entries 的结构体，它内部维护了一个有序的数组或链表。

这个链表里的每一个“节点”并不是一个完整的模块，而是一个叫 nf_hook_ops 的结构体。

链表节点有多个节点
每个节点（nf_hook_ops）主要包含三样东西：    

>回调函数指针 (hook 函数)：指向某个内核模块（如 iptable_filter）中的一段处理代码。  
优先级 (priority)：一个整数，决定了该节点在链表里的位置。  
协议族 (pf)：指定是 IPv4、IPv6 还是 ARP。
#### 钩子链表 (Hook List)是如何运行的
一个钩子上挂着来自“多个模块”注册的“回调函数”
一个数据包被钩子拦住后会，会遍历钩子链表，并调用每一个节点的回调函数
### 连接跟踪（connection tracking）
#### 连接跟踪的数据结构
维护一个名为 conntrack 的哈希表，为每个可跟踪的连接（TCP、UDP、ICMP等）创建一个 struct nf_conn 条目，记录其原始方向、回复方向的状态。
#### 连接的四种状态
>NEW（新连接）： 该包是某个连接的第一个包。例如 TCP 的第一个 SYN 包。

>ESTABLISHED（已建立）： 只要连接的双向都有包发送过，且连接依然活跃，后续所有的包都是这个状态。这是最常见的状态。

>RELATED（关联连接）： 最神奇的状态。它指代那些虽然不是同一个连接、但逻辑上相关的包。

经典例子： FTP 协议。你先建立了指令通道（21 端口），然后服务器主动连你开辟数据通道。这个数据包虽然是“新”的，但由于它和 21 端口的连接有关，被标记为 RELATED。

>INVALID（非法）： 包的逻辑有问题（比如不按顺序发包，或者内存满了无法记录）。这类包通常会被防火墙直接丢弃。  

#### 连接的工作原理
在 PRE_ROUTING 或 OUTPUT 钩子处，nf_conntrack 模块首先查看数据包是否属于已跟踪的连接。它会更新连接状态，并将连接信息（conntrack entry）附着到数据包的内核表示（struct sk_buff）上，供后续NAT和状态防火墙规则使用。

#### connection tracking的作用
Conntrack 在 NAT 中的决定性作用
在 KVM 的 NAT 网络中，Conntrack 是必不可少的。

首包查表： 当虚拟机的第一个包发出时，Netfilter 会去查 NAT 表，决定把源 IP 改成宿主机的 IP。

记录足迹： Conntrack 会立刻在内存里记下：“虚拟机的 10.0.0.2:5000 映射到了宿主机的 1.1.1.1:6000”。

后续自动化： 所有的回程包和后续出站包，不再查 NAT 表，而是直接根据 Conntrack 的这条记录自动进行地址替换。

这就是为什么 iptables -t nat -L 里的规则看起来只有一行，却能处理千万个包的原因。  

### 网络地址转换
依赖：NAT操作需要修改连接跟踪条目中的地址/端口映射，以确保同一个连接后续的所有包（包括回复包）能被正确转换

SNAT (Source NAT)

>操作点：主要在 POST_ROUTING 钩子。  
动作：修改数据包的源IP地址（和可能的源端口）。  
目的：使内网主机使用网关的公网IP访问互联网。MASQUERADE 是SNAT的动态地址版本。

>DNAT (Destination NAT)  
操作点：主要在 PRE_ROUTING 钩子。  
动作：修改数据包的目标IP地址（和可能的目标端口）。  
目的：将发往网关公网IP和端口的请求，转发到内网指定的服务器上（端口转发/负载均衡）。
###  数据包过滤
决策基础：基于注册到钩子点（主要是 LOCAL_IN， FORWARD， LOCAL_OUT）的过滤函数。

判断依据：对数据包及其相关元数据（如IP、协议、端口、连接跟踪状态、内核标记skb->mark、输入/输出接口等）进行逐条规则匹配。

判决结果：

>NF_ACCEPT：允许数据包继续传递到下一个处理阶段或应用。  
NF_DROP：静默丢弃数据包，释放相关资源。  
NF_REJECT：丢弃数据包，并可发送一个错误响应（如TCP RST或ICMP不可达）。  
NF_QUEUE：将数据包传递到用户空间程序处理（需配合libnetfilter_queue）。  
NF_STOLEN / NF_REPEAT：更底层的控制，供特定内核模块使用。


#### 回调机制
函数返回
| Verdict 常量 | 含义 | 协议栈后续动作 |
| :--- | :--- | :--- |
| **NF_ACCEPT** | 接受 | 继续执行下一个 Hook 回调函数或进入下一协议层。 |
| **NF_DROP** | 丢弃 | 释放 skb 内存，数据包在内核中彻底消失。 |
| **NF_STOLEN** | 截留 | 停止 Hook 遍历，内核不再处理，由该回调函数全权负责后续逻辑。 |
| **NF_QUEUE** | 排队 | 将包通过 nfnetlink_queue 传送到用户态空间。 |
| **NF_REPEAT** | 重复 | 重新调用当前 Hook 的第一个回调函数。 |


回调的执行流程

触碰钩子：一个 IP 数据包到达了 PREROUTING 位置。

触发回调：内核发现这个钩子点上挂了一个链表（Linked List），里面登记了三个回调函数（来自 raw、mangle、nat 模块）。

按序执行：内核按照**优先级（Priority）**从小到大，依次调用这三个函数。

内核调用第一个函数（raw 模块）。

函数返回 NF_ACCEPT。

内核调用第二个函数（mangle 模块）。

函数返回 NF_ACCEPT……

最终去向：如果中途某个函数返回了 NF_DROP，内核会立即中止传送，把数据包直接销毁。

### 全流程
#### 流量进入协议栈
当数据包从网卡通过驱动程序进入网络层时，它首先触发 NF_INET_PRE_ROUTING 钩子。

链表遍历：内核调取该钩子下的 nf_hook_entries 数组。

按序执行模块注册的回调：

Raw 模块 (Priority -300)：决定是否要绕过跟踪。

Conntrack (Priority -200)：状态登记。如果是新包，标记为 NEW；如果是回包，标记为 ESTABLISHED。

Mangle 模块 (Priority -150)：修改 TTL 或打 MARK。

NAT 模块 (Priority -100)：如果是外网访问内网，在此执行 DNAT。

判定跳转：只要其中任一函数返回 NF_DROP，包立即销毁；全部返回 NF_ACCEPT 则进入下一步。

#### 路由决策 (Routing Decision)
内核检查目的 IP：  
目的地是本机：进入 INPUT 流程。  
目的地是外部：进入 FORWARD 流程（前提是开启了转发功能）。

#### 栈内流量三大路径
路径 A：数据包发送给主机
触发 NF_INET_LOCAL_IN 钩子：

执行链表：Mangle -> Filter (核心过滤) -> Security。

最终归宿：如果通过了 Filter 的 NF_ACCEPT 判定，包被剥离网络层头部，交给 TCP/UDP 协议栈，最后进入应用的 Socket 缓冲区。

路径 B：数据包转发
触发 NF_INET_FORWARD 钩子：

执行链表：Mangle -> Filter (转发过滤)。

路径 C：本机发出的数据包
本机进程产生数据，触发 NF_INET_LOCAL_OUT 钩子：

执行链表：Raw -> Conntrack -> Mangle -> NAT (DNAT) -> Filter。

意义：控制本机程序对外访问的权限。

#### 路由后处理 (Post-Routing)
无论包是转发的还是本机产生的，最后都会汇聚到 NF_INET_POST_ROUTING 钩子。

链表遍历：Mangle -> NAT 模块 (SNAT/MASQUERADE)。

Conntrack的作用：如果是已建立连接（ESTABLISHED）的包，NAT 模块会直接根据内存中的记录自动替换源地址，无需再次查表。

最终动作：交给网卡驱动，出栈。
