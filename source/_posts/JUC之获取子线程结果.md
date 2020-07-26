---
title: JUC之获取子线程结果
date: 2020-07-26 18:00:59
categories:
- 多线程
tags:
- Java
- 多线程
---

在多线程开发中，我们如果想获取子线程的执行结果需要使用Callable和Future。

<!-- more -->

## 为什么需要Callable和Future ##

Runnable的缺点：没有返回值也不能抛出Checked Exception。

## Callable和Future的关系 ##

JDK1.5开始，提供了一个能返回线程执行结果的接口Callable，需要定义子类去实现Callable接口。接口定义如下：

```java

	@FunctionalInterface
	public interface Callable<V> {
	    /**
	     * Computes a result, or throws an exception if unable to do so.
	     *
	     * @return computed result
	     * @throws Exception if unable to compute a result
	     */
	    V call() throws Exception;
	}
```

Future也是一个接口。用于表示异步计算的结果，我们可以用Future.get()来获取Callable接口返回的结果，还可以用Future.isDone()来判断任务是否已经执行完毕，以及取消任务，限时获取任务结果等。所以Future是一个存储器，它存储了call()方法这个任务的结果，而这个任务的执行时间是无法确定的，这取决于call()方法的执行情况。


## Callable和Future的用法 ##

Future的get()方法可以获取线程的结果的行为取决于Callable.call()方法的状态，这几种状态如下：

- 任务正常完成：get立即返回结果。
- 任务尚未完成(任务未开始或进行中)：get将阻塞直到任务完成。
- 任务执行过程中抛出Exception：get会抛出ExecutionException。
- 任务被取消：get会抛出CancellationException。

get(long timeout, TimeUnit unit)在指定时间内没有获取到结果，会抛出TimeoutException。

我们可以使用线程池的submit()方法来提交Callable任务，submit()方法定义如下：

```java

	<T> Future<T> submit(Callable<T> task);
```

Future获取结果的示例代码如下：

```java

	import java.util.Random;
	import java.util.concurrent.*;
	
	/**
	 * @author gelong
	 * @date 2020/7/26 18:44
	 */
	public class OneFuture {
	
	    public static void main(String[] args) {
	        ExecutorService service = Executors.newFixedThreadPool(10);
	        Future<Integer> future = service.submit(() -> {
	            Thread.sleep(1000);
	            return new Random().nextInt();
	        });
	        try {
	            System.out.println("线程执行结果：" + future.get());
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } catch (ExecutionException e) {
	            e.printStackTrace();
	        }
	        service.shutdown();
	    }
	}
```

![future.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gh4ksy0pglj20gn05zwem.jpg)


任务产生了异常不会马上就抛出，直到get()执行时才会抛出。isDone()只会关心任务是否执行完成，而不会关心任务执行结果的好坏。代码如下：

```java

	import java.util.concurrent.ExecutionException;
	import java.util.concurrent.ExecutorService;
	import java.util.concurrent.Executors;
	import java.util.concurrent.Future;
	
	/**
	 * @author gelong
	 * @date 2020/7/26 19:16
	 */
	public class GetException {
	
	    public static void main(String[] args) {
	        ExecutorService service = Executors.newFixedThreadPool(10);
	        Future<Integer> future = service.submit(() -> {
	            throw new IllegalArgumentException();
	        });
	        try {
	            Thread.sleep(2000);
	            System.out.println("线程是否执行完毕：" + future.isDone());
	            System.out.println("线程执行结果：" + future.get());
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } catch (ExecutionException e) {
	            e.printStackTrace();
	        }
	        service.shutdown();
	    }
	}
````

![future2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gh4ldfjwtzj20we0a8myb.jpg)

cancel(boolean mayInterruptIfRunning)方法可以取消线程，分以下这三种情况：

- 如果线程没有开始：直接取消方法返回true。
- 如果线程已完成或者已取消：返回false。
- 如果线程已经开始执行：那么将会根据参数是否中断执行的线程。当mayInterruptIfRunning为false的时候，任务会继续执行。当mayInterruptIfRunning为true的时候，会给执行任务的线程一个中断的信号。

cancel实例代码如下：

```java

	import java.util.concurrent.*;
	
	/**
	 * @author gelong
	 * @date 2020/7/26 21:12
	 */
	public class Cancel {
	
	    public static void main(String[] args) throws InterruptedException {
	        ExecutorService service = Executors.newFixedThreadPool(10);
	        Future<Integer> future = service.submit(() -> {
	            try {
	                Thread.sleep(1000);
	            } catch (InterruptedException e) {
	                System.out.println("任务被中断了");
	            }
	            return 1;
	        });
	        Thread.sleep(100);
	        try {
	            boolean cancel = future.cancel(true);
	            System.out.println("线程是否取消成功：" + cancel);
	            System.out.println("线程执行结果：" + future.get());
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } catch (ExecutionException e) {
	            e.printStackTrace();
	        } catch (CancellationException e) {
	            e.printStackTrace();
	        }
	        service.shutdown();
	    }
	}
```

![cancel.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1gh4pvusk62j20n708h3z1.jpg)


## FutureTask ##

FutureTask除了实现Future接口外，还实现了Runnable接口。因此，FutureTask可以交给Executor执行，也可以由Thread类直接执行。

FutureTask实例代码如下：

```java

	import java.util.concurrent.ExecutionException;
	import java.util.concurrent.FutureTask;
	
	/**
	 * @author gelong
	 * @date 2020/7/26 22:40
	 */
	public class FutureTaskDemo {
	
	    public static void main(String[] args) {
	        FutureTask<Integer> futureTask = new FutureTask<>(() -> 1);
	        Thread thread = new Thread(futureTask);
	        thread.start();
	        try {
	            System.out.println("获取线程结果：" + futureTask.get());
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } catch (ExecutionException e) {
	            e.printStackTrace();
	        }
	    }
	}
```