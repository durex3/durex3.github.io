---
title: JUC之AQS
date: 2020-07-20 13:13:44
categories:
- 多线程
tags:
- Java
- 多线程
---

AQS(AbstractQueuedSynchronizer)是JDK并发工具包下的一个模板类，我们经常使用的ReentrantLock，CountDownLatch，CyclicBarrier等都是基于它实现的。

<!-- more -->

## 为什么需要AQS ##

ReentrantLock，CountDownLatch，CyclicBarrier等类都有类似的“协作”(或者叫“同步”)功能，其实，它们底层都用了一个共同的基类，这就是AQS。

因为上面的协作类，他们很多工作都是类似的，所有如果能提取出一个工具类，那么就可以直接用，对于协作类而言就可以屏蔽很多细节，只关注它们自己的“业务逻辑”就可以了。
