---
title: Java多线程之sleep、join、yield
date: 2020-06-17 00:07:03
categories:
- 多线程
tags:
- Java
- 多线程
---
Java在多线程开发中，Object类的sleep、join、yield这三个方法比较常见。

<!-- more -->

## sleep()的作用和用法

sleep方法可以让线程进入Timed_Waiting状态，并且不占用CPU资源，但是不释放锁，直到规定时间后再执行，休眠期间如果被中断，会抛出异常并清除中断状态。

sleep()不会释放monitor锁。代码如下：

```java

	/**
	 * @author gelong
	 * @date 2020/6/17 22:15
	 */
	public class SleepDontReleaseMonitor implements Runnable {
	
	    @Override
	    public synchronized void run() {
	        sync();
	    }
	
	    private void sync() {
	        System.out.println(Thread.currentThread().getName() + "获取了锁");
	        try {
	            Thread.sleep(1000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	        System.out.println(Thread.currentThread().getName() + "退出同步代码块");
	    }
	
	    public static void main(String[] args) {
	        Runnable runnable = new SleepDontReleaseMonitor();
	        Thread threadA = new Thread(runnable);
	        Thread threadB = new Thread(runnable);
	        threadA.start();
	        threadB.start();
	    }
	}
```

![sleep1.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfvpo6e392j20ht06u3yt.jpg)

sleep()也不会释放lock锁。代码如下：

```java

	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/6/17 22:15
	 */
	public class SleepDontReleaseLock implements Runnable {
	
	    private static final Lock lock = new ReentrantLock();
	
	    public static void main(String[] args) {
	        Runnable runnable = new SleepDontReleaseLock();
	        Thread threadA = new Thread(runnable);
	        Thread threadB = new Thread(runnable);
	        threadA.start();
	        threadB.start();
	    }
	
	    @Override
	    public void run() {
	        lock.lock();
	        try {
	            System.out.println(Thread.currentThread().getName() + "获取了锁");
	            Thread.sleep(1000);
	            System.out.println(Thread.currentThread().getName() + "已经苏醒");
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            lock.unlock();
	        }
	    }
	}
```

![sleep2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfvqh9sk72j20he07ujrp.jpg)

sleep也可以响应中断。代码如下：

```java

	import java.util.concurrent.TimeUnit;
	
	/**
	 * @author gelong
	 * @date 2020/6/18 0:24
	 */
	public class SleepInterrupted implements Runnable {
	    @Override
	    public void run() {
	        for (int i = 0; i < 10; i++) {
	            try {
	                System.out.println(i);
	                TimeUnit.SECONDS.sleep(1);
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Runnable runnable = new SleepInterrupted();
	        Thread thread = new Thread(runnable);
	        thread.start();
	        TimeUnit.SECONDS.sleep(3);
	        thread.interrupt();
	    }
	}
```

![sleep3.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfvr9393faj21090ay758.jpg)

wait和sleep方法的相同点：

1. wait和sleep方法都可以使线程阻塞，对应线程状态是Waiting或Time_Waiting。
2. wait和sleep方法都可以响应中断Thread.interrupt()。


wait和sleep方法的不同点：

1. wait方法的执行必须在同步方法中进行，而sleep则不需要。
2. 在同步方法里执行sleep方法时，不会释放monitor锁，但是wait方法会释放monitor锁。
3. sleep()方法短暂休眠之后会主动退出阻塞，而没有指定时间的wait方法则需要被其他线程中断后才能退出阻塞。
4. wait()、notify()/notifyAll()是Object类的方法，sleep()是Thread类的方法。

## join()的作用和用法

join()方法的作用就是：因为新的线程加入了我们，所以我们要等待它执行完再出发。比如main线程里面新建了一个线程并且调用了join()方法，那么main线程就得等待这个子线程执行完毕后才能继续执行。代码如下:

```java

	/**
	 * @author gelong
	 * @date 2020/6/18 22:35
	 */
	public class Join {
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread threadA = new Thread(() -> {
	            try {
	                Thread.sleep(1000);
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	            System.out.println(Thread.currentThread().getName() + "执行完毕");
	        });
	        Thread threadB = new Thread(() -> {
	            try {
	                Thread.sleep(1000);
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	            System.out.println(Thread.currentThread().getName() + "执行完毕");
	        });
	        threadA.start();
	        threadB.start();
	        System.out.println("开始等待子线程执行");
	        threadA.join();
	        threadB.join();
	        System.out.println("所有子线程执行完毕");
	    }
	}
```

![join1.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfwupkh5aqj20hs08sdg5.jpg)

join也可以响应中断，不过不是调用join()方法的线程响应中断而是被加入的线程才能响应中断。比如线程A加入了main线程，那么就是main线程去响应中断。代码如下:

```java

	/**
	 * @author gelong
	 * @date 2020/6/18 22:47
	 */
	public class JoinInterrupted {
	    public static void main(String[] args) {
	        Thread main = Thread.currentThread();
	        Thread threadA = new Thread(() -> {
	            main.interrupt();
	        });
	        threadA.start();
	        try {
	            threadA.join();
	        } catch (InterruptedException e) {
	            System.out.println(Thread.currentThread().getName() + "响应了中断");
	            e.printStackTrace();
	        }
	    }
	}
```

![join2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gfwuqy6gszj20zd0a1t9o.jpg)


## yield()的作用

yield()方法会让出当前线程CPU的时间片，使正在运行中的线程重新Runnable状态，并重新竞争CPU的调度权。它可能会获取到，也有可能被其他线程获取到。yield实际用得并不多，不过在juc工具包中用得比较多。

yield和sleep方法的相同点：

1. yield，sleep 两个在暂停过程中，如已经持有锁，则都不会释放锁资源。 

yield和sleep方法的不同点：

1. yield不能被中断，而sleep则可以响应中断。
2. sleep可以指定具体休眠的时间，而yield则依赖CPU的时间片划分。