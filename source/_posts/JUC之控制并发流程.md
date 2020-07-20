---
title: JUC之控制并发流程
date: 2020-07-17 22:42:42
categories:
- 多线程
tags:
- Java
- 多线程
---

JUC提供了一些控制并发流程的工具类，作用就是帮助我们程序员更容易的让线程之间进行合作，让线程之间相互配合，来满足业务需求。

<!-- more -->

## 概览 ##

控制并发流程的工具类如下:

![concurrent.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggucrzg75kj20yy0khagx.jpg)

## CountDownLatch ##

CountDownLatch使用一个计数器，计数器初始值为线程的数量。当每一个线程完成自己任务后，计数器的值就会减一。当计数器的值为0时，表示所有的线程都已经完成一些任务，然后在CountDownLatch上等待的线程就可以恢复执行接下来的任务。需要注意的CountDownLatch不能重置计数值，一个CountDownLatch使用完毕，如果还想进行倒数，就得重新new一个对象。

CountDownLatch主要方法如下：

- CountDownLatch(int count)：仅有的一个构造函数，count就是需要传入的计数值。
- await()：调用await()的线程将会被挂起，直到count为0时才继续执行。
- countDown()：将count减1，直到为0时，等待的线程被唤醒。

CountDownLatch用法一：一个线程等待多个线程都执行完毕，再继续执行自己的工作。代码如下：

```
	import java.util.Random;
	import java.util.concurrent.CountDownLatch;
	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	
	/**
	 * @author gelong
	 * @date 2020/7/18 18:15
	 */
	public class CountDownLatchDemo1 {
	    public static void main(String[] args) throws InterruptedException {
	        CountDownLatch latch = new CountDownLatch(5);
	        ExecutorService executorService = Executors.newFixedThreadPool(5);
	        Runnable runnable = () -> {
	            try {
	                Thread.sleep(new Random().nextInt(1000));
	                System.out.println(Thread.currentThread().getName() + "已完成");
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            } finally {
	                latch.countDown();
	            }
	        };
	        for (int i = 0; i < 5; i++) {
	            executorService.execute(runnable);
	        }
	        latch.await();
	        System.out.println("所有线程已完成");
	        executorService.shutdown();
	    }
	}
```

![countdownlatch.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggvbuj2o58j20hr093gm3.jpg)

CountDownLatch用法二：多个线程等待某一个线程的信号，同时开始执行。代码如下：

```java

	import java.util.concurrent.CountDownLatch;
	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	
	/**
	 * @author gelong
	 * @date 2020/7/18 18:15
	 */
	public class CountDownLatchDemo2 {
	    public static void main(String[] args) throws InterruptedException {
	        CountDownLatch latch = new CountDownLatch(1);
	        ExecutorService executorService = Executors.newFixedThreadPool(5);
	        Runnable runnable = () -> {
	            try {
	                System.out.println(Thread.currentThread().getName() + "准备开始执行");
	                latch.await();
	                System.out.println(Thread.currentThread().getName() + "开始执行");
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            } finally {
	                latch.countDown();
	            }
	        };
	        for (int i = 0; i < 5; i++) {
	            executorService.execute(runnable);
	        }
	        Thread.sleep(1000);
	        System.out.println("所有线程可以开始执行");
	        latch.countDown();
	        executorService.shutdown();
	    }
	}
```

![countdownlatch2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggvc90go7fj20j50drjsf.jpg)

## Semaphore ##

Semaphore(信号量)是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。假如这里有N个资源，那就对应于N个许可证，同一时刻也只能有N个线程访问。一个线程获取许可证就调用acquire方法，用完了释放资源就调用release方法。如图：

![semaphore.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggvgrrrpvuj211u0chdj7.jpg)

Semaphore的主要方法如下：

- acquire()：从此信号量获取一个许可，在提供一个许可前一直将线程阻塞，否则线程被中断。
- acquire(int permits)：获取permits个信号量，否则等待。
- release()：释放一个许可，将其返回给信号量。
- release(int permits)：释放permits个获取的信号量。
- tryAcquire()：尝试获取一个信号量，获取成功返回true。
- tryAcquire(long timeout, TimeUnit unit)()：尝试获取一个信号量，直到出了指定的等待时间，获取成功返回true。

Semaphore的基本用法如下：

```java

	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	import java.util.concurrent.Semaphore;
	
	/**
	 * @author gelong
	 * @date 2020/7/19 23:40
	 */
	public class SemaphoreDemo {
	
	    private static Semaphore semaphore = new Semaphore(3, true);
	
	    public static void main(String[] args) {
	        ExecutorService executorService = Executors.newFixedThreadPool(50);
	        Runnable runnable = () -> {
	            try {
	                semaphore.acquire();
	                System.out.println(Thread.currentThread().getName() + "获取到了许可证");
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	            try {
	                Thread.sleep(1000);
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            } finally {
	                System.out.println(Thread.currentThread().getName() + "释放了许可证");
	                semaphore.release();
	            }
	        };
	        for (int i = 0; i < 50; i++) {
	            executorService.execute(runnable);
	        }
	        executorService.shutdown();
	    }
	}
```

![semaphore2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggwq1eixaaj20ji0en75l.jpg)

## Condition接口 ##

Synchronized加锁状态时，是使用wait/notify/notifyAll进行线程间的通信。那么在使用ReentrantLock加锁时，也可以通过Condition进行线程间的通信，Condition中await()、signal()、signalAll()分别对应Object中的wait()、notify()、notifyAll()方法，但是Condition更加灵活。

