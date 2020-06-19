---
title: Java多线程之wait、notify/notifyAll
date: 2020-06-11 22:22:13
categories:
- 多线程
tags:
- Java
- 多线程
---
wait()、notify/notifyAll()方法是Object的final方法，无法被重写。

<!-- more -->

## 1. wait()、notify()/notifyAll()的作用和用法

wait()/wait(long timeout)使当前线程进入waiting、timed_waiting状态，前提是必须先获得锁，所以wait()方法的使用必须配合synchronized关键字。直到以下四种情况之一发生时，才会被唤醒：

- 另一个线程调用这个对象的notify()方法且刚好被唤醒的对象是本线程
- 另一个线程调用这个对象的notifyAll()方法
- 过了wait(long timeout)方法规定的时间，如果传入0就是永久等待
- 线程自身调用了interrupt()方法被中断

notify()方法只会唤醒等待队列其中一个线程，唤醒哪个线程取决于操作系统对多线程管理的实现。代码如下：

```java
	
	/**
	 * @author gelong
	 * @date 2020/6/11 23:10
	 */
	public class Wait {
	
	    private static final Object object = new Object();
	
	    static class ThreadA extends Thread {
	        @Override
	        public void run() {
	            synchronized (object) {
	                System.out.println(Thread.currentThread().getName() + "获取了锁");
	                System.out.println(Thread.currentThread().getName() + "准备等待");
	                try {
	                    object.wait();
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                }
	                System.out.println(Thread.currentThread().getName() + "被唤醒了");
	            }
	        }
	    }
	
	    static class ThreadB extends Thread {
	        @Override
	        public void run() {
	            synchronized (object) {
	                System.out.println(Thread.currentThread().getName() + "获取了锁");
	                System.out.println(Thread.currentThread().getName() + "唤醒其他线程");
	                object.notify();
	            }
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        ThreadA threadA = new ThreadA();
	        threadA.start();
	        Thread.sleep(100);
	        ThreadB threadB = new ThreadB();
	        threadB.start();
	    }
	}
```

![notify.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfs73a74lgj20j2081mxa.jpg)

notifyAll()会唤醒所有等待的线程。代码如下：

```java
	
	/**
	 * @author gelong
	 * @date 2020/6/14 22:47
	 */
	public class WaitNotifyAll implements Runnable {
	
	    private static final Object object = new Object();
	
	    @Override
	    public void run() {
	        synchronized (object) {
	            System.out.println(Thread.currentThread().getName() + "获取了锁");
	            try {
	                System.out.println(Thread.currentThread().getName() + "准备进入等待");
	                object.wait();
	                System.out.println(Thread.currentThread().getName() + "即将运行结束");
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Runnable runnable = new WaitNotifyAll();
	        Thread threadA = new Thread(runnable);
	        Thread threadB = new Thread(runnable);
	        Thread threadC = new Thread(() -> {
	            synchronized (object) {
	                System.out.println(Thread.currentThread().getName() + "唤醒所有线程");
	                object.notifyAll();
	            }
	        });
	        threadA.start();
	        threadB.start();
	        Thread.sleep(100);
	        threadC.start();
	    }
	}
```

![notifyAll.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfs8jdifmwj20ip08s0t0.jpg)

当我们把线程c的object.notifyAll()改成object.notify会导致有一个线程无法被唤醒，导致程序永远无法停止。代码如下：

```java

	/**
	 * @author gelong
	 * @date 2020/6/14 22:47
	 */
	public class WaitNotifyAll implements Runnable {
	
	    private static final Object object = new Object();
	
	    @Override
	    public void run() {
	        synchronized (object) {
	            System.out.println(Thread.currentThread().getName() + "获取了锁");
	            try {
	                System.out.println(Thread.currentThread().getName() + "准备进入等待");
	                object.wait();
	                System.out.println(Thread.currentThread().getName() + "即将运行结束");
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Runnable runnable = new WaitNotifyAll();
	        Thread threadA = new Thread(runnable);
	        Thread threadB = new Thread(runnable);
	        Thread threadC = new Thread(() -> {
	            synchronized (object) {
	                System.out.println(Thread.currentThread().getName() + "唤醒所有线程");
	                // object.notifyAll();
	                object.notify();
	            }
	        });
	        threadA.start();
	        threadB.start();
	        Thread.sleep(100);
	        threadC.start();
	    }
	}
```

![notifyAll2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfs8mp9dulj20il07g0sz.jpg)

如果线程持有多把锁wait方法只会释放当前对象的monitor锁。代码如下：

