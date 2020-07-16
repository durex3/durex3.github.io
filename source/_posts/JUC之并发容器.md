---
title: JUC之并发容器
date: 2020-07-14 21:27:34
tags:
---
Java的集合框架中，容器主要分为List、Set、Queue、Map四大类，常用的容器ArrayList、LinkedList、HashSet、HashMap等都不是线程安全的。为了保证线程安全Java提供了同步容器和并发容器。

<!-- more -->

## 同步容器 ##

常见的同步容器有Vector、Stack、HashTable、静态工厂方法创建的类(由Collections.synchronizedXxxx方法提供)。同步容器的同步原理就是在方法上用synchronized修饰，显然，这种方式比没有使用 synchronized 的容器性能要差。

## 并发容器 ##

java.util.concurrent包中针对并发环境设计的容器。并发容器使用了写时复制、分段锁、CAS等技术，在并发环境下能提供更高的吞吐量。

### ConcurrentHashMap ###

Java7中的ConcurrentHashMap最外层是多个segment，每个segment的底层数据结构与HashMap类似，仍然是数组+链表。

每个segment独立使用了ReentrantLock锁，segment之间互不影响，提高了并发效率。

ConcurrentHashMap默认有16个segment，可以支持16个线程并发写。这个默认值可以在初始化时进行指定，但是一旦初始化后就不能扩容了。如图：

![ConcurrentHashMap.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggs0fjjdt0j20wc0nf42l.jpg)

Java8中放弃了Segment分段锁的设计，使用的是Node+CAS+Synchronized来保证线程安全性。如图：

![ConcurrentHashMap2.jpg](http://ww1.sinaimg.cn/large/b1bbb565gy1ggs0rhzcisj215o0igtbd.jpg)

### CopyOnWriteArrayList ###

CopyOnWriteArrayList是线程安全的ArrayList，读取是完全不用加锁的，写入也不会阻塞读取操作，就是在写入操作时，进行一次自我复制。也就是对原有的数据进行一次复制，将修改的内容写入副本中。写完之后，再将修改完的副本替换原来的数据。这样就可以保证写操作不影响读了，适合读多写少的场景。

对于remove()、clear()跟set()和add()是类似的，这里我们看add()方法原理：在添加的时候就上锁，并复制一个新数组，增加操作在新数组上完成，将array指向到新数组中，最后解锁。源码如下:

```java

    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

再来看看get()方法，源码如下：

```java

    /**
     * {@inheritDoc}
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        return get(getArray(), index);
    }

    /**
     * Gets the array.  Non-private so as to also be accessible
     * from CopyOnWriteArraySet class.
     */
    final Object[] getArray() {
        return array;
    }
```

CopyOnWriteArrayList在使用迭代器遍历的时候，操作的都是原数组，所以在容器遍历的时候对其进行修改不会抛出异常。源码如下：

```java

    /**
     * Returns an iterator over the elements in this list in proper sequence.
     *
     * <p>The returned iterator provides a snapshot of the state of the list
     * when the iterator was constructed. No synchronization is needed while
     * traversing the iterator. The iterator does <em>NOT</em> support the
     * {@code remove} method.
     *
     * @return an iterator over the elements in this list in proper sequence
     */
    public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
    }

    static final class COWIterator<E> implements ListIterator<E> {
        /** Snapshot of the array */
        private final Object[] snapshot;
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code remove}
         *         is not supported by this iterator.
         */
        public void remove() {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code set}
         *         is not supported by this iterator.
         */
        public void set(E e) {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code add}
         *         is not supported by this iterator.
         */
        public void add(E e) {
            throw new UnsupportedOperationException();
        }

        @Override
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            Object[] elements = snapshot;
            final int size = elements.length;
            for (int i = cursor; i < size; i++) {
                @SuppressWarnings("unchecked") E e = (E) elements[i];
                action.accept(e);
            }
            cursor = size;
        }
    }
```

看了上面的实现源码我们就知道CopyOnWriteArrayList有以下缺点：

- 内存占用：每次add()、set()、remove()这些增删改操作都要复制一个数组出来。
- 数据一致性：CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。

### ArrayBlockingQueue与LinkedBlockingQueue ###

ArrayBlockingQueue是采用数组实现的有界阻塞线程安全队列。常用方法如下：

- put(e) ：当阻塞队列满了以后，生产者继续通过 put添加元素，队列会一直阻塞生产者线程，直到队列可用。
- take()：基于阻塞的方式获取队列中的元素，如果队列为空，则take方法会一直阻塞，直到队列中有新的数据可以消费。

put源码如下：

```java

    /**
     * Inserts the specified element at the tail of this queue, waiting
     * for space to become available if the queue is full.
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
		// 阻塞过程中可以被中断
        lock.lockInterruptibly();
        try {
			// 队列满了的情况下，当前线程将会被notFull挂起加到等待队列中
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

LinkedBlockingQueue不同于ArrayBlockingQueue，它如果不指定容量，默认为Integer.MAX_VALUE，也就是无界队列。LinkedBlockingQueues使用了两个lock，分别是takeLock出队锁和putLock入队锁。put源码如下：

```java

	/**
     * Inserts the specified element at the tail of this queue, waiting if
     * necessary for space to become available.
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
     	// 队列临时容量缓存，作为执行唤醒/阻塞线程操作标记
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
		// 队列元素个数
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            // 自旋排除硬件加锁延时问题
            // 如果队列已满，线程阻塞等待
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
 			// 原子操作 i++
            c = count.getAndIncrement();
			// 队列未满，仍可添加，唤醒等待的入列线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
		// c == 0 说明队列为空，唤醒入列线程入列 
        if (c == 0)
            signalNotEmpty();
    }
```