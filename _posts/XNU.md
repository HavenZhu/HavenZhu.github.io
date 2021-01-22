## XNU

#### mach
- 进程和线程抽象
- 虚拟内存管理
- 进程间通信和消息传递机制
- 任务调度


#### BSD
建立在mach层之上，提供了POSIX兼容性
- UNIX进程模型
- POSIX线程模型(pthread)及其相关的同步原语
- UNIX用户和组
- 网络协议栈(BSD Socket API)
- 文件系统访问
- 设备访问(通过/dev 目录访问)

#### libkern
在XNU中，设备驱动程序可以使用c++编写，为了支持c++运行时并提供所需要的基类，XNU包含了libkern。

#### I/O Kit
苹果引入了I/O Kit驱动程序框架，这样开发者可以方便编写设备驱动程序


