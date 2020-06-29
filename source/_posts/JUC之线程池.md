---
title: JUC之线程池
date: 2020-06-27 18:58:10
categories:
- 多线程
tags:
- JUC
- 多线程
---
在实际使用中，线程是很占用系统资源的，如果对线程管理不善很容易导致系统问题。因此，在大多数并发框架中都会使用线程池来管理线程。

<!-- more -->

## 1. 为什么要使用线程池

如果不使用线程池，每一个任务都新开一个线程处理，这样开销太大，我们希望有固定数量的线程，来执行一些任务，这样就避免了反复创建线程并销毁线程所带来的开销问题。使用线程池管理线程主要有如下好处：

- 降低资源消耗。通过复用已存在的线程和降低线程关闭的次数来尽可能降低系统性能损耗。

- 加快系统响应速度。通过复用线程，省去创建线程的过程，因此整体上提升了系统的响应速度。

- 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，因此，需要使用线程池来管理线程。

## 2. 线程池的参数

```java

	public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

- corePoolSize：指的是核心线程数，线程池在完成初始化后，默认情况下，线程池中并没有任何线程，线程池会等待有任务来时，再创建新的线程去执行任务。
- maximumPoolSize：表示线程池能创建线程的最大个数。如果当阻塞队列已满时，并且当前线程池线程个数没有超过maximumPoolSize的话，就会创建新的线程来执行任务。
- keepAliveTime：空闲线程存活时间。如果当前线程池的线程个数已经超过了corePoolSize，并且线程空闲时间超过了keepAliveTime的话，就会将这些空闲线程销毁，这样可以尽可能降低系统资源消耗。
- unit：时间单位。为keepAliveTime指定时间单位。
- workQueue：任务存储队列。
- threadFactory：当线程池需要新的线程的时候，就会使用threadFactory来生成新的线程。
- handler：由于线程池无法接受你所提交的任务的拒绝策略。

线程的添加规则如下：

1. 如果线程数小于corePoolSize，即使其他工作线程处于空闲状态，也会创建一个新的线程来执行新任务。
2. 如果线程数 > corePoolSize则放入workQueue。
3. 如果workQueue已满。并且当前线程数 <= maximumPoolSize，则创建一个新线程来执行任务。

整体流程如下：

![thread-add.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gg76rfxiutj213t0igq73.jpg)

## 3.线程池的用法

线程池Executor家族如下

![](http://ww1.sinaimg.cn/large/b1bbb565gy1gg9lxs6bidj20mk0j3glv.jpg)

java.util.concurrent.ThreadPoolExecutor是真正意义上的线程池，java.util.concurrent.Executors是一个工具类，为我们提供了构造线程池的便捷方法。Executors提供了如下的这几种常用的线程池：

### FixedThreadPool ###

FixedThreadPool传进去的LinkedBlockingQueue是没有容量上限的，所以请求数越来越多，并且无法及时处理完毕的时候，会容易造成占用大量的内存，可能导致OOM。代码如下：

```java

	public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
	}
```

### SingleThreadExecutor ###

SingleThreadExecutor和newFixedThreadPool差不多，只不过把线程数直接设置成了1，所以请求数越来越多，并且无法及时处理完毕的时候，会容易造成占用大量的内存，可能导致OOM。代码如下：

```java

	public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
	}
```

### CachedThreadPool ###

CachedThreadPool在执行过程中通常会创建与所需数量相同的线程，然后在它回收旧线程时停止创建新线程。SynchronousQueue装等待的任务，这个阻塞队列没有存储空间，这意味着只要有请求到来，就必须要找到一条工作线程处理他。这种线程池同样会造成导致OOM。代码如下：

```java

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```


### ScheduledThreadPoolExecutor ###

ScheduledThreadPoolExecutor支持定时执行任务，创建一个corePoolSize为传入参数。
可以调用schedule方法执行任务。代码如下：

```java

    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }


	// 达到给定的延时时间后，执行任务
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);
```

还可以进行周期性执行任务，代码如下：

```java
	
	// 当达到延时时间initialDelay后，任务开始执行。是以上一个任务开始的时间计时，period时间过去后，
	// 检测上一个任务是否执行完毕，如果上一个任务执行完毕，
	// 则当前任务立即执行，如果上一个任务没有执行完毕，则需要等上一个任务执行完毕后立即执行
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

	// 当达到延时时间initialDelay后，任务开始执行。上一个任务执行结束后到下一次
	// 任务执行，中间延时时间间隔为delay。以这种方式，周期性执行任务。
	public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
	                                                     long initialDelay,
	                                                     long delay,
	                                                     TimeUnit unit);
