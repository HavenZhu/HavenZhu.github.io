## 基础
- nonatomic和atomic，atomic一定安全吗
atomic底层是通过锁机制来实现的，保证可以得到一个完整的结果
nonatomic则可能导致得到一个部分完整的结果
atomic仅仅保证了本属性getter/setter方法的线程安全，当调用属性的其他方法，或者对象多个atomic属性之间，是没法保证线程安全的。

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
- block可以修改外部变量吗？(为什么直接就可以修改全局变量？使用__block才可以修改局部变量？)
- block分几种类型？内存区域在什么位置？
- 实现响应链传递的两个方法？（hitTest和pointInside）在哪些地方使用过？


## 中级
- KVO的底层实现原理，能否手动实现一个KVO，怎么处理？
- block底层实际上是什么？block有几种类型？block使用时的注意事项
- OC对象本质是什么？
- Runtime是什么？用到过runtime吗？
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


