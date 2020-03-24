## 理解不够深刻的内容

- 这个写法会出什么问题： @property (copy) NSMutableArray *array;
1、添加,删除,修改数组内的元素的时候,程序会因为找不到对应的方法而崩溃.因为 copy 就是复制一个不可变 NSArray 的对象
2、使用了 atomic 属性会严重影响性能

- 为什么block要使用copy修饰
默认情况下，block会存档在栈中，所以block会在函数调用结束被销毁，在调用会报空指针异常。
如果用copy修饰的话，可以使其保存在堆区，它的生命周期会随着对象的销毁而结束的。只要对象不销毁，我们就可以调用在堆中的block。

- nstimer什么时候释放
timer会强引用target，一般情况下target就是self，所以导致self不会被释放，需要手动调用invalidate方法
timer被加入到runloop中，不会自动释放

- assign和weak的区别
如果用来指向oc对象，都不会增加引用计数
区别是weak指针在指向的对象被释放时会自动置为nil，assign指针不会被置为nil，从而出现野指针


- 一次完整的http请求过程
DNS解析获取到对象的ip地址
应用层：创建http请求报文
运输层：创建tcp报文，在http的基础上加上了源端口和目的端口号
网络层：创建ip报文，在tcp报文的基础上加上了源ip地址和目的ip地址
链路层：拿到ip报文后，创建以太网帧，需要目标mac地址，同一个子网状态下，通过arp协议可以拿到目标mac地址，不同网络情况下，目标mac地址填自己的网关路由地址，最终路由负责转发到目标地址
目标受到请求之后，解析出http请求的内容，封装好response，通过相同的路径返回过来

- 本地通知
UNUserNotificationCenter用于管理app和app extension通知相关任务。
你可以在任意线程同时调用该方法，该方法会根据任务发起时间串行执行。

- 点击一次界面时，触摸事件是如何传递到被点击的那个view的

- 客户端如何统计消息的到达率


- NSOperation中的main()和start()方法的区别
如果想要自己管理operation，一般需要重写start方法来管理operation的状态
正常会把operation加入到queue中，这时候只需要重写main方法，operation的状态会由queue来管理
operation的concurrent属性将要被废弃，使用synchronize属性来决定此operation是否需要在执行的线程中同步还是异步执行

正常的流程是这样的，调用start方法，设置isExecuting为YES，调用main()方法，main返回之后，设置isFinished，operation就结束了
自己重写start()方法，不调用父类的start()方法，自己决定什么时候结束operation

- GCD会碰到什么锁相关的问题


- 单例需要怎么实现，才能确保不会通过alloc init创建出新的对象
+ (instancetype)sharedInstance
{
    return [[self alloc] init];
}
- (instancetype)init
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super init];
        instance.height = 10;
        instance.object = [[NSObject alloc] init];
        instance.arrayM = [[NSMutableArray alloc] init];
    });
    return instance;
}
+ (instancetype)allocWithZone:(struct _NSZone *)zone
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super allocWithZone:zone];
    });
    return instance;
}


- socket连接建立的过程
服务器：创建socket(domain, type, protocol)，bind(host, port)，listen(port)，accept()等待连接
客户端：创建socket()，connect()

客户端调用connect()的时候，发送了SYN，此时connect()还没有返回
服务器受到SYN后，返回ACK+SYN，accept()还阻塞着
客户端收到ACK+SYN之后，返回ACK，connect()结束
服务器受到ACK之后，accept()结束，连接建立


## 清单
- runtime（对象，类，消息）
对象，类在runtime层面都是objc_object，有一个isa属性，对象指向类，类指向元类，元类指向根元类，根元类指向自身
Class在runtime层面是是objc_class

category
1. 附加category到类的工作会先于+load方法的执行
2. +load的执行顺序是先类，后category，而category的+load执行顺序是根据编译顺序决定的。
3. category的方法被加到的原方法列表的上面，所以会有覆盖的效果
4. category不能添加实例变量，因为对象的内存布局已定，在运行时不允许修改

class_addIvar() 函数用于给类添加成员变量
This function may only be called after objc_allocateClassPair and before objc_registerClassPair.
Adding an instance variable to an existing class is not supported.


- KVO
- 内存（weak，autoreleasepool等）
- runloop
- 锁
- 多线程

## 基础
- nonatomic和atomic，atomic一定安全吗
atomic底层是通过锁机制来实现的，atomic仅仅保证了本属性getter/setter方法的原子性，那就是一旦开始了，一定能得到结果，当调用属性的其他方法，或者对象多个atomic属性之间，是没法保证线程安全的。

