# -EtherChannel-DHCP-OSPF-NAT
<img width="934" height="902" alt="image" src="https://github.com/user-attachments/assets/b2ff4c95-1292-490c-8fbd-d73d4c804ffb" />

结构：网络按照三层结构构建，拥有 200 Mbps 的冗余骨干网 (EtherChannel) 和 1 Gbps 的服务器连接。
连通性：VLAN 10、20 和 30 中的所有客户端都通过 IP Helper 配置从 DHCP 服务器自动接收 IP 设置。
互联网：多亏了 Router_1 上的 NAT 配置和 OSPF 路由，所有 VLAN 都能上网。
安全性：访客 (VLAN 30) 通过访问控制列表 (ACL) 成功与内部企业网络隔离，但仍保持互联网访问权限。
测试结果：执行的测试 (Ping 和 DHCP) 证实网络稳定且功能正常。
-------------------------------------------------------------------------------------------------------------------------------------
网络排错（Troubleshooting）的核心思想其实就是 “顺藤摸瓜”。在网络工程中，我们通常遵循 OSI 模型，
采用 “自底向上” (Bottom-Up) 的方法。
第一方向：物理层与接口 (Layer 1 - Physical)
口诀：线插好了吗？灯亮了吗？
现象：Ping 都不通，或者接口显示 Down。
怎么查：
看拓扑图上的灯是绿的吗？如果是红的，说明线没连对，或者端口 shutdown 了。
命令：show ip interface brief
重点看：Status 和 Protocol 必须都是 Up。如果 Status 是 Admin Down，说明你忘了输 no shutdown。

第二方向：二层交换与 VLAN (Layer 2 - Data Link)
口诀：大家在同一个房间（VLAN）吗？路（Trunk）通了吗？
现象：电脑拿不到 DHCP IP，或者同部门能不能互通。
怎么查：
查 VLAN 划分：端口是不是划错 VLAN 了？
命令：show vlan brief
常见错误：Switch_2 的连接员工的口还在 VLAN 1，没划入 VLAN 10。
查 Trunk：交换机之间的线是 Trunk 吗？允许 VLAN 通过了吗？
命令：show interfaces trunk
常见错误：Switch_ML 和 Switch_1 之间忘了配 Trunk，VLAN 流量过不去。
查 EtherChannel：如果用了聚合链路，状态对吗？
命令：show etherchannel summary
常见错误：两边协议不一致（一边 LACP 一边 PAgP），或者没捆绑成功（显示 SD 而不是 SU）。

第三方向：三层路由与网关 (Layer 3 - Network)
口诀：网关设对了吗？路由器知道路怎么走吗？
现象：能 Ping 通自己，Ping 不通网关；或者 Ping 不通外网。
怎么查：
查 IP 和网关：电脑获取的 IP 对吗？网关是对面路由器的 IP 吗？
常见错误：PC 的网关填错了，或者 DHCP Server 里的网关填错了。
查路由表：路由器知道去往目标的路吗？
命令：show ip route
这是你之前遇到的最大问题： 默认路由 (Gateway of last resort) 是不是指向了 ISP？如果路由表里没有 0.0.0.0/0，路由器就会丢弃上网的数据包。
查 OSPF：邻居建立了吗？网段宣告了吗？
命令：show ip ospf neighbor
常见错误：宣告网段写错了，或者把连接邻居的接口设为了 Passive。

第四方向：服务与转换 (Services & NAT)
口诀：翻译（NAT）工作了吗？中继（Helper）喊话了吗？
现象：Ping 内网通，Ping 8.8.8.8 不通；或者完全拿不到 IP。
怎么查：
查 NAT：私网 IP 转换成公网 IP 了吗？
命令：show ip nat statistics (看有没有命中) 或 show ip nat translations (看转换表)。
常见错误：接口 inside/outside 标反了，或者 ACL 没允许该网段。
查 DHCP Relay：Switch_ML 帮 VLAN 20/30 喊话了吗？
命令：show run (看 vlan 接口下有没有 ip helper-address)。
常见错误：Helper Address 填错了，或者服务器本身 IP 配错。

