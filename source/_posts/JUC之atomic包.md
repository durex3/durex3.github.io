---
title: JUC之atomic包
date: 2020-07-09 22:52:44
categories:
- 多线程
tags:
- Java
- 多线程
---

Java从JDK 1.5开始提供了java.util.concurrent.atomic包，这个包中的原子操作类提供了一种用法简单、性能高效、线程安全地更新一个变量的方式。

<!-- more -->

## 什么是原子类 ##

所谓原子操作是指不会被线程调度机制打断的操作，这种操作一旦开始，就一直运行到结束。

原子类的作用和锁类似，是为了保证并发情况下线程安全。不过原子类相比于锁，有一定优势：

- 粒度更细：原子变量可以把竞争范围缩小到变量级别。
- 效率更高：通常，使用原子类的效率会比锁更高，除了高度竞争的情况。

在atomic包中的这些原子类大致分为：

![atomic.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggl4kxhqrgj20wx0kjn19.jpg)

## 原子类的基本使用 ##

### AtomicInteger ###

AtomicInteger是Integer类型的线程安全原子类，可以在程序中以原子的方式更新int值。代码如下：

```java

	import java.util.concurrent.atomic.AtomicInteger;
	
	/**
	 * @author gelong
	 * @date 2020/7/9 23:36
	 */
	public class AtomicIntegerDemo1 implements Runnable {
	
	    private static AtomicInteger atomicInteger = new AtomicInteger(0);
	    private static int basicCount = 0;
	
	    public void incrementAtomic() {
	        atomicInteger.getAndIncrement();
	    }
	
	    public void incrementBasic() {
	        basicCount++;
	    }
	
	    @Override
	    public void run() {
	        for (int i = 0; i < 10000; i++) {
	            incrementAtomic();
	            incrementBasic();
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Runnable runnable = new AtomicIntegerDemo1();
	        Thread thread1 = new Thread(runnable);
	        Thread thread2 = new Thread(runnable);
	        thread1.start();
	        thread2.start();
	        thread1.join();
	        thread2.join();
	        System.out.println("atomic：" + atomicInteger.get());
	        System.out.println("basic：" + basicCount);
	    }
	}
```

![atomicinteger.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggl5xnswqsj20gp06kdg0.jpg)

### AtomicIntegerArray ###

原子数组并不是说可以让线程以原子方式一次性地操作数组中所有元素的数组，而是指对于数组中的每个元素，可以以原子方式进行操作。代码如下：

```java

	import java.util.concurrent.atomic.AtomicIntegerArray;
	
	/**
	 * @author gelong
	 * @date 2020/7/10 23:06
	 */
	public class AtomicArrayDemo1 {
	
	    private static AtomicIntegerArray array = new AtomicIntegerArray(1000);
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread[] threadsIncrementer = new Thread[100];
	        Thread[] threadsDecrementer = new Thread[100];
	        Runnable incrementer = new Incrementer(array);
	        Runnable decrementer = new Decrementer(array);
	        for (int i = 0; i < 100; i++) {
	            threadsIncrementer[i] = new Thread(incrementer);
	            threadsDecrementer[i] = new Thread(decrementer);
	            threadsIncrementer[i].start();
	            threadsDecrementer[i].start();
	            threadsIncrementer[i].join();
	            threadsDecrementer[i].join();
	        }
	        for (int i = 0; i < array.length(); i++) {
	            if (array.get(i) != 0) {
	                System.out.println(i + "出错了");
	            }
	        }
	        System.out.println("运行结束");
	    }
	}
	
	class Incrementer implements Runnable {
	
	    private AtomicIntegerArray array;
	
	    public Incrementer(AtomicIntegerArray array) {
	        this.array = array;
	    }
	
	    @Override
	    public void run() {
	        for (int i = 0; i < array.length(); i++) {
	            array.getAndIncrement(i);
	        }
	    }
	}
	
	class Decrementer implements Runnable {
	
	    private AtomicIntegerArray array;
	
	    public Decrementer(AtomicIntegerArray array) {
	        this.array = array;
	    }
	
	    @Override
	    public void run() {
	        for (int i = 0; i < array.length(); i++) {
	            array.getAndDecrement(i);
	        }
	    }
	}
```

