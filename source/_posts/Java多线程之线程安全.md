---
title: Java多线程之线程安全
date: 2020-06-19 23:51:44
categories:
- 多线程
tags:
- Java
- 多线程
---
当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象是线程安全的。

<!-- more -->

通俗地说就是，不管业务中遇到怎样的多个线程访问某对象或某方法的情况，而在编程这个业务逻辑的时候，都不需要额外做任何额外的处理(也就是可以像单线程编程一样)，程序也可以正常运行(不会因为多线程而出错)，就可以称为线程安全。相反，如果在编程的时候，需要考虑这些线程在运行时的调度和交替(例如在get()调用到期间不能调用set())，或者需要进行额外的同步(比如使用synchronized关键字等)，那么就是线程不安全的。下面我们来看看几个常见的线程不安全的实例。

## 运行结果错误

这种情况就是我们常见的a++导致的线程不安全的问题，由于a++并不是原子性的，所以当多个线程同时进行a++时就会导致运行结果不正确。代码如下：

```java

	/**
	 * 第一种：运行结果出错
	 * 计数不准确（减少），找出具体出错的位置
	 * @author gelong
	 * @date 2019/12/23 23:06
	 */
	public class MultiThreadsError implements Runnable {
	
	    private static int index = 0;
	    private static MultiThreadsError instance = new MultiThreadsError();
	
	    @Override
	    public void run() {
	        for (int i = 0; i < 10000; i++) {
	            index++;
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread thread1 = new Thread(instance);
	        Thread thread2 = new Thread(instance);
	        thread1.start();
	        thread2.start();
	        thread1.join();
	        thread2.join();
	        System.out.println("表面上的结果是: " + index);
	    }
	}
```

我们可以把上面的的程序升级一下，看看a++具体消失在哪里。CyclicBarrier强行让两个线程都执行完a++操作后才继续往下执行。AtomicInteger是原子类是线程安全的，用来记录a++的正确结果和错误次数。marked来记录某个值的自增操作是否已经执行了。

```java

	import java.util.concurrent.BrokenBarrierException;
	import java.util.concurrent.CyclicBarrier;
	import java.util.concurrent.atomic.AtomicInteger;
	
	/**
	 * 第一种：运行结果出错
	 * 计数不准确（减少），找出具体出错的位置
	 * @author gelong
	 * @date 2019/12/23 23:06
	 */
	public class MultiThreadsError1 implements Runnable {
	
	    private static int index = 0;
	    private static MultiThreadsError1 instance = new MultiThreadsError1();
	    private static AtomicInteger realIndex = new AtomicInteger();
	    private static AtomicInteger wrongCount = new AtomicInteger();
	    private final boolean[] marked = new boolean[100000];
	    private static CyclicBarrier cyclicBarrier1 = new CyclicBarrier(2);
	    private static CyclicBarrier cyclicBarrier2 = new CyclicBarrier(2);
	
	    @Override
	    public void run() {
	        marked[0] = true;
	        for (int i = 0; i < 10000; i++) {
	            try {
	                cyclicBarrier1.await();
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            } catch (BrokenBarrierException e) {
	                e.printStackTrace();
	            }
	            index++;
	            try {
	                cyclicBarrier2.await();
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            } catch (BrokenBarrierException e) {
	                e.printStackTrace();
	            }
	            realIndex.incrementAndGet();
	            synchronized (instance) {
	                if (marked[index] && marked[index - 1]) {
	                    wrongCount.incrementAndGet();
	                    System.out.println("发生错误: " + index);
	                }
	                marked[index] = true;
	            }
	        }
	
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread thread1 = new Thread(instance);
	        Thread thread2 = new Thread(instance);
	        thread1.start();
	        thread2.start();
	        thread1.join();
	        thread2.join();
	        System.out.println("表面上的结果是: " + index);
	        System.out.println("真正的结果是: " + realIndex);
	        System.out.println("错误次数: " + wrongCount);
	    }
	}
```

## 死锁

```java

	/**
	 * 死锁
	 * @author gelong
	 * @date 2019/12/23 23:06
	 */
	public class MultiThreadsError2 implements Runnable {
	
	    private int flag = 0;
	    private static final Object o1 = new Object();
	    private static final Object o2 = new Object();
	
	    @Override
	    public void run() {
	        System.out.println("flag = " + flag);
	        if (flag == 0) {
	            synchronized (o1) {
	                try {
	                    Thread.sleep(500);
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                }
	                synchronized (o2) {
	                    System.out.println(0);
	                }
	            }
	        }
	        if (flag == 1) {
	            synchronized (o2) {
	                try {
	                    Thread.sleep(500);
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                }
	                synchronized (o1) {
	                    System.out.println(1);
	                }
	            }
	        }
	    }
	
	    public static void main(String[] args) {
	        MultiThreadsError2 r1 = new MultiThreadsError2();
	        MultiThreadsError2 r2 = new MultiThreadsError2();
	        r1.flag = 0;
	        r2.flag = 1;
	        new Thread(r1).start();
	        new Thread(r2).start();
	    }
	}
```
## 总结
访问共享的变量或资源，会有并发风险，比如对象的属性、静态变量、共享缓存、数据库等。所有依赖时序的操作，即使每一步操作都是线程安全的，还是存在并发问题。我们使用其他类的时候，如果对方没有声明自己是线程安全的，那么大概率会存在并发问题。