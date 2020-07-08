---
title: JUC之不可不说的“锁”事
date: 2020-07-02 23:19:05
categories:
- 多线程
tags:
- JUC
- 多线程
---

相比同步锁，JUC包中的锁的功能更加强大，锁的种类也非常多，包括可重入锁和非重入锁、公平锁和非公平锁、共享锁和排他锁、自旋锁和阻塞锁等等。

<!-- more -->

## Lock接口 ##

Lock接口常用的实现是ReentrantLock。Lock并不是用来替代synchronized的，而是当使用synchronized不合适或不满足要求的时候，来提供高级功能的。synchronized存在以下三个缺点：

1. 效率低：锁的释放情况少、试图获得锁时不能设定超时、不能中断一个正在试图获得锁的线程。
2. 不够灵活：加锁和释放的时机单一，每个锁仅有单一的条件(某个对象)，可能是不够的。
3. 无法知道是否成功获取到锁

在Lock中声明了四个方法来获取锁，分别是：

### lock() ###

平常使用得最多的一个方法，就是用来获取锁。如果锁已被其他线程获取，则进行等待。Lock不会像synchronized一样在异常时自动释放锁，必须主动去释放锁，并且将释放锁的操作放在finally块中进行。代码如下：

```java

	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/4 21:31
	 */
	public class MustUnLock {
	
	    private static Lock lock = new ReentrantLock();
	
	    public static void main(String[] args) {
	        lock.lock();
	        try {
	            System.out.println(Thread.currentThread().getName());
	        } finally {
	            lock.unlock();
	        }
	    }
	}
```

lock()方法不能被中断，一旦陷入死锁，lock()就会陷入永久等待。

### tryLock() ###

tryLock()用来尝试获取锁，如果当前锁没有被其他线程占用，则获取成功，返回true，否则返回false。在拿不到锁时不会一直在那等待。

### tryLock(long time, TimeUnit unit) ###

tryLock(long time, TimeUnit unit)方法和tryLock()方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。代码如下：

```java

	import java.util.concurrent.TimeUnit;
	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/4 21:56
	 */
	public class TryLockDeadlock {
	
	    private static Lock lock1 = new ReentrantLock();
	    private static Lock lock2 = new ReentrantLock();
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread thread1 = new Thread(() -> {
	            for (int i = 0; i < 100; i++) {
	                try {
	                    if (lock1.tryLock(800, TimeUnit.MICROSECONDS)) {
	                        try {
	                            System.out.println(Thread.currentThread().getName() + "获取锁1成功");
	                            Thread.sleep(500);
	                            if (lock2.tryLock(800, TimeUnit.MICROSECONDS)) {
	                                try {
	                                    System.out.println(Thread.currentThread().getName() + "获取锁2成功");
	                                    System.out.println(Thread.currentThread().getName() + "获取两把锁成功");
	                                    break;
	                                } finally {
	                                    lock2.unlock();
	                                }
	                            } else {
	                                System.out.println(Thread.currentThread().getName() + "获取锁2失败，已重试");
	                            }
	                        } finally {
	                            lock1.unlock();
	                            Thread.sleep(500);
	                        }
	                    } else {
	                        System.out.println(Thread.currentThread().getName() + "获取锁1失败，已重试");
	                    }
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                }
	            }
	        });
	        Thread thread2 = new Thread(() -> {
	            for (int i = 0; i < 100; i++) {
	                try {
	                    if (lock2.tryLock(800, TimeUnit.MICROSECONDS)) {
	                        try {
	                            System.out.println(Thread.currentThread().getName() + "获取锁2成功");
	                            Thread.sleep(500);
	                            if (lock1.tryLock(800, TimeUnit.MICROSECONDS)) {
	                                try {
	                                    System.out.println(Thread.currentThread().getName() + "获取锁1成功");
	                                    System.out.println(Thread.currentThread().getName() + "获取两把锁成功");
	                                    break;
	                                } finally {
	                                    lock1.unlock();
	                                }
	                            } else {
	                                System.out.println(Thread.currentThread().getName() + "获取锁1失败，已重试");
	                            }
	                        } finally {
	                            lock2.unlock();
	                            Thread.sleep(500);
	                        }
	                    } else {
	                        System.out.println(Thread.currentThread().getName() + "获取锁2失败，已重试");
	                    }
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                }
	            }
	        });
	        thread1.start();
	        thread2.start();
	        thread1.join();
	        thread2.join();
	    }
	}
```

