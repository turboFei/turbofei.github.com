---
layout: post
category: hadoop
tagline: ""
summary: 关于Hdfs的租约机制
tags: [hadoop, hdfs]
---
{% include JB/setup %}
目录

* toc
{:toc}
{{ page.summary }}

### 背景

最近有需求，需要了解一下hdfs 的lease机制。

### Hdfs lease机制

本章节代码基于hadoop-2.7.4分支。

hdfs是一个分布式文件系统，分布式意味着高并发，经常会面临文件同时访问的问题，如果保证合理有序的访问这些文件呢？答案就是lease机制，lease顾名思义租约，就是服务端给客户端一个临时的ticket，无此ticket以及ticket过期将不会再允许对此文件进行某些操作。

Hdfs管理租约的类叫LeaseManager，里面维护了三个有序集合(TreeMap/TreeSet)，相当于三个不同的索引，用于不同的操作进行查询。

- 租约持有者与租约的有序映射
- 文件路径与租约的映射
- 按照时间排序的租约集合

此外里面有两个重要的字段:

```java
  private long softLimit = HdfsConstants.LEASE_SOFTLIMIT_PERIOD; // 60s
  private long hardLimit = HdfsConstants.LEASE_HARDLIMIT_PERIOD; // 1hour
```

这里的softLimit 和hardLimit与linux中 soft limit和hard limit不同，linux中limit代表的是打开文件的上限，而这里的limit其实是一个周期。

LeaseManager中大多数方法都是 get/remove/set方法以及renew操作，这些方法都是用于与dfsClient进行交互，是一些被动的操作，LeaseManager中一个最重要的操作是释放过期的租约，因为往往会有异常情况发生，DfsClient没有优雅的发送释放自己租约的请求而就异常关闭了。这时候，如果租约得不到释放， 将会影响到其他dfsClient对其持有文件的访问。

LeaseManager中的做法是创建一个守护监控线程，定时的来监控和释放租约，其Monitor实现类如下:

```java
  class Monitor implements Runnable {
    final String name = getClass().getSimpleName();

    /** Check leases periodically. */
    @Override
    public void run() {
      for(; shouldRunMonitor && fsnamesystem.isRunning(); ) {
        boolean needSync = false;
        try {
          fsnamesystem.writeLockInterruptibly();
          try {
            // 当前模式不是安全模式
            if (!fsnamesystem.isInSafeMode()) {
              // 调用checkLease，并且返回sortedLeases 是否需要sync
              needSync = checkLeases();
            }
          } finally {
            fsnamesystem.writeUnlock("leaseManager");
            // lease reassignments should to be sync'ed.
            if (needSync) {
              // 如果需要sync sortedLeases，则进行同步
              fsnamesystem.getEditLog().logSync();
            }
          }
          // 周期为 2000 ms
          Thread.sleep(HdfsServerConstants.NAMENODE_LEASE_RECHECK_INTERVAL);
        } catch(InterruptedException ie) {
          if (LOG.isDebugEnabled()) {
            LOG.debug(name + " is interrupted", ie);
          }
        }
      }
    }
  }
```

代码中显示，其会根据NAMENODE_LEASE_RECHECK_INTERVAL(2000ms)定时的调用`checkLeases`方法。

代码可以分为三部分：

1. 由于sortedLease是按照时间对租约排序，第一部分就是取出最老的lease 用于检查
2. 第二部分是一个while循环检查这个lease
3. 第三部分是在检查完之后，检查sortedLeases是否需要进行sync，返回检查结果，这个结果用于后续在monitor线程中对这个sortedLease进行同步。

代码很清晰，核心逻辑是第二部分，我们看下第二部分的while循环代码。

可以看到一上来就是先看当前lease 是否超出了hardLimit的限制，如果没有，那么直接退出check操作。

如果超出了hardLimit的限制：

1. 获取当前lease持有的文件路径
2. 如果其持有文件路径，则这些文件路径从租约中删除
3. 将此租约删除。

```java
   while(leaseToCheck != null) {
      if (!leaseToCheck.expiredHardLimit()) {
        break;
      }

      LOG.info(leaseToCheck + " has expired hard limit");

      final List<String> removing = new ArrayList<String>();
      // need to create a copy of the oldest lease paths, because 
      // internalReleaseLease() removes paths corresponding to empty files,
      // i.e. it needs to modify the collection being iterated over
      // causing ConcurrentModificationException
      String[] leasePaths = new String[leaseToCheck.getPaths().size()];
      leaseToCheck.getPaths().toArray(leasePaths);
      for(String p : leasePaths) {
        try {
          INodesInPath iip = fsnamesystem.getFSDirectory().getINodesInPath(p,
              true);
          boolean completed = fsnamesystem.internalReleaseLease(leaseToCheck, p,
              iip, HdfsServerConstants.NAMENODE_LEASE_HOLDER);
          if (LOG.isDebugEnabled()) {
            if (completed) {
              LOG.debug("Lease recovery for " + p + " is complete. File closed.");
            } else {
              LOG.debug("Started block recovery " + p + " lease " + leaseToCheck);
            }
          }
          // If a lease recovery happened, we need to sync later.
          if (!needSync && !completed) {
            needSync = true;
          }
        } catch (IOException e) {
          LOG.error("Cannot release the path " + p + " in the lease "
              + leaseToCheck, e);
          removing.add(p);
        }
      }

      for(String p : removing) {
        removeLease(leaseToCheck, p);
      }
      leaseToCheck = sortedLeases.higher(leaseToCheck);
    }
```

#### Soft limit

