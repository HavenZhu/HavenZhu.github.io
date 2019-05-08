待学：
- 集线器、网桥、交换机、路由器
- VOIP

杂：
- 环回地址：127.0.0.1，不仅仅是这一个，主机号为127的地址保留为环回地址，向主机号127发送的数据都报不会出现在网络上


- CSMA/CD
载波侦听多路访问/冲突检测
有了交换机之后，基本退出历史舞台，除非网卡工作在半双工环境下，比如通过集线器相连的电脑



- DHCP
dynamic host configuration protocol
应用层协议，基于UDP
DHCP server会保留一份过去IP地址分配情况的表，所以通常情况你在同一个网络下每次被分到的IP地址都是一样的
4个步骤：
c -> s: discovery
客户端发送广播，目的ip地址全1，包含了自己的mac地址，请求ip地址
s -> c: offer:
服务器回复广播，表明打算分配某个ip地址给客户端。
如果客户端曾经分配过，会向上次分配的ip地址发去请求，路由器收到请求后，从本地存储的ip到mac的映射(ARP缓存表)中找到对应的客户端，将请求发送过去。
这样可以避免使用广播
c -> s: request
客户端收到了offer，回复一个request，表明我就要这个ip地址了
s -> c: acknowledgement
服务器收到了request，回复一下没问题，这个就分配给你了

租约过了一半之后，客户端会尝试续约，发送unicast request给服务器，如果服务器响应了，那没问题，续约成功。
如果没响应，会持续尝试，如果一段时间内都没有响应，会发送broadcast request，如果有其他服务器相应了，那没问题，重新绑定，续租下去
如果上一步还是失败了，最后租约过期了，重新从discovery开始走起

理解DHCP与PPPoE的区别
早期的宽带上网需要点击桌面上的一个应用，然后输入账号密码点击连接，实际上用的就是PPPoE
现在宽带上网直接插上网线就完事了，用的是DHCP自动获取IP地址


- ARP
address resolution protocol
通过网络地址获取mac地址
以太网协议规定，局域网中两台主机想要相互通信，必须要知道mac地址，在网络层只能知道ip地址，想要放到链路层中时，无法知道mac地址。
通过ARP包来获取目标主机的mac地址，包含了自己的ip地址和mac地址，目标的ip地址，请求目标的mac地址，以ARP广播的形式发送出去
目标ip地址的机器收到这个广播，回复一个ARP包，告知对方自己的mac地址
此后两台主机就互相知道mac地址，可以直接通信了
当目标机器不在当前局域网中时，回复的mac地址是某个路由器的mac地址，通过路由器转发到对应的目标机器

每台主机或者路由器都有自己的ARP缓存，否则每次发请求都需要通过ARP广播来进行，如果网络中存在环路，可能造成ARP广播风暴
将交换机上的两个端口直接连起来就会造成ARP风暴


- STP
spanning tree protocol
需要理解网桥
防止交换机冗余链路导致的环路
当网络中某个交换机与根网桥有超过一条链路时，STP会只保留距离最短的那条，断开其他链路，防止ARP风暴


- ICMP
internet control message protocol
通过IP层发送消息，提供网络环境中问题的反馈
依附于IP，IP头+ICMP报文
常用的ping，traceroute都是使用了ICMP

理解ping的工作原理
发送一个IP报文，内容是ICMP echo request，收到ICMP echo reply，表明目标可达
发送时会记录一个时间，如果记录时间+超时时间之后还没有收到reply，记录为一次ping失败

## 用c或者c++写一个ping

理解traceroute的工作原理
发送的是UDP报文，收到ICMP ttl超时的回复，回复中会有该路由器的ip地址
每次发送3个UDP请求，第一次ttl为1，第二次ttl为2，从而把一层一层的路线给画出来
因为有些防火墙设置了路由器不回复ICMP报文，所以将udp请求的端口号设置为30000以上，UDP规定端口号不可超过30000，所以会回复一个ICMP端口不可达
收到端口不可达的时候，就表示到达了目标主机


- 网关gateway
因为mac地址只能在同一个局域网内通信，所以每次网关转发都会改变mac地址
静态路由：
转发网关，不改变ip地址
NAT网关，改变ip地址


- RP(路由协议)
路由器中存有路由表，路由表保存了一条条记录：
目的网络：包想去哪里
出口：从哪个端口转发出去
下一跳网关：发到这个网关去

路由协议解决三个问题：
Who?  和哪些路由器交换信息
What? 交换什么信息
When? 什么时候交换信息

内部网关协议，运行于自治系统(AS)内部：
RIP:
基于UDP，一种距离向量的路由选择协议（bellman-ford算法）
通过跳数来选择最近的路由，可能在时间上不是最佳，只能用于小型互联网
Who?  直接相邻的路由器
What? 自己的路由表
When? 每个一定的时间间隔

OSPF:
基于IP，一种链路状态路由选择协议（Dijkstra算法）
Who?  所有的路由器，通过洪泛法
What? 与自己相邻的路由器的链路状态
When? 当链路状态发生变化的时候

外部网关协议，运行于自治系统(AS)之间：
BGP:
基于TCP，一种路径向量的路由选择协议。
Who?  自己的邻站
What? 以AS为基本单位，交换的是AS之间的可达性，指到达某个AS所要经过的一系列AS
When? 先建立TCP连接，通过TCP来交换路由的变化，比如新增了路由，或删除了某个路由


