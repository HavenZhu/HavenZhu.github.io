---
layout: post
title: Runtime源码 —— 概述和调试环境准备
category: Runtime
tags: [Runtime]
---

关于runtime的文章已经有很多很多了，有些也写的很详细，但纸上得来终觉浅，我还是决定自己动手学一学，并且记录下这个过程。

这一系列准备写的内容，更多的是写给我自己看。

第一篇文章，决定写写OC的运行时，也就是runtime。
这东西正常几乎接触不到，最常用的应该算是在category中添加property时，使用的AssociatedObject。苹果自己在[runtime官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)中也说了：
> Typically, though, there should be little reason for you to need to know and understand this material to write a Cocoa application.
> 
> 翻译：会不会这玩意儿无关紧要。

但毕竟runtime也算是整个OC的基础，理解runtime对理解这门语言有很大的帮助，我也想以此为切入点，提高一下阅读源码的能力。

runtime的内容很多，一篇肯定写不完，本文先就目前我对runtime的理解做一点简单的介绍，后面随着源码的阅读会分成多篇来详细分析。

## runtime概述

OC是一门动态语言，OC尽可能多的将工作留到了运行时，而不是在编译期和链接期，这赋予了OC很多强大的功能，这也是runtime所要做的工作。

runtime可以看做是一个库，主要由c和汇编来实现，我们在OC里面所写的类，方法，属性等等在runtime层面都会转化成c的结构体。
官方文档中说通常我们会通过三个方面与runtime打交道：
>1. OC源代码
>2. NSObject提供的相关方法
>3. runtime库提供的各种api

第1个方面就不说了，第2和第3部分是这个系列的重点。我会分析正常写OC代码接触较多的，比如类（class），方法（method），属性（property），协议（protocol），分类（category），自动释放池（autoreleasepool），引用计数（retain，release等），初始化（load，alloc，init，new等）等这些内容的源代码。

希望通过对runtime的研究能够加深对OC的理解，戒掉浅尝辄止的坏毛病，走上造轮子的道路。

## 调试环境准备

想要学习runtime，必须有一个可以运行的环境才行，苹果官方的源代码没法编译，这里有可以运行的[runtime source code](https://github.com/RetVal/objc-runtime)。在最新的Mac OS 10.12.2 + Xcode 8.2.1环境下有效。