Condition常用的方法如下：

- await()：使当前线程等待，同时释放当前锁，当其他线程中使用signal()时或者signalAll()方法时，线程会重新获得锁并继续执行。或者当线程被中断时，也能跳出等待。
- await(long time, TimeUnit unit)：当前线程等待，直到线程被唤醒或者中断，或者等待时间耗尽。
- signal()：唤醒一个等待的线程。
- signalAll()：唤醒所有等待的线程。

Condition基本用法如下：

```java

	import java.util.concurrent.locks.Condition;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/20 0:13
	 */
	public class ConditionDemo1 {
	
	    private static ReentrantLock lock = new ReentrantLock();
	    private static Condition condition = lock.newCondition();
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread thread1 = new Thread(() -> {
	             lock.lock();
	             try {
	                 System.out.println(Thread.currentThread().getName() + "进入等待状态");
	                 condition.await();
	                 System.out.println(Thread.currentThread().getName() + "被唤醒了");
	             } catch (InterruptedException e) {
	                 e.printStackTrace();
	             } finally {
	                 lock.unlock();
	             }
	        });
	        Thread thread2 = new Thread(() -> {
	            lock.lock();
	            try {
	                System.out.println(Thread.currentThread().getName() + "唤醒其他线程");
	                condition.signal();
	            } finally {
	                lock.unlock();
	            }
	        });
	        thread1.start();
	        thread2.start();
	        thread1.join();
	        thread2.join();
	    }
	}
```

![condition.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggwqu7a9qnj20gp07dt90.jpg)

也可以使用Condition实现生产者消费者模式，代码如下：

```java

	import java.util.ArrayDeque;
	import java.util.Deque;
	import java.util.concurrent.locks.Condition;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author gelong
	 * @date 2020/7/20 0:13
	 */
	public class ConditionDemo2 {
	
	    private static ReentrantLock lock = new ReentrantLock();
	    private static Condition consumer = lock.newCondition();
	    private static Condition producer = lock.newCondition();
	    private static int size = 10;
	    private static Deque<Integer> queue = new ArrayDeque<>(size);
	
	    public static void main(String[] args) {
	        Thread consumer = new Consumer();
	        Thread producer = new Producer();
	        consumer.start();
	        producer.start();
	    }
	
	    static class Consumer extends Thread {
	
	        @Override
	        public void run() {
	            put();
	        }
	
	        private void put() {
	            while (true) {
	                lock.lock();
	                try {
	                    while (queue.size() == 0) {
	                        System.out.println("队列空，等待生产数据");
	                        consumer.await();
	                    }
	                    queue.poll();
	                    System.out.println("消费者消费了一个数据，还剩" + queue.size() + "个");
	                    Thread.sleep(100);
	                    producer.signalAll();
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                } finally {
	                    lock.unlock();
	                }
	            }
	        }
	    }
	
	    static class Producer extends Thread {
	
	        @Override
	        public void run() {
	            take();
	        }
	
	        private void take() {
	            while (true) {
	                lock.lock();
	                try {
	                    while (queue.size() == size) {
	                        System.out.println("队列满了，等待消费数据");
	                        producer.await();
	                    }
	                    queue.offer(1);
	                    System.out.println("生产者生产了一个数据，还剩" + queue.size() + "个");
	                    Thread.sleep(100);
	                    consumer.signalAll();
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                } finally {
	                    lock.unlock();
	                }
	            }
	        }
	    }
	}
```

![condition2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggx9zt2lxjj20hw0khq4c.jpg)

## CyclicBarrier ##

CyclicBarrier循环栅栏和CountDownLatch很类似，都能阻塞一组线程。CyclicBarrier要等固定数量的线程都到达了栅栏位置才能继续执行，而CountDownLatch只需等待数字到0，也就是说，CountDownLatch作用于事件，CyclicBarrier作用与线程。并且CyclicBarrier可以被重用。

当有大量线程相互配合，分别计算不同任务，并且需要最后统一汇总的时候，我们就可以使用CyclicBarrier。CyclicBarrier可以构造一个集结点，当某一个线程执行完毕，它就会到集结点进行等待，直到所有的线程都到了集结点，那么该栅栏就会被撤销，所有线程再统一出发，继续执行剩下的任务。

CyclicBarrier示例代码如下：

```java

	import java.util.Random;
	import java.util.concurrent.BrokenBarrierException;
	import java.util.concurrent.CyclicBarrier;
	
	/**
	 * @author gelong
	 * @date 2020/7/20 11:58
	 */
	public class CyclicBarrierDemo {
	
	    private static CyclicBarrier barrier = new CyclicBarrier(5, () -> System.out.println("所有线程已经集合，统一出发"));
	
	    public static void main(String[] args) {
	        Runnable runnable = () -> {
	            System.out.println(Thread.currentThread().getName() + "前往集合地点");
	            try {
	                Thread.sleep(new Random().nextInt(1000));
	                System.out.println(Thread.currentThread().getName() + "已经到了集合地点，等待其他线程");
	                barrier.await();
	                System.out.println(Thread.currentThread().getName() + "已经出发");
	            } catch (InterruptedException | BrokenBarrierException e) {
	                e.printStackTrace();
	            }
	        };
	        for (int i = 0; i < 5; i++) {
	            new Thread(runnable).start();
	        }
	    }
	}
```

![cyclicbarrier.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggxb2xm84dj20iz0i8myl.jpg)