- strong和copy的用法，strong是深拷贝吗？对于字符串使用strong会有什么问题？对于可变数组使用copy这样会有什么问题？
对NSString属性使用copy修饰是为了当属性被赋值了可变对象时，本对象的改变不会修改可变对象的内容。
对系统原生对象类型：
copy返回的是不可变对象，如果target是不可变，相当于retain count + 1；如果target可变，深复制对象。
mutable copy返回可变对象，无论target可不可变，均深复制对象。
对NSArray的拷贝，不论是深是浅，对数组中的对象都是浅拷贝，想要深复制数组对象，需要增加额外的操作，例如一个for循环。
实现NSCoping协议可以给自定义对象实现copy方法

- @synthesize和@dynamic是什么？用法？
@dynamic + private ivar = @synthesize, 通常是为readonly属性重写get方法
给protocol添加属性时，在实现类中需要使用@synthesize
给protocol中添加类属性时，在实现类中需要使用@dynamic

对readonly的属性，如果重写了get方法，不会自动生成ivar，需要使用@synthesize
对readwrite属性，如果同时重写了get和set方法，不会自动生成ivar，需要使用@synthesize

- 为什么不推荐在init方法中使用点语法，推荐使用ivar
因为在子类中可能会重写属性的set方法，
假设父类在init方法中设置了属性foo，子类重写了setFoo方法，
那么当子类初始化时，会有一行self = [super init]，在这里实际会把foo设置为子类中重写的结果，而不是父类init方法中设置的值

- 代理是什么？协议/接口 模式应用场景？
delegate在runtime层也是一个objc_object，会存储定义在@protocol中的必选方法和可选方法
protocol中也可以定义属性，不过不会自动生成ivar和get/set方法，所以需要在实现类中添加@synthesize，并手动实现get/set方法

- 实现响应链传递的两个方法？（hitTest和pointInside）在哪些地方使用过？
最常用的应该是用来扩大按钮的点击区域

- alloc&init是什么
alloc分配一块对象大小的内存，初始化isa指针，返回指向对象初始化地址的指针
init方法啥都没做，传入什么返回什么，所以可以不写

- 编译器的优化，分析被编译器优化的部分

- OC对象内存中的分配，OC对象所占用的空间
对象的大小 = isa + ivars

- layer和view

- 如何判断上传下载是否完成
用MD5验证文件的完整性


- Auto Layout
基于Cassowary算法（有空研究一下）
可以方便的描述界面上各个视图之间的关系，提供一种简单的写法，在运行时动态的计算视图的具体位置
Layout Engine生命周期：
(自下而上)update constraint -> 重新计算layout -> superView.setNeedsLayout()
(自上而下)Deferred Layout Pass -> 容错处理，比如有约束冲突 -> 自上而下调用layoutSubviews() -> 得到新的frame
(自上而下)UI update

我们操作界面的时候设定或者更新的都是constraint，

- ARC时代的内存泄漏
如何检测
非oc对象内存泄漏
常用框架内存泄漏

- NSDictionary和NSCache
NSDictionary要求key实现NSCoping协议，会copy key
NSCache不会对key执行copy操作

- volatile关键字
一个定义为volatile的变量是说这变量可能会被意想不到地改变，这样，编译器就不会去假设这个变量的值了。
精确地说就是，优化器在用到这个变量时必须每次都小心地重新读取这个变量的值，而不是使用保存在寄存器里的备份。

- 如何把一个包含自定义对象的数组序列化到磁盘？
对象实现NSCoding协议，使用nskeyedarchiver和nskeyedunarchiver进行归档和解归档


## 中级


连信
- 消息
- 小程序
- 卡顿
- 模块化
- UI: 朋友圈，视频通话，红包，图片编辑
- 启动时间优化
- 做了什么性能优化


连信
- 消息体系
sync机制：通过版本号检测当前数据
时机：在个人或群的信息发生变化，网络连接变化，后台切换到前台
流程：发送http请求拉取sync内容，拉到后分发给handler进行处理，可能会需要连续sync
使用sync的优点是什么

sync优化，减少一次插入数据库的量，防止阻塞队列，导致主线程无法通过队列读取数据库数据，从而卡在主屏幕上
为什么使用串行队列不使用锁


收发消息：通过心跳保持tcp长连接，进行收发消息，如果没有长连接，通过iOS通知。每次打开app时，会进行sync操作
多种消息类型

