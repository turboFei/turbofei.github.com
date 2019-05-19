---
layout: post
category: coding
tagline: "About concurrent programming"
summary: 关于并发编程的一些总结与思考，包括promise, forkJoinPool, and etc.
tags: [concurrent, scala, future]
---
{% include JB/setup %}
目录

* toc
{:toc}
### Background ###

{{ page.summary }}
在使用scala进行并发编程时，常用的一个就是Promise，Promise和Future是相关的。在生产中，Promise往往和Thread结合一起用，在一个线程中去执行Promise.trySuccess.提到线程就不得不提线程池，而ForkJoinPool是一个特殊的线程池，它比较适合计算密集型的场景。

### Promise

Promise是scala中独有的，java中没有。中文意思就是承诺，它可以在获得承诺的value时成功结束，也可以在遇到异常时失败。
一个Promise只能承诺一次，如果它已经完成承诺，或者失败，或者超时，再对它进行调用就会抛出IllegalStateException。在promize中有很多方法,如下
```scala
tryComplete, tryCompleteWith, complete，
tryFailureWith, success, failure, trySuccess, tryFailure
isCompleted， future
```

这些方法有所不同，例如complete系列(包括tryComplete， tryCompleteWith)是可以返回值，也可以是异常的。

而failure系列只能是异常，而success系列智能是返回value。因此，complete系列更像是一个对后两者的并集。

在使用中，我们可以按照自己的需求去选择这些方法。

比如我们可以直接使用complete系列将所有系列包容，也可以使用trySuccess 然后在捕获异常之后，将异常直接给failure方法。

isComplete是用于判断Promise是否已经完成，而future是一个包含Promise结果的Future。

我们通常将Promise和Await一起用。如果Promise在执行中出现了异常，Await是可以将其抛出，而如果Promise没有在规定时间内返回，那么将会抛出TimeoutException.

例子如下:

```scala
import scala.concurrent.duration.Duration
import scala.concurrent.{Await, ExecutionContext, Future, Promise}
import scala.util.Try
import scala.util.control.NonFatal

object TestPromise {

  def method(): Long = {
    /**
     * 此处是为了校验超时异常
     * 以及执行中异常.
     */
    // Thread.sleep(13000)
    // throw new Exception("this is an exception")
    123L
  }

  def trySuccess(promisedLong: Promise[Long]): Unit = {
    new Thread("test promise") {
      override def run(): Unit = {
        try {
          promisedLong.trySuccess {
            method()
          }
        } catch {
          case e: Exception if NonFatal(e) =>
            promisedLong.failure(e)
//            promisedLong.tryFailure(e)
        }
      }
    }.start()
  }

  def tryComplete(promisedLong: Promise[Long]): Unit = {
    new Thread("test promise") {
      override def run(): Unit = {
        try {
          promisedLong.tryComplete {
            Try {
              method()
            }
          }
        }
      }
    }.start()
  }

  def tryCompleteWith(promisedLong: Promise[Long]): Unit = {
    implicit val global=  ExecutionContext.global
    new Thread("test promise") {
      override def run(): Unit = {
        try {
          promisedLong.tryCompleteWith {
            Future {
              method()
            }
          }
        }
      }
    }.start()
  }

  def main(args: Array[String]): Unit = {
    val promisedLong = Promise[Long]
    tryComplete(promisedLong)
    try {
      val r = Await.result(promisedLong.future, Duration("12s"))
      println(r)
    } catch {
      case e: Exception =>
        e.printStackTrace()
    }
  }
}
```

#### Nonfatal & ControlThrowable

上面的例子中提到了Nonfatal，这在生产中是一个常用的类。

顾名思义，Nonfatal代表非致命的，方法体也很短.

可以看出致命的错误有，虚拟机Error，ThreadDeath，中断异常，链接Error，以及`ControlThrowable`。除了这几种，其他都是非致命的。

```scala
object NonFatal {
   /**
    * Returns true if the provided `Throwable` is to be considered non-fatal, or false if it is to be considered fatal
    */
   def apply(t: Throwable): Boolean = t match {
     // VirtualMachineError includes OutOfMemoryError and other fatal errors
     case _: VirtualMachineError | _: ThreadDeath | _: InterruptedException | _: LinkageError | _: ControlThrowable => false
     case _ => true
   }
  /**
   * Returns Some(t) if NonFatal(t) == true, otherwise None
   */
  def unapply(t: Throwable): Option[Throwable] = if (apply(t)) Some(t) else None
}
```

一般在程序中，都是对Nonfatal进行catch处理， 而致命的就不catch了。

那么什么是`ControlThrowable`呢？

```scala
trait ControlThrowable extends Throwable with NoStackTrace
```

我们可以看到它继承了NoStackTrace类，也就是说这个异常栈是不能打印的。ControlThrowable代表这个Throwable是被放在控制流中。因为这个异常时作为控制流异常（比如BreadControl等等）， 因此发生这种异常，需要propagate，而不能catch，不过这一切都被封装在了Nonfatal，我们在编程中只要判断Nonfatal就可以。



#### Try 

关于Try，它和try catch中的try不同，它代表执行一个程序块，通常和match，以及Success， Failure一起使用。

```scala
  def testTry(): Unit = {
    Try {
//      throw new Exception("this is an exception")
      123L
    } match {
      case Success(v) =>
        println(v)
      case Failure(e) =>
        println(e.getMessage)
    }
  }
```

### ForkJoinPool

