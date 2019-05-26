---
layout: post
category: coding
tagline: ""
summary: 
tags: [java,concurrent]
---
{% include JB/setup %}
目录
* toc
{:toc}
### Background ###
{{ page.summary }}

java.util.concurrent 包是java中多线程使用的包。里面的内容包括

- atomic包  提供了一些原子操作的类型，比如atomicLong, atomicReference
- lock包

  - AbstractQueuedSynchronizer, AQS
  - Condition 用于准确通知解锁
  - LockSupport 提供 park 和 unpark方法
  - ReentrantLock可重入锁
  - ReadWriteLock 读写锁
- 线程池类以及线程相关类
- 并发集合

本文讲几个并发集合. `LinkedBlockingQueue` 和`PriorityBlockingQueue`.

#### LinkedBlockingQueue

首先看一下抽象类BlockingQueue有哪些操作。

- add(e)  成功返回true，空间已满抛异常。
- offer(e) 插入成功返回true，当前无空间可用返回false。如果是一个空间限制队列，建议用offer方法。
- put(e)  阻塞一直等待直到插入成功
- offer(e,timeout,timeunit) 有timeout的offer
- take 尝试获得队列头部的元素，阻塞直到获得
- poll(timeout, timeunit) 尝试获得队列头部元素，直到超时
- remove(object) 移除队列中equal的元素，有多个就移除多个，如果队列中包含这个元素，返回true，否则 false

##### Use Case

通常用于生产者消费者模型，线程间通信。

生产者生产任务，然后消费者去取任务。

如spark中，在shuffleFetchIterator中就使用了LinkedBlockingQueue来保存fetch到的数据，将结果保存到阻塞队列，然后取数据的队列调用take方法来取result。

在线程池中，就需要一个阻塞队列作为工作队列。

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
```

这里面的阻塞队列用于让用于来提交task到这个工作队列中，然后线程池中的线程来取这个task进行执行。



#### PriorityBlockingQueue

优先级阻塞队列,是PriorityQueue的线程安全模式。

所以只需要了解一下PriorityQueue,优先级队列，底层实现为堆,是一个无界队列。

优先级队列给每个元素提供了一个优先级，然后对这些元素按照优先级进行排序。如果指定了comparator则按照指定的比较规则进行排序，如果没有指定，那么按照自然序进行排序，如数字就比较大小，小的在前，如果是字符串，则按照字典序。

由于底层为堆，可以用于堆排序，比如获得n个数中前k大的值，属于最优实现。

因为是最小的值放在堆顶，那么只要新的值大于目前已有k个值的最小值，就可以成为前k大，下面是topK的实现。由于优先级队列是无界的，所以需要我们自己来控制是否插入。

```scala
import java.util.PriorityQueue
import scala.collection.JavaConverters._

object HeapSortTopK {
  def topK(arr: Array[Int], k: Int): Array[Int] = {
    if (arr.size <= k) {
      arr
    } else {
      val maxHeap = new PriorityQueue[Int](k)
      for (i <- (0 until arr.length)) {
        if (i < k) {
          maxHeap.offer(arr(i))
        } else {
          maxHeap.offer(math.max(maxHeap.poll(), arr(i)))
        }
      }
      maxHeap.asScala.toArray
    }
  }
}
```

其他操作，比如offer 和 Poll 操作和以上的LinkedBlockingQueue是一样的。

##### Use Case

使用场景是一些需要安排优先级的场景。

在spark中，在`ShutdownHookManager`中使用了优先级队列。因为有些Hoot需要先执行，所以需要安排优先级。

或者是基于无界的PriorityQueue实现有界的优先级队列，只需要在插入元素的时候判断一下目前的size即可，如果已经到达界限，则进行替换。

下面是spark中有界优先级队列的实现。

```scala
import java.io.Serializable
import java.util.{PriorityQueue => JPriorityQueue}

import scala.collection.JavaConverters._
import scala.collection.generic.Growable

/**
 * Bounded priority queue. This class wraps the original PriorityQueue
 * class and modifies it such that only the top K elements are retained.
 * The top K elements are defined by an implicit Ordering[A].
 */
private[spark] class BoundedPriorityQueue[A](maxSize: Int)(implicit ord: Ordering[A])
  extends Iterable[A] with Growable[A] with Serializable {

  private val underlying = new JPriorityQueue[A](maxSize, ord)

  override def iterator: Iterator[A] = underlying.iterator.asScala

  override def size: Int = underlying.size

  override def ++=(xs: TraversableOnce[A]): this.type = {
    xs.foreach { this += _ }
    this
  }

  override def +=(elem: A): this.type = {
    if (size < maxSize) {
      underlying.offer(elem)
    } else {
      maybeReplaceLowest(elem)
    }
    this
  }

  def poll(): A = {
    underlying.poll()
  }

  override def +=(elem1: A, elem2: A, elems: A*): this.type = {
    this += elem1 += elem2 ++= elems
  }

  override def clear() { underlying.clear() }

  private def maybeReplaceLowest(a: A): Boolean = {
    val head = underlying.peek()
    if (head != null && ord.gt(a, head)) {
      underlying.poll()
      underlying.offer(a)
    } else {
      false
    }
  }
}
```