- jenkins自动打包，脚本实现，重签名
- 安全，pk,ck,sk,iv（模拟的https）
- 模块化
- 小程序容器，cordova
定义了一个model来描述小程序，然后使用webview加载出来
小程序包使用webpack打包，讲一哈webpack
说是容器，其实就是controller + webview，本质上还是webview加载webpack打包出来的index.html
小程序分享出去是一个富文本消息，别的用户点击此消息根据url获取对应的小程序并在容器中加载出来

- 卡顿检测
利用runloop observer，无法获取调用栈信息
另开一个线程

- protobuf
- 理解连信中的长连接消息收发过程
- tcpsocket
- udpsocket
socket发送data可能存在不完整的问题
- TCP连接的建立和释放
建立：SYN=1 + seq=x -> ACK=1 + SYN=1 + seq=y + ack=x+1 -> ACK=1 + seq=x+1 + ack=y+1
释放：FIN=1 + seq=u -> ACK=1 + seq=v + ack=u+1
				   -> FIN=1 + ACK=1 + seq=w + ack=u+1 -> ACK=1 + seq=u+1 + ack=w+1

模块化先剥离底层模块，比如log，database，network
业务模块肯定会依赖底层的模块，但业务模块之间不应该存在横向依赖
模块化过程中发现了什么问题？
为了方便直接传递对象，增加了类之间的依赖，比如logmanager依赖了很多model，实际上只是使用model的几个属性，需要改为dictionary

基础
- runtime:method swizzling, associated object，
- runloop
- afnetworking 														done
- masonry															done
- sdwebimage														done
- FMDB
- 动画
- osx和ios内核
- 多线程，GCD，NSOperation，pthread，NSThread
- 性能调优  UIKit Object handling, Rendering, Layout
- 内存管理，MRC(retain, release, autorelease), ARC(@autoreleasepool)	done
- block
- cocoapods
- 锁，spinlock，@syncronized, nsrecrusivelock等
- 事件响应链
- APNS, remote notification, local notification						done
- https																done
- multicastdelegate													done
- redis等其他数据库
- 缓存，内存缓存，硬盘缓存
- javascriptcore
- 用runtime做了什么


dispatch_async + 自定义串行队列：会重启一个线程来执行队列中的任务吗

1. NSURLProtocol拦截http请求，再使用NSURLSession发新的请求出去，使用NSURLProtocol的client在回调中进行处理
2. NSURLConnection被弃用的原因：下载时内存占用大，断点续传操作不便，请求只能开始和取消
3. NSURLSession


沙盒目录里有三个文件夹：
Documents——存储应用程序的数据文件，存储用户数据或其他定期备份的信息；
Library下有两个文件夹，Caches存储应用程序再次启动所需的信息，Preferences包含应用程序的偏好设置文件，不可在这更改偏好设置；
temp存放临时文件即应用程序再次启动不需要的文件。


[super class], 理解super关键字


APNS
- 客户端注册remotenotification，从APNS获取devicetoken
- 将devicetoken发给provider
- provider发送消息给APNS，APNS根据devicetoken及其他信息，将通知发给我们的设备，我们的设备根据通知内容，弹出通知
连信中的通知：
有长连接时，服务器直接通过长连接推送给app，app弹出本地通知
没有长连接，走APNS


- sdwebimage
下载是基于nsurlsession
内存缓存基于NSCache，key为url，注册了通知，会在接收到内存警告的时候清空缓存
默认情况没有磁盘缓存大小的上限，默认磁盘缓存时长是一周，均可以修改
option中可以设置失败是否重试，如果不重试，在一次失败后，会被加到failedUrls数组中，再碰到url相同的就不会去下载了
多次下载同一个图片时，只有第一次会触发下载的请求，后续只会把回调添加到对应url的回调数组中，当url下载完成时，回调数组中的所有block。通过dispatch_barrier_sync来确保添加回调到数组中和触发回调的顺序
磁盘缓存基于文件读写，一个key对应一个文件
加载gif时，sdwebimage会把文件全部读到内存，所以内存消耗会大一些；yywebimage会一张一张的读，所以cpu开销会大一些
yywebimage在此基础上提高了性能，磁盘缓存基于sqlite+文件，写性能sqlite高于写文件，读性能在数据小时sqlite性能好，数据大时读文件性能好，所以综合起来进行缓存


- multicastdelegate
本质上是利用了runtime的消息转发，当一个类没有实现某个方法时，通过forwardInvocation将消息转发出去，在此方法中遍历添加的delegate，依次执行方法


