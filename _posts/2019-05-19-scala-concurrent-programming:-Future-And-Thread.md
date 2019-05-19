---
layout: post
category: coding
tagline: ""
summary: 简单写下scala中的Future以及对Thread的认识
tags: [coding,essay,scala,concurrent]
---
{% include JB/setup %}
目录

* toc
{:toc}
### Background ###
{{ page.summary }}

java和scala中都有Future，那么这两个Future有什么不同呢？Thread是怎么样的，它的状态是如何变化的呢？一些操作比如sleep会涉及到锁么？

### Java Future

java中Future类中方法很简单，也很少.

```java
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
```

比较常用的就是get，可以设置超时时间。下面是使用java Future的一个场景，通常是使用线程池submit Callable。记得线程池要放在finally模块关闭。

```scala
  def testJavaFuture(): Unit = {
    val call = new Callable[Long] {
      override def call(): Long = {
        Thread.sleep(10000)
        123L
      }
    }
    val pool = Executors.newFixedThreadPool(1)
    try {
      val f = pool.submit(call)
      println(f.isDone)
      println(f.get(6, TimeUnit.SECONDS))
    } finally {
      pool.shutdown()
    }
  }
```

### Scala Future

相较于java的Future，scala的Future中方法很丰富。而且scala中伴生对象的apply方法使得创建一个Future非常方便.例如:

```scala
   val f: Future[String] = Future {
     " future!"
   }
```

介绍其中几个方法，用法写在注释中，println结果也在注释中。

```scala
import scala.collection.generic.CanBuildFrom
import scala.concurrent.duration.Duration
import scala.concurrent.{Await, ExecutionContext, Future}
import scala.util.Success
import scala.util.control.NonFatal

object TestScalaFuture {
  implicit val executionContext = ExecutionContext.global
  def createIntFuture(value: Int): Future[Int] = {
    Future {
      value
    }
  }

  def createIntFutureWithFailure(value: Int): Future[Int] = {
    Future {
      1 / 0
      value
    }
  }

  /**
   * recover方法是在Future发生异常时的处理。
   */
  def testRecover(): Unit = {
    val f1 = createIntFutureWithFailure(1).recover {
      case e: Exception =>
        -1
    }
    println(Await.result(f1, Duration("10s")))  // -1
  }

  /**
   * 将两个Future zip到一起，这样就只需要使用一个Await就可以等结果。
   */
  def testZip(): Unit = {
    val f1 = createIntFuture(1)
    val f2 = createIntFuture(2)
    println(Await.result(f1.zip(f2), Duration("10s"))) // (1,2)
  }

  /**
   * 功能类似于zip，是处理更多个，需要指定CanBuildFrom。
   */
  def testSequence(): Unit = {
    val f1 = createIntFuture(1)
    val f2 = createIntFuture(2)
    val f3 = createIntFuture(3)
    implicit val cbf = implicitly[CanBuildFrom[Iterator[Future[Int]], Int, Iterator[Int]]]
    val r = Await.result(Future.sequence(Iterator(f1, f2, f3)), Duration("10s"))
    r.foreach(v => print(v + "\t")) // 1 2 3
  }

  /**
   * 这里的map， flatMap等操作是对返回值进行的操作，也是lazy的。
   * 这里的andThen不会改变返回值。
   * Transform是对返回值进行的操作，以及对异常的转换。
   */
  def testMisc(): Unit = {
    val f1 = createIntFuture(1).map(_ * 7).map(_ + 1)
    println(Await.result(f1, Duration("10s") )) // 8
    val f2 = createIntFuture(2).andThen {
      case Success(v) if v ==2 =>
        println("the value is 2") // 这里只是执行一些操作，但是不会改变Future的返回值
      case _ =>
    }
    println(Await.result(f2, Duration("10s"))) // 2
    val f3 = createIntFuture(3).transform(v => "str:" + v, throwable => throwable match {
      case NonFatal(throwable) => new Exception(throwable)
      case _ => throwable
    })
    println(Await.result(f3, Duration("10s"))) // str:3
  }
}
```

### Thread

Thread类实现了Runnable，是一个特殊的Runnable类。

一个线程代表一个程序的执行。jvm允许一个应用并发执行多个线程。每个线程都有一个优先级，优先级高的线程相对于优先级低的线程，更容易被执行。每个线程都可能被标记为一个守护(daemon)线程。当一个线程创建了一个新的线程，这个新的线程的优先级初始化为和创建它的线程一样。

当一个JVM 启动时，通常是只有一个非守护线程。JVM会一直运行直到:

- exit方法被调用，并且允许exit。
- 所有非守护线程都已经结束，可以是正常返回结束也可以是异常结束。

有两种方法生成一个新的执行线程。

一种是继承Thread类，overwrite run方法，然后start。

另一种是继承`Runnable`类，实现run方法，然后 `new Thread(runnable).start.`

线程的优先级分为1，5，10。1是所允许的最低优先级，5是默认分配，10是能够拥有的最高优先级。

Thread类里面提供了一些静态工具方法. **Deprecated**的方法不再列出.

