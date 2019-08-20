把socket好好写一写，可以结合tcp一起

讲一下linux文件
标准输入、输出、错误为3个默认打开的文件

I/O设备被抽象为文件
所有的输入输出都被抽象为对文件的读和写
 
socket是一种文件类型
目录也是一个文件类型，包含一组链接的文件，这个文件至少包含两个链接条目，"."为自身的链接，".."为目录层次结构父目录的链接


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