- block是什么
- block是如何拦截对象的
- __block的原理
- block循环引用是如何产生的
- 在block使用self一定会被持有吗

- 局部非对象的变量在block中使用时会被拷贝，所以不让改变值，除非加上__block，此时实际拷贝的是指针
- 局部对象的变量在block中使用时也会被拷贝，所以为了防止误操作，在编译时就不允许改变对象指向的地址，但是可以改变对象的属性
- block捕获对象property，也是copy一份在内部使用


nsoperation被加到nsoperationqueue中，本质上还是由GCD来管理，与runloop没有什么关系。
所以应该避免在nsoperation中使用runloop。
想使用runloop，自己创建一个线程，自己来管理。


MBProgressHud中的runloop
mach kernel, pthread, block
nstimer, uievent, autorelease
caanimation


- afnetworking

AFNetworkReachabilityManager 是否用到了runloop？

NSURLSession   根据request生成task
AFURLSessionManager  对 NSURLSession 封装了一层， 更方便的生成各种task，task基于request生成
AFHTTPSessionManager  继承于 AFURLSessionManager， 更方便http相关操作

AFURLSessionManagerTaskDelegate    每一个task有单独的delegate来处理回调，放在一个字典里

AFHTTPRequestSerializer    生成request的相关操作，配置各种header body之类
AFHTTPResponseSerializer    处理response的相关操作

afnetworking从2到3有什么变化，为什么做这样的变化



- KVO的底层实现原理，能否手动实现一个KVO，怎么处理？
KVO例子：视频通话过程中，监听window的屏幕常亮属性，防止由于其他操作关闭了屏幕常亮

- 响应链可以用来做一些delegate做的事，比如将cell中按钮的点击事件传给controller

- 可以用什么方法保证线程安全

- 锁:
- OSSpinLock: 存在优先级反转的问题（低优先级线程拿到锁，高优先级线程忙等，浪费cpu时间，低优先级线程获取不到cpu时间无法处理完）
- dispatch_semaphore: 就是信号量，0开始执行，-1就等待。会进行上下文切换
- pthread_mutex: 互斥锁
- NSLock: 内部封装了pthread_mutex，因为多了一层方法调用，所以性能比pthread_mutex差一点
- NSCondition: 用这玩意儿不如用dispatch_semaphore
- pthread_mutex(recursive): 递归锁，一般情况下，除了递归调用的方法里想要加锁，其他情况很少使用。比一般的锁增加了一个count属性，每次获取锁的时候count加1，释放时count-1，为0时可以被其他线程获取。
- NSRecursiveLock: 同上
- NSConditionLock
- @synchronize：用传入的对象锁，锁存在map中，key是经过计算的对象的内存地址（花点时间看一下源码，深入理解一下）

- 自己实现锁？自旋锁？互斥锁

- 一种不是锁但是可以起到锁作用的方法
串行队列

- block可以修改外部变量吗？(为什么直接就可以修改全局变量？使用__block才可以修改局部变量？)
- block底层实际上是什么？block有几种类型？block使用时的注意事项
都是c的struct，stack, malloc, global

- OC对象本质是什么？
OC所有的对象，类在runtime层都是c struct，都是objc_object

- Runtime是什么？用到过runtime吗？
protocol, property, method, object, class

- SDWebImage底层实现？还有它的缓存机制?
- AFNetWorking在成功回调后是处于什么线程之中？为什么会这样？

- 脚本打包与重签名
签名的原理
1. 在你的 Mac 开发机器生成一对公私钥，这里称为公钥L，私钥L。L:Local
2. 苹果自己有固定的一对公私钥，跟上面 AppStore 例子一样，私钥在苹果后台，公钥在每个 iOS 设备上。这里称为公钥A，私钥A。A:Apple
3. 把公钥 L 传到苹果后台，用苹果后台里的私钥 A 去签名公钥 L。得到一份数据包含了公钥 L 以及其签名，把这份数据称为证书。
4. 在苹果后台申请 AppID，配置好设备 ID 列表和 APP 可使用的权限，再加上第③步的证书，组成的数据用私钥 A 签名，把数据和签名一起组成一个 Provisioning Profile 文件，下载到本地 Mac 开发机。
5. 在开发时，编译完一个 APP 后，用本地的私钥 L 对这个 APP 进行签名，同时把第④步得到的 Provisioning Profile 文件打包进 APP 里，文件名为 embedded.mobileprovision，把 APP 安装到手机上。
6. 在安装时，iOS 系统取得证书，通过系统内置的公钥 A，去验证 embedded.mobileprovision 的数字签名是否正确，里面的证书签名也会再验一遍。
7. 确保了 embedded.mobileprovision 里的数据都是苹果授权以后，就可以取出里面的数据，做各种验证，包括用公钥 L 验证APP签名，验证设备 ID 是否在 ID 列表上，AppID 是否对应得上，权限开关是否跟 APP 里的 Entitlements 对应等。


