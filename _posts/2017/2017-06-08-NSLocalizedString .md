---
layout: post
title: NSLocalizedString的一个小知识点
category: OC
tags: [OC]
---

今天在使用NSLocalizedString的时候碰到一个小问题

如下：
```
中文版：我是来自“xxx”的xxx
英文版：I am xxx from 'xxx'
```
这里有两个问题：
1. 中文版被两个参数分成了4部分，英文版被两个参数分成了5部分
2. 参数的顺序不同

第1个很好办，只要在NSLocalizedString中使用%@这样的参数就可以了，这样在使用的时候按照这样的格式就可以了：
```
Localizable.strings中有如下定义：
"FORMAT" = "我是来自“%@”的%@";

str = [NSString stringWithFormat:NSLocalizedString(@"FORMAT", nil), xxx, xxx];
```
第2个不太常见，原来参数是可以指定顺序的：
```
Localizable.strings中有如下定义：
"FORMAT" = "我是来自“%1$@”的%2$@";
"FORMAT" = "I am %2$@ from '%1$@'";

str = [NSString stringWithFormat:NSLocalizedString(@"FORMAT", nil), xxx, xxx];
```
只需要在%和@中间加上1$,2$等等就可以啦，数字就代表参数的顺序。