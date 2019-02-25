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

- 代理是什么？协议/接口 模式应用场景？
- 实现响应链传递的两个方法？（hitTest和pointInside）在哪些地方使用过？
最常用的应该是用来扩大按钮的点击区域

- alloc&init是什么
- 编译器的优化，分析被编译器优化的部分
- OC对象内存中的分配，OC对象所占用的空间
- layer和view
- autolayout


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

- block可以修改外部变量吗？(为什么直接就可以修改全局变量？使用__block才可以修改局部变量？)
- block底层实际上是什么？block有几种类型？block使用时的注意事项
都是c的struct，stack,malloc,global

- OC对象本质是什么？
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