```

关闭线程池我们可以调用shutdown()方法，调用之后已提交任务会继续执行且不接受新任务。当线程池开始关闭的时候isShutdown()方法会返回true，只有当线程池所有的任务都执行完毕之后线程池才会彻底关闭，此时调用isTerminated()方法会返回true。代码如下：

```java

	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	
	/**
	 * @author gelong
	 * @date 2020/6/28 23:16
	 */
	public class Shutdown {
	    public static void main(String[] args) throws InterruptedException {
	        ExecutorService service = Executors.newFixedThreadPool(10);
	        for (int i = 0; i < 100; i++) {
	            service.execute(new Tack());
	        }
	        Thread.sleep(100);
	        service.shutdown();
	        System.out.println("1：" + service.isShutdown());
	        System.out.println("2：" + service.isTerminated());
	        Thread.sleep(1000);
	        System.out.println("3：" + service.isTerminated());
	    }
	}
	
	class Tack implements Runnable {
	
	    @Override
	    public void run() {
	        try {
	            Thread.sleep(10);
	            System.out.println(Thread.currentThread().getName());
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	    }
	}
```

![shutdown.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gg8ffu210fj20jm0ftq3o.jpg)

除了shutdown()方法以外，waitTermination()这个方法可以判断当前线程池是否为TERMINATED状态，如果不是则睡眠指定的时间，如果睡眠中途线程池变为终止态则会被唤醒并且返回true，睡眠时间过后线程池不是TERMINATED状态则会返回false。代码如下：

```java

	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	import java.util.concurrent.TimeUnit;
	
	/**
	 * @author gelong
	 * @date 2020/6/28 23:16
	 */
	public class Shutdown {
	    public static void main(String[] args) throws InterruptedException {
	        ExecutorService service = Executors.newFixedThreadPool(10);
	        for (int i = 0; i < 100; i++) {
	            service.execute(new Tack());
	        }
	        System.out.println(service.awaitTermination(1, TimeUnit.SECONDS));
	        service.shutdown();
	        System.out.println(service.awaitTermination(3, TimeUnit.SECONDS));
	    }
	}
	
	class Tack implements Runnable {
	
	    @Override
	    public void run() {
	        try {
	            Thread.sleep(10);
	            System.out.println(Thread.currentThread().getName());
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	    }
	}
```

![awaitTermination.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gg8ga9axnuj20ge09g74j.jpg)

shutdownNow()方法也可以关闭线程池，不过这个方法比较暴力。shutdownNow()会尝试interrupt线程池中正在执行的线程，阻塞队列的线程也会被取消，但是并不能保证一定能成功的interrupt线程池中的线程，并且会返回并未终止的线程列表。代码如下：

```java

	import java.util.List;
	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	
	
	/**
	 * @author gelong
	 * @date 2020/6/28 23:16
	 */
	public class Shutdown {
	    public static void main(String[] args) throws InterruptedException {
	        ExecutorService service = Executors.newFixedThreadPool(10);
	        for (int i = 0; i < 100; i++) {
	            service.execute(new Tack());
	        }
	        List<Runnable> runnables = service.shutdownNow();
	        System.out.println(runnables);
	    }
	}
	
	class Tack implements Runnable {
	
	    @Override
	    public void run() {
	        try {
	            Thread.sleep(10);
	            System.out.println(Thread.currentThread().getName());
	        } catch (InterruptedException e) {
	            System.out.println(Thread.currentThread().getName() + "被中断了");
	        }
	    }
	}
```

![shutdownNow.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gg8gog1abmj20m008k0t4.jpg)

## 4. 线程池的状态和使用注意点

线程池一共有5种状态，分别是：

- RUNNING：接受新任务并处理排队任务
- SHUTDOWN：不接受新任务，但处理排队任务
- STOP：不接受新任务，也不处理排队任务，并中断正在进行的任务
- TIDYING：所有任务都已终止，workerCount为0时，线程会切换到TIDYING状态，并运行terminated()方法
- TERMINATED：terminated()运行完成

使用线程的注意点：

- 避免任务堆积
- 避免线程数过度增加
- 排查线程泄露

## 5. 总结

线程池能提高我们的效率，Java 本身提供了工具类java.util.concurrent.Executors便于我们构造线程池，但是在实际开发中我们应该根据业务使用ThreadPoolExecutor的构造方法来构造线程池。
