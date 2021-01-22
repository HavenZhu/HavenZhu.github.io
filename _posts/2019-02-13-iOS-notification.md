---
layout: post
title: iOS通知
category: OC
tags: [OC]
---

iOS提供的通知有两种形式，远程通知和本地通知。这篇来记录一下对通知的理解。

## 远程通知

远程通知(remote notification)，就是我们常听到的APNs提供的服务。APNs(Apple Push Notification service)是苹果官方提供的健壮，安全，高效的消息推送服务。要理解远程通知，本质上就是理解APNs的工作原理。

### APNs的背景

当我们想发送一条消息时，最直接的想法是服务器直接推送消息给app客户端，但这就有一个问题，当app没有运行的时候，消息就没法送达了。为了解决这个问题，苹果提供了APNs服务，当app没有运行时，也能将消息送达到客户端。其实apple自己也在通过APNs给我们提供服务，比如系统更新等等。

### APNs的原理

远程通知由2个部分组成：
> provider --> APNs --> device

provider就是我们的服务器，APNs相当于我们自己的服务器与设备之间的一个中转站。对于客户端开发而言，我们一般会在开发者账号中创建一个推送证书，再与App ID绑定，最后把这个证书发给服务器端就完事了。

那么在推送通知之前，这几部分之间的连接是如何建立的呢？

#### provider --> APNs

<p align="center">
    <img src="http://betterzn.com/assets/images/2019_02_13/provider_APNs.png" />
</p>

上图是provider和APNs之间建立TLS连接的过程：
- provider发出建立TLS的请求给APNs
- APNs发送APNs证书给provider
- provider验证APNs证书，再将开发者中心创建的推送证书发送给APNs
- APNs验证推送证书
- 验证通过，成功建立TLS

服务器只需要将通知按照一定的格式发送给APNs，剩下的事情就由APNs负责了。

#### APNs --> device

APNs与设备之间的连接是在设备初始化的时候自动建立的，不需要app的参与，也就是不需要我们做任何事情。

在设备初始化时，设备操作系统会生成证书和秘钥并存储到钥匙串中，用于与APNs之间的验证。

<p align="center">
    <img src="http://betterzn.com/assets/images/2019_02_13/APNs_device.png" />
</p>

上图是APNs与设备建立连接的过程：
- device发起建立TLS的请求给APNs
- APNs发送APNs证书给device
- device验证APNs证书，再把操作系统生成的设备证书发送给APNs
- APNs验证设备证书
- 验证通过，成功建立TLS

这应该就是为什么激活iPhone时需要联网的原因，有这个与APNs建立连接的过程。

总的来看，provider，APNs，device之间都是通过TCP长连接来通信的。

#### 发送远程通知的流程

连接建立好了，下面就是发送通知了。

一次完整的发送通知的过程如下：
- app向APNs注册接受远程通知
- 成功之后app会收到带有device token的回调，每一个device token实际上是设备+app的组合标识
- app将device token发送给provider
- provider将device token和通知内容发送给APNs
- APNs解析device token，将通知发送给对应的设备
- 设备收到通知后，由操作系统将通知转发给app

这个过程还是比较清晰的，都是通过device token进行验证。我们拿到的device token是APNs加密后的结果，秘钥只有APNs持有，当provider发通知给APNs时，APNs解密device token获取对应的设备与app信息。

## 本地通知

远程通知是由APNs发送给设备的，本地通知就是app发送给设备的。设备有个通知中心，远程和本地通知都是发送到那里去。

比如我们在iPhone自带的时钟app里定了个闹钟，实际上app已经向通知中心注册了此通知，时间到了之后，通知中心就会发出这个通知。

没有了与APNs的交互，这个过程还是很简单的。

## 实践

在微信或者QQ中，这两种通知实际上是结合起来使用的，如果app处于运行状态，无论是前台还是后台，app与服务器之间维持着TCP长连接，服务器直接把消息发送给app，app收到之后弹出本地通知。如果app没有运行，则通过APNs推送远程通知。