```java

	/**
	 * @author gelong
	 * @date 2020/6/15 21:25
	 */
	public class WaitNotifyReleaseOwnMonitor {
	
	    private static final Object resourceA = new Object();
	    private static final Object resourceB = new Object();
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread threadA = new Thread(() -> {
	            synchronized (resourceA) {
	                System.out.println(Thread.currentThread().getName() + "获取锁A");
	                synchronized (resourceB) {
	                    System.out.println(Thread.currentThread().getName() + "获取锁B");
	                    try {
	                        System.out.println(Thread.currentThread().getName() + "准备释放锁A");
	                        resourceA.wait();
	                    } catch (InterruptedException e) {
	                        e.printStackTrace();
	                    }
	                }
	            }
	        });
	        Thread threadB = new Thread(() -> {
	            synchronized (resourceA) {
	                System.out.println(Thread.currentThread().getName() + "获取锁A");
	                System.out.println(Thread.currentThread().getName() + "尝试获取锁B");
	                synchronized (resourceB) {
	                    System.out.println(Thread.currentThread().getName() + "获取锁B");
	                }
	            }
	        });
	        threadA.start();
	        Thread.sleep(100);
	        threadB.start();
	    }
	}
```

![wait.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gftb37oek3j20iw06rglt.jpg)

由于线程0没有释放锁B所以线程1会无穷地等待锁，所以线程如果持有多把锁要主意锁的释放顺序。


## 2. wait()、notify的应用

生产者消费者模式

代码如下：

```java

	import java.util.Date;
	import java.util.LinkedList;
	
	/**
	 * @author gelong
	 * @date 2020/6/15 22:21
	 */
	public class ProducerConsumerModel {
	    public static void main(String[] args) {
	        EventStorage storage = new EventStorage(10);
	        Producer producerA = new Producer(storage);
	        Producer producerB = new Producer(storage);
	        Consumer consumerA = new Consumer(storage);
	        Consumer consumerB = new Consumer(storage);
	        new Thread(producerA).start();
	        new Thread(producerB).start();
	        new Thread(consumerA).start();
	        new Thread(consumerB).start();
	    }
	}
	
	class Producer implements Runnable {
	
	    private EventStorage storage;
	
	    public Producer(EventStorage storage) {
	        this.storage = storage;
	    }
	
	    @Override
	    public void run() {
	        for (int i = 0; i < 100; i++) {
	            storage.put();
	        }
	    }
	}
	
	class Consumer implements Runnable {
	
	    private EventStorage storage;
	
	    public Consumer(EventStorage storage) {
	        this.storage = storage;
	    }
	
	    @Override
	    public void run() {
	        for (int i = 0; i < 100; i++) {
	            storage.take();
	        }
	    }
	}
	
	class EventStorage {
	    private int maxSize;
	    private LinkedList<Date> storage;
	
	    public EventStorage(int maxSize) {
	        this.maxSize = maxSize;
	        this.storage = new LinkedList<>();
	    }
	
	    public synchronized void put() {
	        while (storage.size() == maxSize) {
	            try {
	                this.wait();
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	        }
	        storage.addLast(new Date());
	        System.out.println("仓库里有了" + storage.size() + "个产品");
	        this.notifyAll();
	    }
	
	    public synchronized void take() {
	        while (storage.isEmpty()) {
	            try {
	                this.wait();
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	        }
	        System.out.println("拿到了" + storage.removeFirst() + "还剩下" + storage.size() + "个产品");
	        this.notifyAll();
	    }
	}
```

两个线程交替打印0-100的奇偶数。

代码如下：

```java

	/**
	 * @author gelong
	 * @date 2020/6/16 23:01
	 */
	public class WaitNotifyPrintOddEvenSync {
	
	    private static final Object lock = new Object();
	    private static int count = 0;
	
	    public static void main(String[] args) {
	        new Thread(() -> {
	            while (count < 100) {
	                synchronized (lock) {
	                    if ((count & 1) == 0) {
	                        System.out.println(Thread.currentThread().getName() + "：" + count++);
	                    }
	                }
	            }
	        }, "偶数").start();
	        new Thread(() -> {
	            while (count < 100) {
	                synchronized (lock) {
	                    if ((count & 1) == 1) {
	                        System.out.println(Thread.currentThread().getName() + "：" + count++);
	                    }
	                }
	            }
	        }, "奇数").start();
	    }
	}
```

这种使用synchronized打印奇偶数的方法效率不高，因为当某个线程一直持有锁时只有一次循环是有效的，后面的循环都是废操作。

用wait、notify就可以解决上面的问题。

代码如下：

```java

	/**
	 * @author gelong
	 * @date 2020/6/16 23:21
	 */
	public class WaitNotifyPrintOddEvenWait {
	
	    private static final Object lock = new Object();
	    private static int count = 0;
	
	    static class TurningRunner implements Runnable {
	
	        @Override
	        public void run() {
	            while (count <= 100) {
	                synchronized (lock) {
	                    System.out.println(Thread.currentThread().getName() + "：" + count++);
	                    lock.notify();
	                    if (count <= 100) {
	                        try {
	                            lock.wait();
	                        } catch (InterruptedException e) {
	                            e.printStackTrace();
	                        }
	                    }
	                }
	            }
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Runnable runnable = new TurningRunner();
	        Thread threadA = new Thread(runnable, "偶数");
	        Thread threadB = new Thread(runnable, "奇数");
	        threadA.start();
	        Thread.sleep(10);
	        threadB.start();
	    }
	}
```