**其实看源码中注释是理解源码最好的方式。**
ForkJoinPool是一个用于运行ForkJoinTask的线程池. ForkJoinPool和其他的ExecutorService不同，他有一套work-窃取机制，每个线程都可以尝试去find和执行pool中或者其他task提交的任务，这样就可以更加高效，因为每个线程的执行都不是限制死的，如果它空闲了就可以去窃取其他forkJointask的任务，这样也减少了线程的上下文切换，所以对于计算密集型的任务效率会很高，所以，如果你的任务是计算密集型，不妨试一下ForkJoinPool。相当于大家同心协力去把pool中的所有task运行完，这样避免了因为倾斜带来的低效。

asyncMode默认是false，当设为true，这更适合于事件类型的任务，从来不会有join。

下面是一个计算从1到n和的一个程序，采用了普通线程池和ForkJoinPool来实现，实验证明，forkjoinpool性能领先很大。

```scala
import java.util.ArrayList;
import java.util.concurrent.*;

public class TestForkJoinPool {
    public static void main(String[] args) {
        long current = System.currentTimeMillis();
        System.out.println(sumWithExecutorService(1, 100000000, 1000));
        long cost = (System.currentTimeMillis() - current);
        System.out.println("Cost Time is:" + cost + "ms!");
        current = System.currentTimeMillis();
        System.out.println(executeWithForkJoinPool(1, 100000000, 100000));
        cost = (System.currentTimeMillis() - current);
        System.out.println("ForkJoinPoll Cost Time is:" + cost + "ms!");
    }

    public static int sumWithExecutorService(int from, int to, int threadNum) {
        ExecutorService pool = Executors.newFixedThreadPool(threadNum);
        int sum = 0;
        try {
            /**
             * 这里，如果前面不进行强制类型转换，那么除了之后就是一个int
             * 就没有必要取ceil 了，切记。
             */
            int step = (int)Math.ceil(((double)(to - from + 1))/threadNum);
            ArrayList<Future<Integer>> futures = new ArrayList<>();
            for (int i = 0; i< threadNum; i++) {
                if ((from + i * step) <= Math.min(to, from + (i+1) * step -1)) {
                    futures.add(pool.submit(new SumTask(
                            from + i * step, Math.min(to, from + (i+1) * step -1))));
                }
            }
            for (Future<Integer> future: futures) {
                sum += future.get();
            }
        } catch (Exception ignore) {

        } finally {
            pool.shutdown();
        }
        return sum;
    }

    private static class SumTask implements Callable<Integer> {
        private int from;
        private int to;
        public SumTask(int from, int to) {
            this.from = from;
            this.to = to;
        }
        @Override
        public Integer call() throws Exception {
            int sum = 0;
            for (int i =from; i<= to; i++) {
                sum += i;
            }
            return sum;
        }
    }

    public static int executeWithForkJoinPool(int from, int to, int thresHold) {
        ForkJoinPool forkJoinPool = new ForkJoinPool(1);
        try {
            return forkJoinPool.invoke(new ForkJoinSumTask(from, to, thresHold));
        } finally {
            forkJoinPool.shutdown();
        }
    }

    private static class ForkJoinSumTask extends RecursiveTask<Integer> {
        private int from;
        private int to;
        private int thresHold;
        public ForkJoinSumTask(int from, int to, int thresHold) {
            this.from = from;
            this.to = to;
            this.thresHold = thresHold;
        }

        @Override
        protected Integer compute() {
            if (to - from < thresHold) {
                int sum = 0;
                for (int i =from; i<=to;i++) {
                    sum += i;
                }
                return sum;
            } else {
                int mid = (to + from) / 2;
                ForkJoinSumTask leftTask = new ForkJoinSumTask(from, mid, thresHold);
                ForkJoinSumTask rightTask = new ForkJoinSumTask(mid +1, to, thresHold);
                leftTask.fork();
                rightTask.fork();
                return leftTask.join() + rightTask.join();
            }
        }
    }
}

```

#### work-steamling机制

ForkJoinPool中的fork和join是unix中创建线程的方法，在Unix中使用fork可以创建一个子进程，然后join是让父进程等待子进程执行完毕才进行。但是在ForkJoinPool中并不是每次fork都要创建一个子线程，我们可以设置poolSize，规定线程数目的上限。

ForkJoinPool中的每个线程会维护一个工作队列.这个队列是双端队列，在每次执行自己队列的任务时会尝试随机窃取一个task，窃取对应队列的顺序是FIFO，而执行自己队列中的任务在同步模式下是LIFO。可以看到fork函数是将task放置在队列的尾部。

```scala
    public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```

而join操作呢? 

1. 判断该任务是否已经完成，如果完成返回，否则2
2. 这个任务是自己的工作队列中，如果在，则执行，等待其完成。
3. 如果不在自己的工作队列中，则已经被小偷窃取。
4. 找到小偷，窃取他队列中的任务，FIFO方式窃取，帮助他早日完成任务。
5. 如果小偷已经做完自己的任务，自己在等待被其他小偷窃取走的任务时，帮助他。
6. 递归5，直到返回结果。

```scala
    private int doJoin() {
        int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
        return (s = status) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).
            tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :
            externalAwaitDone();
    }

    /**
     * Blocks a non-worker-thread until completion.
     * @return status upon completion
     */
    private int externalAwaitDone() {
        int s = ((this instanceof CountedCompleter) ? // try helping
                 ForkJoinPool.common.externalHelpComplete(
                     (CountedCompleter<?>)this, 0) :
                 ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);
        if (s >= 0 && (s = status) >= 0) {
            boolean interrupted = false;
            do {
                if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                    synchronized (this) {
                        if (status >= 0) {
                            try {
                                wait(0L);
                            } catch (InterruptedException ie) {
                                interrupted = true;
                            }
                        }
                        else
                            notifyAll();
                    }
                }
            } while ((s = status) >= 0);
            if (interrupted)
                Thread.currentThread().interrupt();
        }
        return s;
    }
```