## 进阶

- 组件化
组件化本质是解耦，两种常见方式
1. url-scheme
通过苹果提供的openurl方法，统一管理内部和外部的跳转

2. target-action
利用runtime的反射获取类和方法
可以统一管理异常情况
需要给每个target创建一个target_xx类，还需要给xx类创建一个category

将1和2结合起来，外部使用1，内部使用2，1最终还是会转化为2，从而可以统一管理跳转。


- 卡顿监测
基于runloop observer
主线程添加了观察者，观察主线程runloop的afterwaiting到beforewaiting回调，回调触发信号量
进入beforewaiting后，信号量进入：dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER)
runloop被唤醒后，进入afterwaiting，触发_semaphore，进入dispatch_semaphore_wait(_semaphore, waitingTime)
如果超过waitingtime，那么本次loop记为超时，将堆栈收集后上传
为了准确收集主线程堆栈，在子线程中需要定时记录主线程堆栈
腾讯Matrix是每隔50ms记录一次，检测到卡顿后，查询记录的20次堆栈，找到最多的相同栈顶方法，标记该方法为卡顿

子线程等待主线程的信号量，超过400ms记录为一次卡顿。Matrix是超过2秒记录为一次卡顿


- runloop
一个 Mode 可以将自己标记为”Common”属性（通过将其 ModeName 添加到 RunLoop 的 “commonModes” 中）。
每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 “Common” 标记的所有Mode里。
runloop在主线程操作autoreleasepool是发生以下几处：
entry: push，优先级最高，发生在其他回调之前
beforewaiting: pop + push，优先级最低，发生在其他回调之后
exit: pop，优先级最低，发生在其他回调之后

Source1 (基于 mach port 的) 用来接收系统事件，通常的触摸事件就是source1
当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。
SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。
随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。
_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。
通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

界面更新发生于beforewaiting回调，会调整所有标记为需要更新的view/layer样式，更新UI


- 动态链接
内核创建虚拟地址空间
将可执行文件映射到虚拟地址空间
将dyld映射到虚拟地址空间
dyld接手，将各种库load进来
runtime接手，处理出各种struct方便访问（runtime只是其中的一个分支）
dyld接手，init各种库
runtime接手，执行+load函数
dyld接手，进入真正的main函数


- app的启动过程
加载可执行文件
动态链接器链接动态库
runtime绑定了dyld的一些回调，会在map，init，unmap image的时候进行处理
mapimage：
image被加载到内存后，runtime会对类、category、protocol、方法等进行处理，处理完成后，原本二进制的内容就以各种struct的形式存在了

init：
调用类的+load方法，父类优先子类，category在父类之后，没有申明+load方法就不会调用
调用+load方法的前后会调用autoreleasepoolpush和autoreleasepoolpop，所以在+load方法中不需要手动添加autoreleasepool

unmap：
将image移除

- app启动时都做了什么
1. main函数执行前
加载可执行文件
加载动态链接库
运行时初始化，包括类注册，category注册等
初始化，包括了执行 +load() 方法、attribute((constructor)) 修饰的函数的调用、创建 C+ 静态全局变量


2. main函数执行，到didFinishLaunchingWithOptions中首屏渲染相关方法执行完
首屏所需配置文件的读写
首屏所需数据的读写
首屏渲染的计算


3. didFinishLaunchingWithOptions中其他方法执行完
非首屏相关的配置文件、数据读写及计算


- 启动性能优化
1. main函数执行前
减少动态库数量
减少不必要的+load方法，放到+initialize方法中进行
控制c++全局变量的数量

2. main函数执行，到didFinishLaunchingWithOptions中首屏渲染相关方法执行完
只读写和渲染首屏需要的相关内容，其他内容放到首屏渲染之后再去处理

3. didFinishLaunchingWithOptions中其他方法执行完
用户已经可以看到首屏了，需要优化会卡住主线程的操作