### lockInterruptibly() ###

lockInterruptibly()相当于tryLock(long time, TimeUnit unit)把超时时间设置为无限。在等待过程中，线程可以被中断。代码如下：

```java

	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/5 12:50
	 */
	public class LockInterruptibly implements Runnable {
	
	    private static Lock lock = new ReentrantLock();
	
	    @Override
	    public void run() {
	        System.out.println(Thread.currentThread().getName() + "尝试获取锁");
	        try {
	            lock.lockInterruptibly();
	            try {
	                System.out.println(Thread.currentThread().getName() + "获取锁成功");
	                Thread.sleep(1000);
	            } catch (InterruptedException e) {
	                System.out.println(Thread.currentThread().getName() + "sleep期间被中断了");
	            } finally {
	                lock.unlock();
	            }
	        } catch (InterruptedException e) {
	            System.out.println(Thread.currentThread().getName() + "获取锁期间被中断了");
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Runnable runnable = new LockInterruptibly();
	        Thread thread1 = new Thread(runnable);
	        Thread thread2 = new Thread(runnable);
	        thread1.start();
	        thread2.start();
	        thread2.interrupt();
	    }
	}
```

![lock_interruptibly.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggg0h5ulb6j20iu07cdg8.jpg)

## 锁的分类 ##

### 可重入锁 ###

可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁，不会因为之前已经获取过还没释放而阻塞。Java中ReentrantLock和synchronized都是可重入锁，可重入锁的优点是可一定程度避免死锁。代码如下：

```java

	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/6 22:14
	 */
	public class RecursionDemo {
	
	    private static ReentrantLock lock = new ReentrantLock();
	
	    public static void accessResource() {
	        lock.lock();
	        try {
	            if (lock.getHoldCount() < 5) {
	                System.out.println("线程锁的个数：" + lock.getHoldCount());
	                accessResource();
	                System.out.println("线程锁的个数：" + lock.getHoldCount());
	            }
	        } finally {
	            lock.unlock();
	        }
	    }
	
	    public static void main(String[] args) {
	        accessResource();
	    }
	}
```

![reentrantlock.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gghm9rgkizj20gu0bm74s.jpg)

### 公平锁和非公平锁 ###

公平锁：是指多个线程按照请求的顺序来获取锁，类似排队打饭，先来后到。

非公平锁：是指多个线程获取锁的顺序并不是按照请求的顺序来获取锁，有可能后申请的线程比先申请的线程优先获取锁，在高并发的情况下，有可能会造成优先级反转或者饥饿现象。，

Synchronized是一种非公平锁，ReentrantLock通过构造函数指定该锁是否是公平锁，默认非公平锁，非公平锁优点在于吞吐量比公平锁大。公平锁代码如下：

```java

	import java.util.Random;
	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/6 23:10
	 */
	public class FairLock {
	
	    public static void main(String[] args) throws InterruptedException {
	        PrintQueue queue = new PrintQueue();
	        Thread[] threads = new Thread[10];
	        Runnable runnable = new Job(queue);
	        for (int i = 0; i < 10; i++) {
	            threads[i] = new Thread(runnable);
	        }
	        for (int i = 0; i < 10; i++) {
	            threads[i].start();
	            Thread.sleep(100);
	        }
	    }
	}
	
	class Job implements Runnable {
	
	    private PrintQueue queue;
	
	    public Job(PrintQueue queue) {
	        this.queue = queue;
	    }
	
	    @Override
	    public void run() {
	        System.out.println(Thread.currentThread().getName() + "开始打印");
	        queue.printJob();
	        System.out.println(Thread.currentThread().getName() + "打印完毕");
	    }
	}
	
	class PrintQueue {
	
	    private Lock lock = new ReentrantLock(true);
	
	    public void printJob() {
	        print();
	        print();
	    }
	
	    private void print() {
	        lock.lock();
	        try {
	            int duration = new Random().nextInt(10) + 1;
	            System.out.println(Thread.currentThread().getName() + "正在打印，需要" + duration + "s");
	            Thread.sleep(duration * 1000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            lock.unlock();
	        }
	    }
	}
```

