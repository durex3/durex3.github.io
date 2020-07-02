---
title: JUC之ThreadLocal
date: 2020-06-30 22:38:43
categories:
- 多线程
tags:
- JUC
- 多线程
---
ThreadLocal是除了加锁这种同步方式之外的一种保证一种规避多线程访问出现线程不安全的方法，当我们在创建一个变量后，如果每个线程对其进行访问的时候访问的都是线程自己的变量这样就不会存在线程不安全问题。

<!-- more -->

## ThreadLocal是什么

ThreadLocal叫做线程变量，意思是ThreadLocal中填充的变量属于当前线程，该变量对其他线程而言是隔离的。ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

## ThreadLocal的用法

典型场景一：每个线程需要一个独享的对象(通常是工具类，例如SimpleDateFormat和Random等)。当多个线程去调用SimpleDateFormat类的format()方法的时候，由于format()方法不是线程安全的，所以就会引发线程不安全的问题。代码如下:

```java

	import java.text.SimpleDateFormat;
	import java.util.Date;
	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	
	/**
	 * @author gelong
	 * @date 2020/6/30 23:22
	 */
	public class ThreadLocalNormalUsage00 {
	
	    private static ExecutorService service = Executors.newFixedThreadPool(10);
	    private static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
	
	    public static void main(String[] args) {
	        for (int i = 0; i < 100; i++) {
	            int finalI = i;
	            service.execute(() -> {
	                String date = new ThreadLocalNormalUsage00().date(finalI);
	                System.out.println(date);
	            });
	        }
	        service.shutdown();
	    }
	
	    public String date(int seconds) {
	        Date date = new Date(seconds);
	        return simpleDateFormat.format(1000 * seconds);
	    }
	}
```

![format.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggar3809c1j20o40d70tm.jpg)

解决办法有多种，我们既可以把SimpleDateFormat类声明为date()方法的局部变量，也可以对date()方法进行加锁。不过这两种方法一个复用性太差(每一个线程都要new一个对象)，另一个效率太多(高并发下需要排队)，更好的解决方案是使用ThreadLocal，给每个线程分配自己的SimpleDateFormat对象。代码如下：

```java

	import java.text.SimpleDateFormat;
	import java.util.Date;
	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	
	/**
	 * @author gelong
	 * @date 2020/6/30 23:22
	 */
	public class ThreadLocalNormalUsage01 {
	
	    private static ExecutorService service = Executors.newFixedThreadPool(10);
	
	    public static void main(String[] args) {
	        for (int i = 0; i < 100; i++) {
	            int finalI = i;
	            service.execute(() -> {
	                String date = new ThreadLocalNormalUsage01().date(finalI);
	                System.out.println(date);
	            });
	        }
	        service.shutdown();
	    }
	
	    public String date(int seconds) {
	        Date date = new Date(seconds);
	        return ThreadSafeFormatter.threadLocal.get().format(1000 * seconds);
	    }
	}
	
	class ThreadSafeFormatter {
	    public static ThreadLocal<SimpleDateFormat> threadLocal = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd hh:mm:ss"));
	}
```

典型场景二：每个线程内需要保持全局变量(例如在拦截器中获取用户信息，可以让不同的方法直接调用，避免参数传递的麻烦)。一个比较繁琐的解决方案是把user作为参数层层传递，但是这样会导致代码冗余且不易维护。如图：

![user.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggbsucnfm6j20w40c5wg1.jpg)

还有一种方法是把user对象存进线程安全的map，但这样会对性能有所损耗。更好的解决方案是使用ThreadLocal。代码如下：

```java

	/**
	 * @author gelong
	 * @date 2020/6/30 23:22
	 */
	public class ThreadLocalNormalUsage02 {
	    public static void main(String[] args) {
	        new Service1().process();
	    }
	}
	
	class User {
	    String name;
	
	    public User(String name) {
	        this.name = name;
	    }
	}
	
	class Service1 {
	    public void process() {
	        User user = new User("test");
	        UserContextHolder.holder.set(user);
	        System.out.println("service1：" + user.name);
	        new Service2().process();
	    }
	}
	
	class Service2 {
	    public void process() {
	        User user = UserContextHolder.holder.get();
	        System.out.println("service2：" + user.name);
	        new Service3().process();
	    }
	}
	
	class Service3 {
	    public void process() {
	        User user = UserContextHolder.holder.get();
	        System.out.println("service3：" + user.name);
	    }
	}
	
	class UserContextHolder {
	    public static final ThreadLocal<User> holder = new ThreadLocal<>();
	}
```

![userserivce.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggbu8kbkolj20gr07qmxe.jpg)

## ThreadLocal的原理

每个Thread对象中持有一个ThreadLocal.ThreadLocalMap的成员变量，key是ThreadLocal，value是保存的对象，所以存储的对象是保存在Thread中的。如图：

![threadlocal.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggbwg6czvrj20o20i8q7u.jpg)

get()方法先取出当前线程的ThreadLocalMap，然后调用map.getEntry(this)方法，把当前的ThreadLocal对象作为key，取出map中的value。源码如下：

```java

	public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

set()方法先取出当前线程的ThreadLocalMap，然后直接调用map.set(this, value)方法，把当前的ThreadLocal对象作为key，存储传入的value值。源码如下：

```java

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

initialValue()方法只有当第一次调用get()方法且ThreadLocalMap为null时才会调用。源码如下：

```java

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

## ThreadLocal的注意点

在使用ThreadLocal的过程中可能会导致内存泄露，某个对象不再有用，但是占用的内存缺不能被回收。ThreadLocalMap中的每个Entry都是对key的弱引用，每个Entry都包含了一个对value的强引用。源码如下：

```java

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
```

正常情况下，当线程终止，保存在ThreadLocalMap里的value会被垃圾回收，但是，如果线程不终止(比如线程需要保持很久)，那么key对应的value就不能被回收，因为有以下调用链：

	Thread-ThreadLocalMap-Entry(key为null)-value

因为value和Thread之间还存在着强引用链路，所以导致value无法被回收，就可能出现OOM。所以在使用完ThreadLocal之后，就应该调用remove()方法，删除对应的Entry对象。



