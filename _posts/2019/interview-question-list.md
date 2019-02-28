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


- autolayout

- ARC时代的内存泄漏
如何检测
非oc对象内存泄漏
常用框架内存泄漏


## 中级
- KVO的底层实现原理，能否手动实现一个KVO，怎么处理？
KVO例子：视频通话过程中，监听window的屏幕常亮属性，防止由于其他操作关闭了屏幕常亮

- 响应链可以用来做一些delegate做的事，比如将cell中按钮的点击事件传给controller

- 可以用什么方法保证线程安全
队列，

- 锁:
- OSSpinLock: 存在优先级反转的问题（低优先级线程拿到锁，高优先级线程忙等，浪费cpu时间，低优先级线程获取不到cpu时间无法处理完）
- dispatch_semaphore：就是信号量，0开始执行，-1就等待。会进行上下文切换
- pthread_mutex：互斥锁
- NSLock：内部封装了pthread_mutex，因为多了一层方法调用，所以性能比pthread_mutex差一点
- NSCondition：用这玩意儿不如用dispatch_semaphore
- pthread_mutex(recursive)：递归锁，一般情况下，除了递归调用的方法里想要加锁，其他情况很少使用
- NSRecursiveLock：同上
- NSConditionLock
- @synchronize：用传入的对象锁，锁存在map中，key是经过计算的对象的内存地址（花点时间看一下源码，深入理解一下）

- 自己实现锁？自旋锁？互斥锁

- block可以修改外部变量吗？(为什么直接就可以修改全局变量？使用__block才可以修改局部变量？)
- block底层实际上是什么？block有几种类型？block使用时的注意事项
都是c的struct，stack, malloc, global

- OC对象本质是什么？
OC所有的对象，类在runtime层都是c struct，都是objc_object

- Runtime是什么？用到过runtime吗？
protocol, property, method, object, class

- SDWebImage底层实现？还有它的缓存机制?
- AFNetWorking在成功回调后是处于什么线程之中？为什么会这样？


## 进阶
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
- 这个写法会出什么问题： @property (copy) NSMutableArray *array;
- 如何让自己的类用 copy 修饰符？如何重写带 copy 关键字的 setter？
- @property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的
- @protocol 和 category 中如何使用 @property
- runtime 如何实现 weak 属性

- @property中有哪些属性关键字？/ @property 后面可以有哪些修饰符？

- weak属性需要在dealloc中置nil么？

- @synthesize和@dynamic分别有什么作用？

- ARC下，不显式指定任何属性关键字时，默认的关键字都有哪些？

- 用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题？

- 对非集合类对象的copy操作

- 集合类对象的copy与mutableCopy

- @synthesize合成实例变量的规则是什么？假如property名为foo，存在一个名为_foo的实例变量，那么还会自动合成新变量么？

- 在有了自动合成属性实例变量之后，@synthesize还有哪些使用场景？

- objc中向一个nil对象发送消息将会发生什么？

- objc中向一个对象发送消息[obj foo]和objc_msgSend()函数之间有什么关系？

- 什么时候会报unrecognized selector的异常？

- 一个objc对象如何进行内存布局？（考虑有父类的情况）

- 一个objc对象的isa的指针指向什么？有什么作用？

- _objc_msgForward 函数是做什么的，直接调用它将会发生什么？
- runtime如何实现weak变量的自动置nil？
- 能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？
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
