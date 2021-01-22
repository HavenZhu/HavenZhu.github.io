把socket好好写一写，可以结合tcp一起

### socket是什么？
这篇说一说socket，这玩意大部分人都是只知其表不知其里，都能说个一二，但是稍微深入就被问住了。

socket是什么？socket其实是一种文件格式。这里先要讲一讲文件。

#### 文件

在linux系统中，所有的I/O设备被抽象为文件，包括我们可以理解的磁盘，以及不可以理解的网络，终端等。所有的输入、输出被抽象为对相应文件的读写。这样抽象的好处是linux内核可以提供一套底层的应用接口——UNIX I/O——使得所有的输入输出都能以统一的方式进行。

理解了什么是文件，我们再来描述一下socket：一种用来与另一个进程进行跨网络通信的文件。

作为引申，目录也是一个文件类型，包含一组链接的文件，这个文件至少包含两个链接条目，"."为自身的链接，".."为目录层次结构父目录的链接。每个进程在启动的时候，都会默认打开3个文件，标准输入、标准输出、标准错误。
（文件也单独写一篇）

服务端调用了listen()，此时创建了一个监听socket，内核为每个监听socket创建了2个队列，一个为未完成连接的队列，一个为已连接队列，调用完listen()之后客户端就可以与服务端进行连接
客户端发送connect()，开始3次握手，3次握手和connect()函数本身没有什么关系，connect()只是负责通知内核，内核负责3次握手。
内核3次握手结束后，客户端connect()返回。已连接队列中增加一员，增加的就是新建立的连接，该连接在调用accept()之后返回。
通常我们所说的accept()阻塞，其实就是已连接队列为空，待connect()结束后，已连接队列中有了内容，accept()取到了连接然后才会返回。



socket接口：
int socket(int domain, int type, int protocol);
参数：domain为IP 地址类型，常用的有 AF_INET 和 AF_INET6；type为套接字类型，常用的有SOCK_STREAM和SOCK_DGRAM；protocol表示传输协议，常用的有IPPROTO_TCP和IPPTOTO_UDP
返回：非负socketfd
socket接口返回的是主动套接字，主动套接字存在于客户端，当服务端使用socket创建了套接字之后，使用listen()方法将该套接字转为监听套接字

int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
参数：客户端创建的socketfd；服务端的socket地址，由客户端填好后作为参数传入；addr的长度
返回：0 - 成功，-1 - 失败

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
参数：服务端socketfd；服务端socket地址；addr的长度
返回：0 - 成功，-1 - 失败

int listen(int sockfd, int backlog);
参数：服务端socketfd；请求队列长度，请求队列达到该长度后，服务器就会拒绝连接请求
返回：0 - 成功，-1 - 失败

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
参数：服务端socketfd；客户端空的socket地址，accept方法返回后，此地址会记录客户端的相关地址；addr的长度
返回：非负connectfd，此后通过此connectfd与客户端进行通信，-1 - 失败
**注意区分connectfd与socketfd，socketfd只有一个，connectfd在每次接收一个连接请求的时候都会创建一个，用于与相应的客户端通信**