- KVO
通过添加中间类，重写了class方法，属性的set方法，dealloc方法，增加了isKVOA方法
在重写的set方法中增加了willChangeValueForKey和didChangeValueForKey方法，来通知观察者
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key; 通过比对key并返回NO，可以关闭自动KVO，如果不关闭，再重写了set方法，会触发两次KVO
然后手动重写set方法，加上willChangeValueForKey和didChangeValueForKey方法，可以手动启动KVO
两个方法必须同时加上，缺少任意一个都不会触发KVO


- 断点续传和下载
上传：服务器会告知客户端传了多少，客户端从该位置继续向后传
下载：在http请求头中加上Ranges，表明本次下载从哪里开始


- 响应链
从被触摸的控件开始，比如一个UITextfield或者一个UIButton，向上找UIView，UIViewController，UIWindow，UIApplication，UIApplicationDelegate


- https
客户端发出请求到服务器
服务器返回证书给客户端
客户端对比服务器发过来的证书与本地内置的证书，确定证书没问题
证书没问题的话从证书中提取出公钥
本地生成对称秘钥，用公钥加密该对称秘钥然后发送给服务器
服务器用自己的私钥解密，取出客户端生成的对称秘钥

- 如果让你去设计SDWebImage的缓存机制，怎么去设计？
- 如果一个自定义的Cell可能被用在多处，每处后台返回的数据源模型（json原始数据）字段都是不同的，如果让你去设计一种Model去兼容所有的业务需求，即不管原始数据什么样都能一样进行解析赋给Cell，你怎么去设计？
- __block 能修改外部变量根本原因是什么？
- 自动释放池中的变量什么时候释放
- 动态库和静态库的区别
- block根据内存区间分为几种类型？
- 当一个tableView上面加载很多Cell，但它们同时用SDWebImage去加载同一张网络图片（即URL相同），SD的底层会请求多次网络请求吗？如果不会，它是怎么做到同一张图片下载完成后进行多次成功的回调并同时加载到Cell上呢？

## 框架
- cocoa愿景MVC架构
- 实际开发MVC存在的问题
- 面向协议编程的MVP架构思想
- 双向绑定的MVVM架构思想（面试题：MVVM是如何实现双向绑定的？）
- RAC与MVVM双剑合璧的体验

## 动画
- 核心动画中仿射变换
- OpenGL 中模型视图变换
- 3D数学--旋转/平移/缩放的数学原理
- 核心动画中的特殊图层
- 点赞动画
- 弹幕动画
- 礼物动画
- 常用的动画，帧动画，UIView的类方法动画



## 题库

- 什么情况使用 weak 关键字，相比 assign 有什么不同？
- 怎么用 copy 关键字？

- 如何让自己的类用 copy 修饰符？如何重写带 copy 关键字的 setter？
- @property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的
- @protocol 和 category 中如何使用 @property
- runtime 如何实现 weak 属性
首先weak的作用是在指向的对象释放之后，自动置为nil，也就是不会增加对象的引用计数
为了做到这一点，就需要有一个地方来存放所有指向某个对象的weak指针，这个结构里面存放了所有指向某个对象的weak指针，这个结构可以是个数组
为了把这个数组和被指向的对象关联起来，需要使用字典，key就是被指向对象的地址，value就是weak指针的数组
当被指向的对象释放了之后，会从字典中以对象的地址为key查找所有的weak指针，将所有的指针都置为nil

SideTables在iOS中有8个，以disguise之后的对象地址作为index，取到对应的SideTable，对象肯定不止8个，所以会有多个对象使用同一个SideTable
一个SideTable中管理了一个weak_table，这是真正管理弱引用的地方，所有的弱引用都在weak_entries中管理

weak_entries是个数组，index是通过被引用的对象的内存地址计算出来的
数组的每个对象都是一个weak_entry，weak_entry中存放了该对象的所有的弱引用，如果weak_entry的对象不是要查找的对象，就将index+1，继续向后查找
找到满足条件的weak_entry之后，
- 如果是被引用的对象置为了nil，那么要将weak_entry中管理的所有弱引用的指针置为nil
- 如果是某个弱引用指针被位置nil，从weak_entry管理的referrer数组中找到需要操作的指针，然后置为nil

当weak对象过了作用域被释放时，会调用objc_storeWeak方法，从weak_entry中解除此weak引用，如果只有当前一个weak引用，还需要从weak_entries数组中移除与被引用对象相关的这条记录

当弱引用数量小于4的时候，使用不可变数组管理所有的弱引用指针，当数量大于4的时候，使用可变数组，应该是出于性能考虑。
使用定长数组时，就是遍历数组；使用可变数组时，使用对象的hash_pointer作为index进行依次查找

struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};

