
- block是什么
- block的原理
- block拦截对象的本质
- block会持有哪些对象
- __block的原理
- 为什么要在block中使用weak_self


写过oc代码的程序员应该都不会对block感到陌生，block的语法虽然有点怪异，但使用起来非常方便。这篇就来好好探讨一下。

## block是什么

> 匿名函数指针

## block的实现原理

当我们写下一个block时，究竟发生了什么？