![fair.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggho3tq9p7j20t80lcjt9.jpg)

可以看到公平锁的打印是按照顺序的，线程获取锁按照请求队列的顺序。ReentrantLock构造函数传入的是false的话，就是非公平锁了。运行结果如下：

![notfair.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gghobvb6d6j20gk0i7abe.jpg)

可以看到每个线程会直接打印两次，并不是按照请求队列的顺序，在其他线程阻塞的时候，当前线程就有可能再次获取锁。

### 共享锁和独占锁 ###

共享锁：共享锁是指该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A再加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。

独占锁：顾名思义就是指该锁一次只能被一个线程所持有。如果线程T对数据A加上排它锁后，则其他线程不能再对A加任何类型的锁。获得排它锁的线程即能读数据又能修改数据。

ReentrantReadWriteLock有两把锁：ReadLock和WriteLock，分别对应共享锁和独占锁。

ReadLock用于只读操作，它是“共享锁”，能同时被多个线程获取。WriteLock用于写入操作，它是“独占锁”，写入锁只能被一个线程获取。代码如下：

```java

	import java.util.concurrent.locks.ReentrantReadWriteLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/7 22:29
	 */
	public class CinemaReadWrite {
	
	    private static final ReentrantReadWriteLock reentrantReadWriteLock = new ReentrantReadWriteLock();
	    private static final ReentrantReadWriteLock.ReadLock readLock = reentrantReadWriteLock.readLock();
	    private static final ReentrantReadWriteLock.WriteLock writeLock = reentrantReadWriteLock.writeLock();
	
	    private static void read() {
	        readLock.lock();
	        try {
	            System.out.println(Thread.currentThread().getName() + "获取了读锁");
	            Thread.sleep(1000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            System.out.println(Thread.currentThread().getName() + "释放了读锁");
	            readLock.unlock();
	        }
	    }
	
	    private static void write() {
	        writeLock.lock();
	        try {
	            System.out.println(Thread.currentThread().getName() + "获取了写锁");
	            Thread.sleep(1000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            System.out.println(Thread.currentThread().getName() + "释放了写锁");
	            writeLock.unlock();
	        }
	    }
	
	    public static void main(String[] args) {
	        new Thread(CinemaReadWrite::read).start();
	        new Thread(CinemaReadWrite::read).start();
	        new Thread(CinemaReadWrite::write).start();
	        new Thread(CinemaReadWrite::write).start();
	    }
	}
```

![read-write.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggis9k5xiwj20hu09ht9a.jpg)

读写锁在公平的情况下是按部就班地排队，假如是非公平的情况：线程2和线程4正在同时读取，线程3想要写入，于是进入等待队列，线程5不在队列里，现在过来想要读取。此时有两种策略。

- 策略一：线程5直接插队。因为读锁可以同时存在多个，所以这种策略效率高。但是这样后面有很多线程都可以插队，线程3可能一直得不到执行，容易造成饥饿。
- 策略二：线程3获取写锁，线程5进入等待队列。在这种策略下可以避免线程饥饿，ReentrantReadWriteLock就是使用此策略。

写锁是可以随时进行插队，但是读锁仅在等待队列头结点不是想获取写锁线程的时候可以插队。源码如下：

```java

	/**
     * Nonfair version of Sync
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```

写锁可以降级为读锁，但是读锁不能升级为写锁。因为多个线程可以存在多读，但是读写不能同时存在，当两个线程都想升级为写锁的时候，必须互相等待对方的读锁释放，这样就会造成死锁。所以当读锁想升级为写锁时就会阻塞，代码如下：