```java
 		// 得到当前线程	
		public static native Thread currentThread();
    public static native void yield();
    public static native void sleep(long millis) throws InterruptedException;
    public synchronized void start();
    public void run();
    public void interrupt();
    public boolean isInterrupted();
    public final native boolean isAlive();
    public final void setPriority(int newPriority);
    public final int getPriority()；
    public final synchronized void setName(String name);
		public final String getName();
		public final ThreadGroup getThreadGroup();
		public static int activeCount();
		public static int enumerate(Thread tarray[]);
		public final synchronized void join(long millis);
		public final void setDaemon(boolean on);
		public final boolean isDaemon();
		public final void checkAccess();
		public ClassLoader getContextClassLoader();
		public void setContextClassLoader(ClassLoader cl);
		public static native boolean holdsLock(Object obj);
		public StackTraceElement[] getStackTrace();
		public static Map<Thread, StackTraceElement[]> getAllStackTraces();
		public long getId();
		public State getState();
		static void processQueue(ReferenceQueue<Class<?>> queue,
                             ConcurrentMap<? extends
                             WeakReference<Class<?>>, ?> map);

```

#### Thread状态

首先，thread的五种状态.

- NEW 线程被创建，还没start
- RUNNABLE  在JVM上运行，可能在等操作系统的资源，比如时间片
- BLOCKED 阻塞状态，等待lock来进入同步代码块
- WAITING  
  - Object.wait 没有指定timeout
  - 因为Thread.join 无timeout等待
  - LockSupport.park()无限期等待
- TIMED_WAITING  有timeout的WAITING
  - Object.wait(long)
  - Thread.join(long)
  - LockSupport.parkNanos LockSupport.parkUntil
- TREMINATED  线程退出

#### Thread 方法解析

**yield**

yield方法是给调度器一个hint表明自己自愿放弃当前的处理器，调度器可以忽略这个hint。这个方法不推荐，很少使用，可以用于避免cpu过度利用，但是使用之前要做好详细的分析与benchmark。spark项目中没有用到过yield.

**sleep**

sleep方法比较常用，这是将当前线程放弃执行，休眠一段时间，但是sleep不会放弃自己得到的monitor.

sleep(0)的意思代表是，大家所有线程重新抢占一下处理器。

**threadGroup**

 在创建thread时候可以传入threadGroup参数。如果没有传入group，如果该线程指定了securityManager，则去问securityManager拿group，最终是拿currentThread的group，如果没指定securityManager，则和父线程一组。

**start**

开始运行线程，jvm调用run()方法。一个线程只能启动一次，否则会报IllegalThreadStateException。

**run**

实现的Runnable的run方法，用于让jvm调用

**interrupt**

如果是线程自己interrupt自己，是允许的，否则，需要securityManager进行checkAccess，可能会抛出SecurityException。

interrupt之后会加一个标志位interrupted.

如果此时该线程被 wait, join, sleep, 那么这个interrupted标志位会被清除然后抛出InterruptedException.

如果线程被`java.nio.channels.InterruptibleChannel`的I/O操作阻塞，那么这个channel将被关闭，然后set interrupted标志位，这个线程会收到一个`ClosedByInterruptException`.

如果线程被`java.nio.channels.Selector`阻塞，那么将会设置interrupted标志位，并马上从selection操作返回。

如果上述情况都没发生，那么这个线程设置interrup状态标志位.

如果线程已经dead，interrupt操作没丝毫作用，也不会出错。

**isInterrupted**

查看是否被设为interrupted.

**setPriority**

改变线程的优先级。首先会由securityManager进行校验，校验失败抛SecurityException. 校验成功，则取设置的值和当前threadGroup的最大权限中的较小值，作为线程的优先级。

**getPriority**

获得线程优先级.

**setName, getName**

设置线程名字，获取线程名字

**getThreadGroup**

获得线程的threadGroup

**activeCount**

获得当前线程的threadGroup以及subGroup中的线程数.由于线程在动态变化，因此只是一个估计值，主要是用于debug以及monitoring.

**join(time)**

等待线程结束，如果join(0)代表一直等待。如果该线程被其他thread interrupt，那么这个线程的interrupted标志位被清除，然后抛出`InterruptedException`.

**dumpStack**

打印当前线程的栈，只用于**debug**.

**setDaemon(isDaemon)**

设为守护线程或者用户线程。JVM会在所有用户线程都挂掉之后退出。

必须在线程启动之前设置，如果线程已经是alive，会抛`IllegalThreadStateException`.同样也会检查SecurityManager当前线程是否有权限去设置。

**isDaemon**

是否是守护线程

**checkAccess**

检查当前线程有没有权限去修改这个线程。

**getContextClassLoader, setContextClassLoader**

classLoader是用于加载classes和resources。默认的classLoader是父线程的classLoader。原始的线程classLoader通常是设置为应用的classLoader。如果classLoader不为空， 且securityManager不为空，将会进行权限校验。**权限校验几乎伴随thread的每个操作**，后面就不再提了.

**holdsLock(Object obj)**

线程是否持有某个monitor.

**getStackTrace, getAllStackTraces**

一个是打印当前线程的stack，一个是所有线程的stack，用户debug

**getId**

获得线程Id

**getState**

获得线程状态

**UncaughtExceptionHandler**

是一个接口，用于当线程由于一些未捕获的异常而导致终止时的处理。

里面只有一个方法.

```java
 void uncaughtException(Thread t, Throwable e);
```

**get(set)DefaultUncaughtExceptionHandler, get(set)UncaughtExceptionHandler**

关于设置UncaughtExceptionHandler。ThreadGroup是UncaughtExceptionHandler的一个实现类，如果当前thread没有设置UncaughtExceptionHandler，那么返回threadGroup。