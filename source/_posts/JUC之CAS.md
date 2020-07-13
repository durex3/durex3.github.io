---
title: JUC之CAS
date: 2020-07-13 21:13:01
categories:
- 多线程
tags:
- Java
- 多线程
---

JUC有两大核心：CAS和AQS，其中CAS是java.util.concurrent.atomic包的基础。

<!-- more -->

## 什么是CAS ##

CAS的全称为Compare-And-Swap，是一种思想和算法，同时是一条CPU指令。CAS有三个操作数：内存值V、预期值A、要修改的值B，当且仅当预期值和内存值相同时，才将内存值修改为B，否则什么都不做。

## CAS源码分析 ##

CAS是乐观锁，是一种冲突重试的机制，在并发竞争不是很激烈的情况下，他的性能要好于基于锁的并发性能。因为并发竞争激烈的话，冲突重试的过程很多。这里以AtomicInteger为例分析源码。

AtomicInteger会加载Unsafe工具，用来直接操作内存数据。Unsafe是CAS的核心类。Java无法直接访问底层操作系统，而是通过本地(native)方法来访问。不过尽管如此，JVM还是开了一个后门，JDK中有一个类Unsafe，它提供了硬件级别的原子操作。valueOffset表示的是变量值在内存中的偏移地址，因为Unsafe就是根据内存偏移地址获取数据的原值的，这样我们就能通过unsafe来实现CAS了。value是用volatile修饰的，保证了多线程之间看到的value值是同一份。源码如下：

```java

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

AtomicInteger的核心方法就是getAndAddInt(Object var1, long var2, int var4)。通过调用unsafe的getIntVolatile(var1, var2)，这是个native方法，其实就是获取var1中，var2偏移量处的值。var1就是AtomicInteger，var2就是我们前面提到的valueOffset，这样我们就从内存里获取到现在valueOffset处的值了。compareAndSwapInt(var1, var2, var5, var5 + var4)其实换成compareAndSwapInt(obj, offset, expect, update)比较清楚，意思就是如果obj内的value和expect相等，就证明没有其他线程改变过这个变量，那么就更新它为update，如果这一步的CAS没有成功，那就采用自旋的方式继续进行CAS操作。源码如下：

```java

    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

## CAS的缺点 ##

CAS也有缺点，就是ABA问题。什么是ABA问题？有如下操作：

- 线程一：获取出数据的初始值是A，计划实施CAS，期望数据仍是A的时候，修改才能成功
- 线程二：将数据修改成B
- 线程三：将数据修改回A

此时线程一进入发现数据仍然为期望数据，但是这个时候的A已经不是之前的A了，CAS因为是比较的是数值，所以可能存在安全隐患。

