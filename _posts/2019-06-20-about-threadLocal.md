---
layout: post
category: coding
tagline: ""
summary: 本文讲ThreadLocal的使用场景，注意事项以及源码实现。
tags: [java,jvm,concurrent]
---
{% include JB/setup %}
目录

* toc
{:toc}
{{ page.summary }}

### 什么是ThreadLocal

ThreadLocal，顾名思义，是线程本地的，也就是说非多线程共享，是为了解决线程并发时，变量共享的问题。但是在使用时需要注意内存泄露，脏数据等问题。

那么什么时候需要用到ThreadLocal呢？此处举几个例子.

#### Case 1

例如我们定义了一个类，代表一个Student，然后场景是一个考场，每个学生用一个线程进行表示，而每个学生有一个做卷子的进度变量，这个变量必须是对应一个学生，也就是一个线程。这个时候，我们需要定义一个`ThreadLocal`类型的state变量来表示每个学生目前的状态，这样每个学生的状态不会互相干扰.

例如下面的这段代码，在同一个线程里面的Student 的ThreadLocal值是一样的。

```java
import java.util.concurrent.atomic.AtomicInteger;

public class TestThreadLocal {
  public static void main(String[] args) {
    Student s1 = new Student();
    System.out.println("Current Thread value:" + s1.getState());// Current Thread value:1
    Student s2 = new Student();
    System.out.println("Current Thread value:" + s2.getState()); // Current Thread value:1
    s1.removeState();
    
    new Thread(() -> {
      Student s3 = new Student();
      System.out.println("New Thread value:" + s3.getState()); // New Thread value:2
      s3.removeState();
    }).start();
  }

}

class Student {
  private static AtomicInteger al = new AtomicInteger();
  private static ThreadLocal<Integer> state = ThreadLocal.withInitial(() -> al.incrementAndGet());
  public int getState() {
    return state.get();
  }
  public void removeState() {
    state.remove();
  }
}
```



#### Case 2

对于一些非线程安全的类，例如SimpleDateFormat，定义为static，会有数据同步风险。SimpleDateFormat内部有一个Calendar对象，在日期转字符串或者字符串转日期的过程中，多线程共享时有非常高的概率产生错误，推荐的方式是使用ThreadLocal，让每个线程单独拥有这个对象.

```java
  private static final ThreadLocal<DateFormat> DATA_FORMAT_THREADLOCAL = 
          ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-mm-dd"));
```

#### Case 3

在父线程和子线程之间传递变量，可以使用ThreadLocal.

ThreadLocal有一个子类是InheritThreadLocal. 使用方式如下:

```java
import java.util.Date;

class Helper {
  public static ThreadLocal<Long> time = new InheritableThreadLocal<Long>() {
    @Override
    protected Long initialValue() {
      return new Date().getTime();
    }
  };
}

public class TestInheritThreadLocal {
  public static void main(String[] args) {
    System.out.println("ParentThreadTime:" + Helper.time.get()); // ParentThreadTime:1561222644968
    new Thread(() -> {
      System.out.println("CurrentThreadTime:" + new Date().getTime()); // CurrentThreadTime:1561222645061
      System.out.println("InheritTime:" + Helper.time.get()); // InheritTime:1561222644968
    }).start();
  }
}
```

### 如何正确使用ThreadLocal

在使用ThreadLocal时候要注意避免产生脏数据和内存泄露。这两个问题通常是在线程池的线程中使用ThreadLocal引发的，因为线程池中有线程复用和内存常驻两个特点.

**脏数据**

线程复用可能会产生脏数据，由于线程池会重用Thread对象，那么与Thread绑定的类的静态属性ThreadLocal变量也会被重用。如果在实现的线程`run()`方法体重不显示的调用remove()清理与线程相关的ThreadLocal信息，那么倘若下一个线程不调用set()设置初始值，就可能get到重用的线程信息，包括ThreadLocal所关联的线程对象的Value值.

**内存泄露**

在源码注释中提示使用static关键字来修饰ThreadLocal。在此场景下，寄希望于ThreadLocal对象失去引用后，触发弱引用机制来回收不显示，因此在线程执行完毕之后，需要执行remove()方法，不然其对应ThreadLocal持有的值不会被释放。

### 源码实现

#### 引用类型

首先介绍一下Java中的四种引用类型.