上面我们看到在checkLease方法中，使用到了hardLimit，如果租约超时hardLimit，那么就将该lease相关清除掉。

那么softLimit的作用呢？

```java
    /** @return true if the Soft Limit Timer has expired */
    public boolean expiredSoftLimit() {
      return monotonicNow() - lastUpdate > softLimit;
    }
```

通过查看代码，我们看到有一个`expiredSoftLimit`方法， 其调用是发生在`FSNameSystem`中，对应方法为`recoverLeaseInternal`，调用部分如下，如果当前的持有者已经在上个softLimit周期没有刷新这个lease,

那么调用`internalReleaseLease`， 顾名思义，释放internalLease, 其注释为 `Move a file that is being written to be immutable.`, 也就是说把一个正在被写的文件变为可改变的状态。

```java
        // If the original holder has not renewed in the last SOFTLIMIT 
        // period, then start lease recovery.
        //
        if (lease.expiredSoftLimit()) {
          LOG.info("startFile: recover " + lease + ", src=" + src + " client "
              + clientName);
          if (internalReleaseLease(lease, src, iip, null)) {
            return true;
          } else {
            throw new RecoveryInProgressException(
                op.getExceptionMessage(src, holder, clientMachine,
                    "lease recovery is in progress. Try again later."));
          }
        }
```



关于这方面的上下文如下:

- 首先DFSClient是获取lease的主体，当其生成之后会被添加到对应的LeaseRenewer的dfsClient列表里面，周期性的进行刷新lease，周期是固定的softLimitPeriod/2,也就是30s
- 所以一般情况下这个lease不会超过softLimit，除非

  - LeaseRenewer发生严重的GC，无法renew //可能性极小

  - DFSClient异常退出，已经从LeaseRenewer中移出，但是租约还未超过HardLimit所以租约还未移除 
- 变为可被改变的状态意味着另外一个dfsClient可以对其进行append操作。

##### 基于Soft Limit的Lock？

那么是否可以利用soft limit做一个简单的锁？

例如在app1 中create 一个lock文件，在应用正常结束时会清理掉这个lock文件，当然如果其异常退出，是不会清理掉这个lock文件的。

另外一个app2尝试去探测这个lock文件，探测方法为如果这个lock存在则尝试append这个lock文件，如果append成功，则代表可以获得这个lock。

如果不能append成功，则有两种可能:

- 这个lock正在被另一个app持有
- 另一个app异常退出，但是距其异常退出的间隔还未到达softLimitPeriod

所以这个锁，就相当于app正常退出会释放， 如果app异常退出，超过softLimitPeriod(60s)也会自动释放。

### 附录-Soft Limit 单元测试

在pom.xml文件中添加:

```xml
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-minicluster</artifactId>
      <version>${hadoop.version}</version>
      <scope>test</scope>
    </dependency>
```
```scala
import scala.util.{Failure, Success, Try}

import org.apache.hadoop.fs.Path
import org.apache.hadoop.hdfs.{DistributedFileSystem, HdfsConfiguration, MiniDFSCluster}
import org.apache.hadoop.hdfs.protocol.HdfsConstants
import org.scalatest.Assertions.intercept

object SoftLimitLockSuite {
  val hdfsConf = new HdfsConfiguration
  hdfsConf.set("fs.hdfs.impl.disable.cache", "true")
  val cluster = new MiniDFSCluster.Builder(hdfsConf).build()
  cluster.waitClusterUp()
  val fs = cluster.getFileSystem

  def main(args: Array[String]): Unit = {
    try {
      val lock = new Path(fs.getHomeDirectory, "LOCK")
      // app 正常结束
      val app1 = new AppLockThread(lock, abort = false)
      app1.start()
      Thread.sleep(1000)
      app1.interrupt()
      app1.join()
      // fs 可以append 这个lock文件成功
      fs.append(lock).close()

      // app 异常退出
      val app2 = new AppLockThread(lock, abort = true)
      app2.start()
      Thread.sleep(1000)
      app2.interrupt()
      app2.join()
      intercept[Exception](fs.append(lock).close())
      // 等待一个soft limit 周期
      Thread.sleep(HdfsConstants.LEASE_SOFTLIMIT_PERIOD)
      // fs 可以append成功
      fs.append(lock).close()
    } finally {
      fs.close()
      cluster.shutdown()
    }
  }

  /**
   * 模拟一个app 线程, abort 参数代表是否正常退出
   */
  class AppLockThread(lock: Path, abort: Boolean = false) extends Thread {
    override def run(): Unit = {
      var dfs: Option[DistributedFileSystem] = None
      try {
        dfs = Some(cluster.getFileSystem)
        dfs.get.create(lock)
        while (true) {
          Thread.sleep(HdfsConstants.LEASE_SOFTLIMIT_PERIOD)
        }
      } catch {
        case _: InterruptedException =>
          try {
            // Here is an reflection implementation of DistributedFileSystem.close()
            dfs.foreach(_.getClient.closeOutputStreams(abort))
            invokeSuperMethod(dfs.get, "close")
          } finally {
            dfs.foreach(_.getClient.close())
          }
      }
    }
  }

  /**
   * Invoke a super method of an object via reflection.
   */
  private def invokeSuperMethod(o: Any, name: String): Any = {
    Try {
      val method = try {
        o.getClass.getSuperclass.getDeclaredMethod(name)
      } catch {
        case e: NoSuchMethodException =>
          o.getClass.getMethod(name)
      }
      method.setAccessible(true)
      method.invoke(o)
    } match {
      case Success(value) => value
      case Failure(e) => throw e
    }
  }
}
```