![atomicintegerarray.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggmam35nwnj20gw0750su.jpg)

### AtomicReference ###

AtomicReference和AtomicInteger非常类似，不同之处就在于AtomicInteger是对整数的封装，而AtomicReference则对应普通的对象引用，可以保证你在修改对象引用时的线程安全性。AtomicReference有一个compareAndSet()方法，它可以将引用与预期引用进行比较，如果它们相等，则设置一个新的引用。代码如下：

```java

	import java.util.concurrent.atomic.AtomicReference;
	
	/**
	 * @author gelong
	 * @date 2020/7/10 23:38
	 */
	public class AtomicReferenceDemo1 {
	
	    private static AtomicReference<String> reference = new AtomicReference<>("init");
	
	    public static void main(String[] args) {
	        System.out.println(reference.compareAndSet("init", "new"));
	        System.out.println("修改之后的值：" + reference.get());
	        System.out.println(reference.compareAndSet("init", "new"));
	    }
	}
```

![atomicreference.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggmb1a0el9j20ik07g74h.jpg)

### AtomicIntegerFieldUpdater ###

AtomicIntegerFieldUpdater可以使普通变量升级为原子变量，一般使用在偶尔需要一个原子操作的场景。需要注意的是要升级的变量不能是static的。代码如下：

```java

	import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;
	
	/**
	 * @author gelong
	 * @date 2020/7/11 13:06
	 */
	public class AtomicIntegerFieldUpdaterDemo1 implements Runnable {
	
	    private static Candidate a = new Candidate();
	    private static Candidate b = new Candidate();
	    private AtomicIntegerFieldUpdater<Candidate> updater = AtomicIntegerFieldUpdater
	            .newUpdater(Candidate.class, "score");
	
	    @Override
	    public void run() {
	        for (int i = 0; i < 10000; i++) {
	            a.score++;
	            updater.getAndIncrement(b);
	        }
	    }
	
	    public static void main(String[] args) throws InterruptedException {
	        Runnable runnable = new AtomicIntegerFieldUpdaterDemo1();
	        Thread thread1 = new Thread(runnable);
	        Thread thread2 = new Thread(runnable);
	        thread1.start();
	        thread2.start();
	        thread1.join();
	        thread2.join();
	        System.out.println("a：" + a.score);
	        System.out.println("b：" + b.score);
	    }
	}
	
	class Candidate {
	    volatile int score;
	}
```

![updater.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggmyumigl8j20i307y0sw.jpg)

### LongAdder ###

LongAdder在高并发的场景下会AtomicLong具有更好的性能，代价是消耗更多的内存空间。LongAdder内部有一个base变量和一个cell数组，竞争不激烈条件下，直接累加到base变量上，竞争激烈条件下，累加各个线程自己的值到cell数组中。最终结果的计算是 base + cell数组的所有值，源码如下：

```java

	public long sum() {
        Cell[] as = cells; 
		Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```
需要注意的是sum()方法没有加锁，在并发的情况下，这个方法只能得到某个时刻的近似值，这也就是LongAdder并不能完全替代AtomicLong的原因之一。

### LongAccumulator ###

LongAccumulator是LongAdder的功能增强版。LongAdder的API只有对数值的加减，而LongAccumulator提供了自定义的函数操作。其构造函数如下：

```java

    public LongAccumulator(LongBinaryOperator accumulatorFunction,
                           long identity) {
        this.function = accumulatorFunction;
        base = this.identity = identity;
    }
```

- accumulatorFunction：需要执行的二元函数，接收2个long，并返回1个long
- identity：初始值

示例代码如下：

```java

	import java.util.concurrent.atomic.LongAccumulator;
	
	/**
	 * @author gelong
	 * @date 2020/7/11 14:17
	 */
	public class LongAccumulatorDemo1 {
	
	    private static LongAccumulator accumulator = new LongAccumulator((x, y) -> x + y, 0);
	
	    public static void main(String[] args) {
	        accumulator.accumulate(5);
	        System.out.println(accumulator.get());
	    }
	}
```