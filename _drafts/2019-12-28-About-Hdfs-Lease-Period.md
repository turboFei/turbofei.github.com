---
layout: post
category: hadoop
tagline: "Soft Limit and Hard Limit"
summary: 关于hdfs的lease的周期
tags: [hadoop, hdfs]
---
{% include JB/setup %}
目录

* toc
{:toc}
{{ page.summary }}

### 背景

在Spark中有一个比较比较严重的数据质量问题，就是向一张表中insert数据时，如果有多个操作并发执行，或者前面的操作执行时异常退出，working目录没有清理干净，后面操作在复用这个working目录时，会造成数据重复。

关于该问题的详细描述，笔者在前面的文章[关于Spark数据计算结果异常的场景分析#场景B](../../../../spark/2019/09/30/关于Spark数据计算结果异常的场景分析#场景b)中详细描述过这个问题的原因。也在社区提过一个[PR](https://github.com/apache/spark/pull/25863), 这个PR目前合入了eBay的Spark分支(当然做了一些更新，后续更新社区PR).

关于PR中提出的方案，在此处做简要描述。

假设一张表 ta, 其有若干个分区键, p1,p2...pn (n>=0)。

我们根据其插入时候指定的分区键的kv值(specifiedPartitionKV)，创建一个位于表目录下的staging目录。

假设其指定了k个partition kv值，当然这k个kv值时要按顺序指定的。

其目录格式为 `$tbl_path/.spark-staging-${specifiedPartitionKV.size}/p1=sv1/p2=sv2/.../pk=svk/applicationId`（加入一级applicationID目录，用于标志正在进行该写入操作的applicationID）

我们将这个目录格式当做一个标志，代表有操作在向这个partition路径进行写入操作。

在每次写入操作前，我们会进行一次冲突检查。

例如我要向表ta进行 p1=v1/p2=v2/.../pk=vk的写入。我要检查以下staging 目录是否存在

- .spark-staging-0/
- .spark-staging-1/p1=v1
- .spark-staging-2/p1=v1/p2=v2
- ...
- .spark-staging-k/p1=v1/p2=v2/.../pk=vk
- .spark-staging-${k+1}/p1=v1/p2=v2/.../pk=vk
- ...
- .spark-staging-n/p1=v1/p2=v2/.../pk=vk

只要上面的任何一个staging目录存在，我们都会认为存在写入冲突。

上面我们也提到过加了一级applicationID目录，这样我们在发现写入冲突的时候可以报出异常，然后将这个applicationID抛出，并且抛出该目录最后的修改时间。当发现写入冲突时有两种可能性：

1. applicationID对应的操作仍在运行中，我们需要等待其完成之后再进行写入操作
2. applicationID对应的操作异常退出，没有清理掉staging目录，需要进行手动清除

这个时候，就需要用户通过applicationID和目录修改时间，进行手动的判定该目录是否存在，为了安全，我们是建议用户手动清理。

该功能上线之后，很好的保证了数据质量，但是由于其需要人工介入， 可能会干扰到一些自动化运行的任务的下游任务阻塞。

当然上线之后我们也一直在思考，如果更好的去自动化的识别和清除异常退出的staging目录。

经过探讨和探索之后，我们决定利用hdfs的lease机制来实现。

### Hdfs lease机制

本章节代码基于hadoop-2.7.4分支。

hdfs是一个分布式文件系统，分布式意味着高并发，经常会面临文件同时访问的问题，如果保证合理有序的访问这些文件呢？答案就是lease机制，lease顾名思义租约，就是服务端给客户端一个临时的许可证，无此许可证以及许可证过期将不会再允许对此文件进行操作。

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

LeaseManager的Structure如下:

![](/imgs/hdfs-lease/leasemanager.png)

我们看到大多数方法都是 get/remove/set方法以及renew操作，这些方法都是用于与dfsClient进行交互。都是一些被动的操作，其实LeaseManager中一个最重要的操作是释放过期的租约，因为往往会有异常情况发生，DfsClient没有优雅掉发送释放自己租约的请求而就异常关闭了。这时候，如果租约得不到释放， 将会影响到其他dfsClient对其持有文件的访问。

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
3. 第三部分是在检查完之后，看这个用于检查的lease是否和检查之后的sortedLease的第一个lease一样，返回检查结果，这个结果用于后续在monitor线程中对这个sortedLease进行同步。

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

通过查看代码，我们看到有一个`expiredSoftLimit`方法， 其调用是发生在`FSNameSystem`中，对应方法为`recoverLeaseInternal`，调用部分如下，如果当前的持有者已经在上个softLimit周期没有刷新这个lease。

那么尝试调用internalReleaseLease， 顾名思义，释放internalRelese.

其注释为 `Move a file that is being written to be immutable.`, 也就是说把这个文件变为一个可以被写，被改变的状态。

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
- 变为写改变意味着另外一个dfsClient可以对其进行append操作。