```java

	import java.util.concurrent.locks.ReentrantReadWriteLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/7 22:29
	 */
	public class Upgrading {
	
	    private static final ReentrantReadWriteLock reentrantReadWriteLock = new ReentrantReadWriteLock();
	    private static final ReentrantReadWriteLock.ReadLock readLock = reentrantReadWriteLock.readLock();
	    private static final ReentrantReadWriteLock.WriteLock writeLock = reentrantReadWriteLock.writeLock();
	
	    private static void readUpgrading() {
	        readLock.lock();
	        try {
	            System.out.println(Thread.currentThread().getName() + "获取了读锁");
	            Thread.sleep(1000);
	            System.out.println("升级会带来阻塞");
	            writeLock.lock();
	            System.out.println(Thread.currentThread().getName() + "获取了写锁");
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            System.out.println(Thread.currentThread().getName() + "释放了写锁");
	            writeLock.unlock();
	            System.out.println(Thread.currentThread().getName() + "释放了读锁");
	            readLock.unlock();
	        }
	    }
	
	    private static void writeDowngrading() {
	        writeLock.lock();
	        try {
	            System.out.println(Thread.currentThread().getName() + "获取了写锁");
	            Thread.sleep(1000);
	            readLock.lock();
	            System.out.println(Thread.currentThread().getName() + "获取了读锁");
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            System.out.println(Thread.currentThread().getName() + "释放了读锁");
	            readLock.unlock();
	            System.out.println(Thread.currentThread().getName() + "释放了写锁");
	            writeLock.unlock();
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread thread1 = new Thread(Upgrading::writeDowngrading);
	        thread1.start();
	        thread1.join();
	        System.out.println("--------------");
	        Thread thread2 = new Thread(Upgrading::readUpgrading);
	        thread2.start();
	    }
	}
```

![upgrading.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggjzav28goj20ie0agdgc.jpg)

### 自旋锁 ###

当一个线程尝试去获取某一把锁的时候，如果这个锁此时已经被别人占用，那么此线程不会阻塞，而是不断地尝试获取锁，避免切换线程的开销，这就是自旋锁。如果锁被占用的时间过长，那么自旋的线程只会白白浪费处理器资源。

在Java1.5版本及以上的并发包java.util.concurrent.atomic下的类基本都是自旋锁的实现。自旋锁的实现原理就是CAS，比如AtomicInteger中进行自增操作的源码中的do-while循环就是一个自旋操作，如果修改过程中遇到其他线程竞争导致修改失败，就在while里死循环，直至修改成功。

我们可以实现一个简单的自旋锁。代码如下：

```java

	import java.util.concurrent.atomic.AtomicReference;
	
	/**
	 * @author gelong
	 * @date 2020/7/9 0:04
	 */
	public class SpinLock {
	
	    private AtomicReference<Thread> reference = new AtomicReference<>();
	
	    public void lock() {
	        Thread thread = Thread.currentThread();
	        while (!reference.compareAndSet(null, thread)) {
	            System.out.println(thread.getName() + "获取自旋锁失败，正在重试");
	        }
	    }
	
	    public void unlock() {
	        Thread thread = Thread.currentThread();
	        reference.compareAndSet(thread, null);
	    }
	
	    public static void main(String[] args) {
	        SpinLock lock = new SpinLock();
	        Runnable runnable = () -> {
	            System.out.println(Thread.currentThread().getName() + "开始尝试获取自旋锁");
	            lock.lock();
	            System.out.println(Thread.currentThread().getName() + "获取了自旋锁");
	            try {
	                Thread.sleep(10);
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            } finally {
	                lock.unlock();
	            }
	        };
	        new Thread(runnable).start();
	        new Thread(runnable).start();
	    }
	}
```

![spinlock1.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggk0txqeerj20hg0e9wfx.jpg)

![spinlock2.jpg]](http://ww1.sinaimg.cn/large/b1bbb565gy1ggk0w0eiwfj20h50dvq43.jpg)

### 可中断锁 ###

在Java中，synchronized就属于不可中断锁，而Lock是可中断锁，因为tryLock(long time, TimeUnit unit)和lockInterruptibly()都能响应中断。