第五方向：安全控制 (ACLs)
口诀：是不是被门卫（ACL）拦住了？
现象：明明路由通了，Ping 却显示 Destination Host Unreachable 或 U。
怎么查：
ACL 写对了方向吗？(in 还是 out)？
命令：show access-lists (看有没有匹配计数)。
常见错误：忘了加 permit ip any any，导致除了拒绝的，其他的也被默认拒绝了。
终极武器：Packet Tracer 的 Simulation Mode (模拟模式)
这是 Packet Tracer 里的神器！ 当你怎么都找不到问题时：
点击右下角的 "Simulation" 按钮。
发一个 Ping 包。
点击 "Play" 或者一步步 "Forward"。
看信封在哪里变成了“红叉” (X)。
如果 X 在 Switch 上，那是 VLAN/Trunk 问题。
如果 X 在 Router 上，那是 路由/NAT/ACL 问题。
点击那个信封，它会告诉你 "Layer 3: The routing table does not have a route to the destination IP" (路由表没有路)，直接告诉你答案！
老哥总结： 先 Ping 只要通了物理层和二层，再 Show ip route 看路，最后看 NAT 和 ACL。Ping 是你最好的朋友，Show run 是你的说明书。
--------------------------------------------------------------------------------------------------------------------------------------

1. ISP_Router (模拟互联网/ISP)
作用：模拟外部互联网，比如 Google 服务器和 ISP 运营商。
enable
configure terminal
hostname ISP_Router          ! 设置设备名为 ISP_Router

! 配置连接 Google 服务器的接口
interface GigabitEthernet0/0/0
 ip address 8.0.0.1 255.0.0.0   ! 配置 Google 的公网 IP
 no shutdown                    ! 激活接口
 exit

! 配置连接你公司 (Router_1) 的接口
interface Serial0/1/0
 ip address 200.1.1.2 255.255.255.252 ! 配置 ISP 端的公网 IP
 clock rate 2000000             ! 设置时钟频率 (DCE端必须配置)
 no shutdown                    ! 激活接口
 exit

! 配置 OSPF 路由，让路由器之间互通
router ospf 1
 network 200.1.1.0 0.0.0.3 area 0      !宣告 ISP 和公司之间的网段
 network 8.0.0.0 0.255.255.255 area 0  !宣告 Google 网段
 exit
2. Router_1 (公司边缘路由器)
作用：连接公司内网和 ISP，负责 NAT 转换，让员工能上网。

enable
configure terminal
hostname Router_1

! 配置连接 ISP (外网) 的接口
interface Serial0/1/0
 ip address 200.1.1.1 255.255.255.252 ! 配置公司外网口的 IP
 ip nat outside                 ! 【关键】标记这个接口为“外部”接口 (NAT出口)
 no shutdown
 exit

! 配置连接核心交换机 (内网) 的接口
interface GigabitEthernet0/0/0
 ip address 172.30.0.1 255.255.255.0  ! 配置内网网关 IP
 ip nat inside                  ! 【关键】标记这个接口为“内部”接口 (NAT入口)
 no shutdown
 exit

! 配置默认路由 (Default Route)
ip route 0.0.0.0 0.0.0.0 200.1.1.2    ! 告诉路由器：不知道去哪的流量，全扔给 ISP (.2)

! 配置 NAT (让内网私有 IP 能上公网)
access-list 1 permit 10.0.0.0 0.255.255.255  ! 定义一个列表：允许 10.x.x.x 的内网 IP
ip nat inside source list 1 interface Serial0/1/0 overload ! 开启 NAT 转换 (Overload 代表多人共用一个公网IP)

! 配置 OSPF (告诉核心交换机如何路由)
router ospf 1
 network 172.30.0.0 0.0.0.255 area 0   ! 宣告和 Switch_ML 连接的网段
 network 200.1.1.0 0.0.0.3 area 0      ! 宣告和 ISP 连接的网段
 default-information originate         ! 【关键】告诉内网交换机：“我有通往互联网的路，大家都跟我走”
 passive-interface Serial0/1/0         ! 优化：不往 ISP 发送不必要的 OSPF 报文
 exit
3. Switch_NL (核心三层交换机 / Switch_ML)
作用：网络的“大脑”，负责 VLAN 间路由、DHCP 中继、访客安全隔离。
enable
configure terminal
hostname Switch_NL
ip routing                      ! 【最关键】开启三层路由功能 (不做这个就是个傻瓜交换机)

! 创建所有 VLAN
vlan 10
 name medewerkers               ! 命名：员工
vlan 20
 name ICT                       ! 命名：IT部门
vlan 30
 name gasten                    ! 命名：访客
exit

! 配置 VLAN 网关接口 (SVI)
interface vlan 10
 ip address 10.1.0.1 255.255.255.0  ! 员工网关
 exit
interface vlan 20
 ip address 10.2.0.1 255.255.255.0  ! ICT 网关
 ip helper-address 10.1.0.12        ! 【关键】告诉 ICT 的电脑：去 10.1.0.12 找 DHCP 服务器拿 IP
 exit