- TCP
TCP头的格式
滑动窗口，区分发送窗口和接收窗口
SACK，选择确认。需要在首部可选字段加上选项，填入不连续的序号首尾边界
流量控制：
利用滑动窗口实现，确认报文中携带的窗口大小为0后，会启动持续计时器，来防止携带新窗口大小的报文段丢失导致发送方持续等待新窗口，接收方持续等待新报文而产生的死锁
计时器时间到了之后发送零窗口探测报文段，该报文段的确认报文段会携带最新的窗口大小

拥塞控制：
慢开始，拥塞避免，快重传，快恢复
在网络层通过路由器采用一定的分组丢弃策略，比如主动队列管理AQM


- UDP


- socket
通常我们所说的socket就是指实现了TCP/IP协议的socket接口，TCP只是协议，socket就是遵守这个协议的实现
经常听到的TCP3次握手，4次挥手，其实都都是建立socket连接过程中的步骤
3次握手：
客户端调用connect()就完成了3次握手，3次握手成功之后，客户端就可以向sockfd读写数据了
服务端收到ACK之后，accept()才能成功从backlog的队列中取到一个sockfd并返回，以后服务端就向这个sockfd读写数据
客户端和服务端都可以通过sockfd到进程中找到对应的socket，socket中存放了目的地址和源地址，就可以收发消息了
4次挥手：
为什么第三次挥手除了标记FIN=1，还标记的ACK=1，并且加上了seq和第二次挥手的seq一模一样？
因为可能存在这种异常情况，如果第三次挥手只发送了FIN，然后出现某种特殊情况导致这个请求滞留在网络中，导致重发第三次挥手，然后第四次挥手，连接断开。
接着该端口又重新建立了连接，并再次进入断开状态，此时收到了滞留在网络中的第三次挥手FIN，然后第四次挥手，直接断开连接，而此时某一方可能还有数据没有发送完毕，从而导致了错误的断开状态。
实际上除了发送的SYN包外，每个tcp包都应该带上ACK标志位，带上此内容并不会增加包的大小

int socket(int domain, int type, int protocol);
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int listen(int sockfd, int backlog);
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
int close(int fd);


- HTTPS
先建立tcp连接，然后在此基础上进行TLS握手
c->s: 发送clientHello消息，包括了支持的最高tls版本，一个随机数，推荐的加密算法，其他的加密信息
s->c: 发送serverHello消息，包括选定的tls版本，此版本两边必须都能支持，一个随机数，选择的加密算法
s->c: 发送证书
s->c: 发送serverHelloDone消息，表明此次协商服务端部分已经结束
c->s: 验证证书通过后，用证书里的公钥加密一个pre-master
此时c和s都有了双方的随机数和pre-master，可以通过上面协商好的加密算法进行加密通信
c->s: ChangeCipherSpec，告诉服务器，从后面开始的消息都是走加密了
c->s: 客户端发送finished消息，包含了一个hash和MAC，这里的内容已经加密过了
服务器解密后验证hash和MAC，有错的话终止连接
s->c: ChangeCipherSpec，告诉客户端，从后面开始的消息都是走加密了
s->c: 服务器发送finished消息，包含了一个hash和MAC，这里的内容已经加密过了
客户端解密后验证hash和MAC，有错的话终止连接

握手流程结束，后续走对称加密了

一切都是为了安全，这也是握手流程这么复杂的原因



1. Negotiation phase:
- A client sends a ClientHello message specifying the highest TLS protocol version it supports, a random number, a list of suggested cipher suites and suggested compression methods. If the client is attempting to perform a resumed handshake, it may send a session ID. If the client can use Application-Layer Protocol Negotiation, it may include a list of supported application protocols, such as HTTP/2.
- The server responds with a ServerHello message, containing the chosen protocol version, a random number, CipherSuite and compression method from the choices offered by the client. To confirm or allow resumed handshakes the server may send a session ID. The chosen protocol version should be the highest that both the client and server support. For example, if the client supports TLS version 1.1 and the server supports version 1.2, version 1.1 should be selected; version 1.2 should not be selected.
- The server sends its Certificate message (depending on the selected cipher suite, this may be omitted by the server).[291]
- The server sends its ServerKeyExchange message (depending on the selected cipher suite, this may be omitted by the server). This message is sent for all DHE and DH_anon ciphersuites.[2]
- The server sends a ServerHelloDone message, indicating it is done with handshake negotiation.
- The client responds with a ClientKeyExchange message, which may contain a PreMasterSecret, public key, or nothing. (Again, this depends on the selected cipher.) This PreMasterSecret is encrypted using the public key of the server certificate.
- The client and server then use the random numbers and PreMasterSecret to compute a common secret, called the "master secret". All other key data for this connection is derived from this master secret (and the client- and server-generated random values), which is passed through a carefully designed pseudorandom function.
2.
- The client now sends a ChangeCipherSpec record, essentially telling the server, "Everything I tell you from now on will be authenticated (and encrypted if encryption parameters were present in the server certificate)." The ChangeCipherSpec is itself a record-level protocol with content type of 20.
- Finally, the client sends an authenticated and encrypted Finished message, containing a hash and MAC over the previous handshake messages.
- The server will attempt to decrypt the client's Finished message and verify the hash and MAC. If the decryption or verification fails, the handshake is considered to have failed and the connection should be torn down.
3.
- Finally, the server sends a ChangeCipherSpec, telling the client, "Everything I tell you from now on will be authenticated (and encrypted, if encryption was negotiated)."
- The server sends its authenticated and encrypted Finished message.
- The client performs the same decryption and verification procedure as the server did in the previous step.
4.
- Application phase: at this point, the "handshake" is complete and the application protocol is enabled, with content type of 23. Application messages exchanged between client and server will also be authenticated and optionally encrypted exactly like in their Finished message. Otherwise, the content type will return 25 and the client will not authenticate.