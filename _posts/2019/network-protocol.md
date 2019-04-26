1. DHCP
dynamic host configuration protocol
DHCP server会保留一份过去IP地址分配情况的表，所以通常情况你在同一个网络下每次被分到的IP地址都是一样的
4个步骤：
c -> s: discovery
客户端发送广播，包含了自己的mac地址，请求ip地址
s -> c: offer:
服务器回复广播，表明打算分配某个ip地址给客户端。
如果客户端曾经分配过，会向上次分配的ip地址发去请求，路由器收到请求后，从本地存储的ip到mac的映射中找到对应的客户端，将请求发送过去。
这样可以避免使用广播
c -> s: request
客户端收到了offer，回复一个request，表明我就要这个ip地址了
s -> c: acknowledgement
服务器收到了request，回复一下没问题，这个就分配给你了

租约过了一半之后，客户端会尝试续约，发送unicast request给服务器，如果服务器响应了，那没问题，续约成功。
如果没响应，会持续尝试，如果一段时间内都没有响应，会发送broadcast request，如果有其他服务器相应了，那没问题，重新绑定，续租下去
如果上一步还是失败了，最后租约过期了，重新从discovery开始走起