interface vlan 30
 ip address 10.3.0.1 255.255.255.0  ! 访客网关
 ip helper-address 10.1.0.12        ! 告诉访客电脑：去找 DHCP 服务器
 ip access-group GUEST_BLOCK in     ! 【关键】在访客入口应用 ACL，实施安全隔离
 exit

! 配置上行接口 (连接 Router_1)
interface GigabitEthernet1/0/24
 no switchport                  ! 关闭交换功能，变成路由口
 ip address 172.30.0.2 255.255.255.0 ! 配置 IP
 no shutdown
 exit

! 配置 EtherChannel (连接 Switch_1)
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode active    ! 开启 LACP 链路聚合
 exit
interface Port-channel 1        ! 进入聚合口配置
 switchport mode trunk          ! 设为 Trunk 模式 (允许所有 VLAN 通过)
 exit

! 配置 EtherChannel (连接 Switch_2)
interface range GigabitEthernet1/0/3-4
 channel-group 2 mode active    ! 开启 LACP 链路聚合
 exit
interface Port-channel 2
 switchport mode trunk
 exit

! 配置连接 Switch_3 的接口 (1Gbps)
interface GigabitEthernet1/0/5
 switchport mode trunk          ! 直接设为 Trunk
 exit

! 配置 OSPF (和 Router_1 互通)
router ospf 1
 network 10.0.0.0 0.255.255.255 area 0  ! 把所有 10.x.x.x 的 VLAN 网段都发布出去
 network 172.30.0.0 0.0.0.255 area 0    ! 发布上行链路网段
 exit

! ★ 配置 ACL 安全访问控制列表 (访客隔离) ★
ip access-list extended GUEST_BLOCK
 permit udp any any eq bootpc           ! 允许 DHCP 请求 (否则访客拿不到 IP)
 deny ip 10.3.0.0 0.0.0.255 10.1.0.0 0.0.0.255 ! 禁止访客访问 VLAN 10 (员工)
 deny ip 10.3.0.0 0.0.0.255 10.2.0.0 0.0.0.255 ! 禁止访客访问 VLAN 20 (ICT)
 permit ip any any                      ! 允许访问其他所有 (即互联网)
 exit
4. Switch_1 (接入层 - 访客)
作用：连接访客区域。
enable
configure terminal
hostname Switch_1

! 创建 VLAN (接入层也必须有 VLAN 信息)
vlan 10
 name medewerkers
vlan 20
 name ICT
vlan 30
 name gasten
exit

! 配置上行链路聚合 (连核心)
interface range FastEthernet0/1-2
 channel-group 1 mode active    ! 必须和核心交换机匹配 (mode active)
 exit
interface Port-channel 1
 switchport mode trunk          ! 允许所有 VLAN 通过
 exit

! 配置接入端口 (连接访客 AP/电脑)
interface FastEthernet0/5       ! 假设接的是 F0/5
 switchport mode access         ! 设为接入模式
 switchport access vlan 30      ! 划入 VLAN 30 (访客)
 no shutdown
 exit
5. Switch_2 (接入层 - 员工 & ICT)
作用：连接公司内部员工。
enable
configure terminal
hostname Switch_2

! 创建 VLAN
vlan 10
 name medewerkers
vlan 20
 name ICT
vlan 30
 name gasten
exit

! 配置上行链路聚合
interface range FastEthernet0/1-2
 channel-group 2 mode active    ! 使用组号 2，对应核心配置
 exit
interface Port-channel 2
 switchport mode trunk
 exit

! 配置员工接入端口
interface FastEthernet0/24      ! 假设接的是 F0/24
 switchport mode access
 switchport access vlan 10      ! 划入 VLAN 10 (员工)
 no shutdown
 exit

! 配置 ICT 接入端口
interface FastEthernet0/23      ! 假设接的是 F0/23
 switchport mode access
 switchport access vlan 20      ! 划入 VLAN 20 (ICT)
 no shutdown
 exit
6. Switch_3 (服务器接入)
作用：连接服务器 Server_1，要求 1Gbps 速度。
enable
configure terminal
hostname Switch_3

! 创建 VLAN
vlan 10
 name medewerkers
vlan 20
 name ICT
vlan 30
 name gasten
exit

! 配置上行接口 (1Gbps)
interface GigabitEthernet0/1    ! 使用千兆口
 switchport mode trunk          ! 设为 Trunk
 exit

! 配置服务器接口
interface FastEthernet0/24
 switchport mode access
 switchport access vlan 10      ! 服务器属于 VLAN 10 (员工网段)
 no shutdown
 exit