- 强引用。 例如:`Object obj = new Object()`。只要对象具有可达性，就不能被回收。
- 软引用。在即将OOM之前，即使有可达性，也可回收。
- 弱引用。在下一次YGC时会被回收。
- 虚引用。一种极弱的引用关系，定义完成后，就无法通过该引用获取指向的对象。为一个对象设置虚引用的唯一目的是希望能在这个对象回收时收到一个系统通知。虚引用必须与引用队列联合使用，当垃圾会收拾，如果发现存在虚引用，就会在回收对象内存前，把这个虚引用加入与之关联的引用队列中。**极少使用。**

此处给出软引用和弱引用的使用示例.

```java
import java.lang.ref.SoftReference;
import java.lang.ref.WeakReference;

public class TestReference {
  public static void soft() {
    Integer value = new Integer(10086);
    SoftReference<Integer> soft = new SoftReference<>(value);
    value = null; // 对象设为空，解除强引用劫持.
  }
  
  public static void weak() {
    Integer value = new Integer(10086);
    WeakReference<Integer> soft = new WeakReference<>(value);
    value = null; // 对象设为空，解除强引用劫持.
  }
}
```

ThreadLocal的设计中使用了WeakReference， JDK中设计的愿意是在ThreadLocal对象消失后，线程对象再持有这个ThreadLocal对象是没有任何意义的，应该进行回收，从而避免内存泄露，这种设计的出发点很好，但弱引用的设计增加了对ThreadLocal 和Thread体系的理解难度。

#### ThreadLocalMap

ThreadLocal有个静态内部类叫做`ThreadLocalMap`, 它还有一个静态内部类叫`Entry`, 而Entry是弱引用类型.

ThreadLocal与ThreadLocalMap有三组对应的方法，`get()`,`set()`,`remove()`。在ThreadLocal中只是做校验和判断，最终的实现会落在ThreadLocalMap中。

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```



Entry继承了WeakReference, 只有一个value成员变量，其Key是ThreadLocal对象。

**每一个`Thread` 中有一个`ThreadLocalMap`**(因为一个Thread中可能有多个ThreadLocal，所以需要一个Map保存所有的ThreadLocal变量，而key就是ThreadLocal变量，value是对应的值).

#### ThreadLocal.initialValue()

虽然说每个Thread有一个ThreadLocalMap，那么这个localMap是如何创建的呢？

首先，ThreadLocal有三个方法，其中get方法就是为了去获取其对应的值，如果没有调用get，那么这个ThreadLocal毫无价值。

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

首先，它会去获得这个Thread对应的localMap，如果这个localMap非空，而且可以在这个localMap中找到，那么直接返回找到的值。

如果这个localMap为空，或者说这个localMap没有对应的值。

- 如果localMap为空。那么会创建createMap方法，创建对应的ThreadLocalMap,并且初始化的kv是(当前线程，initialValue创建的值).
- 如果localMap非空，那么就像这个localMap中添加值.

#### InheritThreadLocal

这是ThreadLocal的一个子类，上面我们也提到了其用法。其override了ThreadLocal的几个方法，去掉注释如下.

```Java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    protected T childValue(T parentValue) {
        return parentValue;
    }
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
```

只override了三个方法，其中 childValue 是ThreadLocal不支持的，其调用是只在`createInheritedMap`时进行调用。而其他两个getMap 和 createMap是在从Thread中获取localMap时的改写，以及在创建localMap的改写。

那么对应的 线程本地变量是如何传递进来的呢？

看下面的方法调用. 

```java
    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
        init(g, target, name, stackSize, null, true);
    }
		// 此处 inheritThreadLocals = true
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
  		......
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
   		......
    }
```

可以看到，默认使用`new  Thread()`创建线程就会继承父线程的ThreadLocals.

#### 内存角度

##### JMM

首先介绍下Java的内存模型。

![](/imgs/thread-local/jmm.png)

从抽象的角度看，JMM定义了线程和主内存之间的抽象关系:线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存存储了该线程以读/写共享变量的副本。

**本地内存是JMM的一个抽象概念，并不真实存在。**它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化。

##### TLAB(Thread Local Allocation Buffer)

TLAB代表线程本地变量分配缓冲区，这是Eden内部的一个region, 是被划分为一个Thread的区域,属于非线程共享区域。换句话说，只有一个线程可以在一个TLAB里面分配对象。每个Thread都有对应的TLAB.

所以针对TLAB里面的对象，不需要设置同步操作。

### Reference

[What is Thread Local Allocation Buffer](https://dzone.com/articles/thread-local-allocation-buffers)