- @property中有哪些属性关键字？/ @property 后面可以有哪些修饰符？
- weak属性需要在dealloc中置nil么？
不需要，weak属性会在被关联的对象release时被置为nil，不需要手动操作

- taggedPointer，isa（nonpointer）

- @synthesize和@dynamic分别有什么作用？
- ARC下，不显式指定任何属性关键字时，默认的关键字都有哪些？
- 用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题？
- 对非集合类对象的copy操作
- 集合类对象的copy与mutableCopy
- @synthesize合成实例变量的规则是什么？假如property名为foo，存在一个名为_foo的实例变量，那么还会自动合成新变量么？
- 在有了自动合成属性实例变量之后，@synthesize还有哪些使用场景？
什么情况下不会auto synthesis（自动合成）？
1. 同时重写了 setter 和 getter 时
2. 重写了只读属性的 getter 时
3. 使用了 @dynamic 时
4. 在 @protocol 中定义的所有属性
5. 在 category 中定义的所有属性
6. 子类重载父类的属性，必须在子类中synthesize

- objc中向一个nil对象发送消息将会发生什么？
没问题

- objc中向一个对象发送消息[obj foo]和objc_msgSend()函数之间有什么关系？
[obj foo]最终会转换成objc_msgSend()的形式进行发送，第一个参数是self，第二个是cmd即本方法的名字
方法调用先从本类的缓存和方法列表中查找，找到了会加到缓存中，下次就不走方法列表中一个个查了
本类查不到，从父类的缓存和方法列表中查找，找到了会加到本类的缓存中，父类一层一层查
父类也查不到，走动态方法解析，可以在运行时增加一个方法，加完方法之后会回到最开始的步骤重新处理，此时在方法列表中已经可以找到刚添加的方法了。
动态方法解析也失败了，走消息转发，multicastdelegate就是使用的消息转发，来将消息发送给所有的delegate，达到多播的效果。


- 什么时候会报unrecognized selector的异常？
- 一个objc对象如何进行内存布局？（考虑有父类的情况）
isa + 父类ivars + 子类ivars

- 一个objc对象的isa的指针指向什么？有什么作用？
- Associated Object实现原理
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
{ "object": { "key": "value" } }

有一个Association Manager会管理所有的associated object，也是hashmap的形式，key是被关联的对象地址取反，value就是关联上去的对象
当设置关联对象时，先查看是否关联过对象，如果关联过，需要把原先关联的对象释放掉
关联的过程就是设置key和value的过程
关联完成后，更新被关联对象的isa，里面有个字段就是代表是否有关联对象
如果标志位1，代表有关联对象，那么在对象引用计数降为0的时候需要对关联对象进行release操作

- 使用runtime Associate方法关联的对象，需要在主对象dealloc的时候释放么？
1. 调用 -release ：引用计数变为零
     * 对象正在被销毁，生命周期即将结束.
     * 不能再有新的 __weak 弱引用， 否则将指向 nil.
     * 调用 [self dealloc]
2. 子类 调用 -dealloc
     * 继承关系中最底层的子类 在调用 -dealloc
     * 如果是 MRC 代码 则会手动释放实例变量们（iVars）
     * 继承关系中每一层的父类 都在调用 -dealloc
3. NSObject 调 -dealloc
     * 如果不是nonpointer，不存在弱引用，不存在sidetable，不存在c++析构方法，不存在关联对象，就直接释放
     * 如果不满足上面的条件，会调用 Objective-C runtime 中的 object_dispose() 方法
4. 调用 object_dispose()
     * 为 C++ 的实例变量们（iVars）调用 destructors
     * 为 ARC 状态下的 实例变量们（iVars） 调用 -release
     * 对所有使用 runtime Associate方法关联到此对象上的对象调用release方法
     * 解除所有 __weak 引用
     * 调用 free()

- _objc_msgForward 函数是做什么的，直接调用它将会发生什么？
消息转发，是调用方法的最后一层

- runtime如何实现weak变量的自动置nil？
当某个对象的retaincount变为0之后，已该对象的地址为key，查找存储weak指针的字典，找到所有的weak指针数组，然后依次置为nil

- 能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？
不可以，编译完成后，类的结构就已经确定下来了

- runloop和线程有什么关系？
- runloop的mode作用是什么？
- 以+ scheduledTimerWithTimeInterval...的方式触发的timer，在滑动页面上的列表时，timer会暂定回调，为什么？如何解决？
- 猜想runloop内部是如何实现的？
- objc使用什么机制管理对象内存？
- ARC通过什么方式帮助开发者管理内存？
- 不手动指定autoreleasepool的前提下，一个autorealese对象在什么时刻释放？（比如在一个vc的viewDidLoad中创建）
- BAD_ACCESS在什么情况下出现？
- 苹果是如何实现autoreleasepool的？
- 使用block时什么情况会发生引用循环，如何解决？
- 在block内如何修改block外部变量？
- 使用系统的某些block api（如UIView的block版本写动画时），是否也考虑引用循环问题？
- GCD的队列（dispatch_queue_t）分哪两种类型？
- 如何用GCD同步若干个异步调用？（如根据若干个url异步加载多张图片，然后在都下载完成后合成一张整图）
- dispatch_barrier_async的作用是什么？
- 苹果为什么要废弃dispatch_get_current_queue？
- addObserver:forKeyPath:options:context:各个参数的作用分别是什么，observer中需要实现哪个方法才能获得KVO回调？
- 如何手动触发一个value的KVO
- 若一个类有实例变量 NSString *_foo ，调用setValue:forKey:时，可以以foo还是 _foo 作为key？
- KVC的keyPath中的集合运算符如何使用？
- KVC和KVO的keyPath一定是属性么？
- 如何关闭默认的KVO的默认实现，并进入自定义的KVO实现？
- apple用什么方式实现对一个对象的KVO？
- IBOutlet连出来的视图属性为什么可以被设置成weak?
- IB中User Defined Runtime Attributes如何使用？
- 如何调试BAD_ACCESS错误
- lldb（gdb）常用的调试命令？

数据持久化存储方案有哪些？
沙盒的目录结构是怎样的？各自一般用于什么场合？
SQL语句问题：inner join、left join、right join的区别是什么？
sqlite的优化网络通信用过哪些方式（100%的人说了AFNetworking...）
如何处理多个网络请求并发的情况
在网络请求中如何提高性能
在网络请求中如何保证安全性

- 内存中的栈和堆的区别是什么？那些数据在栈上，哪些在堆上？
一个由c/c++编译的程序占用的内存分为以下几个部分
1、栈区（stack）— 由编译器自动分配释放 ，存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈。
2、堆区（heap） — 一般由程序员分配释放， 若程序员不释放，程序结束时可能由OS回收 。注意它与数据结构中的堆是两回事，分配方式类似于链表。
3、全局区（静态区）（static）—，全局变量和静态变量的存储是放在一块的，初始化的全局变量和静态变量在一块区域，未初始化的全局变量和未初始化的静态变量在相邻的另一块区域。 - 程序结束后有系统释放
4、文字常量区—常量字符串就是放在这里的。 程序结束后由系统释放
5、程序代码区—存放函数体的二进制代码。

- #define和const定义的变量，有什么区别


- 什么情况下会出现内存的循环引用
- block中的weak self，是任何时候都需要加的么？
- GCD的queue，main queue中执行的代码，一定是在main thread么？
是的

使用dispatch_sync执行的代码都会在本线程中执行
在主线程中也可以有不同的队列来执行

NSOperationQueue有哪些使用方式
NSThread中的Runloop的作用，如何使用？
.h文件中的变量，外部可以直接访问么？（注意是变量，不是property）
讲述一下runtime的概念，message send如果寻找不到相应的对象，会如何进行后续处理 ？
TCP和UDP的区别是什么？
MD5和Base64的区别是什么，各自场景是什么？
二叉搜索树的概念，时间复杂度多少？

哪些类不适合使用单例模式？即使他们在周期中只会出现一次。
Notification的使用场景是什么？同步还是异步？
简单介绍一下KVC和KVO，他们都可以应用在哪些场景？

如何添加一个自定义字体到工程中
如何制作一个静态库/动态库，他们的区别是什么？
Configuration中，debug和release的区别是什么？
简单介绍下发送系统消息的机制（APNS）

系统如何寻找到需要响应用户操作的那个Responder
多屏幕尺寸的适配
UIButton的父类是什么？UILabel呢？
push view controller 和 present view controller的区别
描述下tableview cell的重用机制
UIView的frame和bounds的区别是什么

发送10个网络请求，然后再接收到所有回应之后执行后续操作，如何实现？
实现一个第三方控件，可以在任何时候出现在APP界面最上层
实现一个最简单的点击拖拽功能。上面那个拖拽之外，如果在手放开时，需要根据速度往前滑动呢？
如何减小一个应用程序的尺寸？
如何提高一个应用程序的性能？
不同版本的APP，数据库结构变化了，如何处理?

