---
layout: post
category: coding
tagline: "jps, jstack, jstat, jmap, jinfo."
summary: 简单介绍jvm的相关工具，例如 jps, jstack, jstat, jmap, jinfo.
tags: [jvm, java]
---
{% include JB/setup %}
目录

* toc
{:toc}
{{ page.summary }}

### 前言

jvm提供了一些工具用来用来查看运行java程序的一些状态。

#### Jps

首先就是jps，用来查看运行的java线程。

```java
hadoop@spark6:~$ jps
131649 KyuubiSubmit
39087 KyuubiSubmit
144342 Jps
136470 KyuubiSubmit
```

在知道java线程的信息之后，可以使用ps -ef查看一些详细的启动信息.

其中 -e 代表所有线程同A, -f代表 full-format, 包括command-line.

```java
hadoop@spark6:~$ ps -ef|grep KyuubiSubmit
hadoop    39087      1 99 05:29 ?        13:15:44 /home/hadoop/java-current/bin/java -cp /home/hadoop/kyuubi_hz_cluster_10/kyuubi-0.6.2-bin-spark-2.1.3/lib/kyuubi-server-0.6.2.jar:/home/hadoop/kyuubi_hz_cluster_10/spark-2.3.2-bin-ne-0.1.0/conf/:/home/hadoop/kyuubi_hz_cluster_10/spark-2.3.2-bin-ne-0.1.0/jars/*:/home/hadoop/hadoop-client4cluster10/etc/hadoop/ -Xmx164g -XX:+PrintFlagsFinal -XX:+UnlockDiagnosticVMOptions -XX:ParGCCardsPerStrideChunk=4096 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSConcurrentMTEnabled -XX:CMSInitiatingOccupancyFraction=70 -XX:+UseCMSInitiatingOccupancyOnly -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:+UseCondCardMark -XX:PermSize=1024m -XX:MaxPermSize=1024m -XX:MaxDirectMemorySize=8192m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./logs -XX:OnOutOfMemoryError=kill -9 %p -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -Xloggc:./logs/kyuubi-server-gc-%t.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=50 -XX:GCLogFileSize=5M -XX:NewRatio=3 -Dio.netty.noPreferDirect=true -Dio.netty.recycler.maxCapacity=0 -Dio.netty.noUnsafe=true org.apache.spark.deploy.KyuubiSubmit --class yaooqinn.kyuubi.server.KyuubiServer /home/hadoop/kyuubi_hz_cluster_10/kyuubi-0.6.2-bin-spark-2.1.3/lib/kyuubi-server-0.6.2.jar
```

如果我们使用 ps -aux 命令，可以看到更多的信息。

USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

VSZ 进程使用的虚拟內存量(KB)；

RSS 该进程占用的固定內存量(KB)(驻留中页的数量)；

TTY 该进程在哪个终端上进行(登陆者的终端位置)，若与终端无关，則显示(?)。

START 该进程被启动时间；

TIME 该进程实际使用CPU运行时间；

COMMAND 命令的名称和参数；

其他信息参考[PS -aux 命令详解](https://www.cnblogs.com/dion-90/articles/9048627.html)

```java
hadoop@spark6:~$ ps -aux|grep KyuubiSubmit
hadoop    39087  179  3.2 197330936 8695348 ?   Sl   05:29 828:00 /home/hadoop/java-current/bin/java -cp /home/hadoop/kyuubi_hz_cluster_10/kyuubi-0.6.2-bin-spark-2.1.3/lib/kyuubi-server-0.6.2.jar:/home/hadoop/kyuubi_hz_cluster_10/spark-2.3.2-bin-ne-0.1.0/conf/:/home/hadoop/kyuubi_hz_cluster_10/spark-2.3.2-bin-ne-0.1.0/jars/*:/home/hadoop/hadoop-client4cluster10/etc/hadoop/ -Xmx164g -XX:+PrintFlagsFinal -XX:+UnlockDiagnosticVMOptions -XX:ParGCCardsPerStrideChunk=4096 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSConcurrentMTEnabled -XX:CMSInitiatingOccupancyFraction=70 -XX:+UseCMSInitiatingOccupancyOnly -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:+UseCondCardMark -XX:PermSize=1024m -XX:MaxPermSize=1024m -XX:MaxDirectMemorySize=8192m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./logs -XX:OnOutOfMemoryError=kill -9 %p -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -Xloggc:./logs/kyuubi-server-gc-%t.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=50 -XX:GCLogFileSize=5M -XX:NewRatio=3 -Dio.netty.noPreferDirect=true -Dio.netty.recycler.maxCapacity=0 -Dio.netty.noUnsafe=true org.apache.spark.deploy.KyuubiSubmit --class yaooqinn.kyuubi.server.KyuubiServer /home/hadoop/kyuubi_hz_cluster_10/kyuubi-0.6.2-bin-spark-2.1.3/lib/kyuubi-server-0.6.2.jar
```

####  Jstack

Jstack命令可以用来打印当前java 线程的线程栈，比如如果java程序长时间无响应，可以使用Jstack命令查看当前线程是否卡在了哪里，看是否存在死锁等情况。

jstack命令生成的thread dump信息包含了JVM中所有存活的线程.

在dump中，线程一般存在如下几种状态：
1、RUNNABLE，线程处于执行中
2、BLOCKED，线程被阻塞
3、WAITING，线程正在等待

下面是一个示例.

可以看到一个java程序里面有很多线程。一部分是JVM内部的功能线程，另一部分是用户自己的线程，可以参考[JVM内部线程](http://ifeve.com/jvm-thread/)。

```java
$ jstack 29047
2019-06-29 16:51:28
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.111-b14 mixed mode):

"Attach Listener" #13 daemon prio=9 os_prio=31 tid=0x00007fc168804800 nid=0xe13 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"NGSession 1: (idle)" #12 prio=5 os_prio=31 tid=0x00007fc168146000 nid=0x4603 in Object.wait() [0x000070000ec8b000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x00000007aae55e68> (a java.lang.Object)
	at java.lang.Object.wait(Object.java:502)
	at com.martiansoftware.nailgun.NGSession.nextSocket(NGSession.java:167)
	- locked <0x00000007aae55e68> (a java.lang.Object)
	at com.martiansoftware.nailgun.NGSession.run(NGSession.java:186)

"DestroyJavaVM" #11 prio=5 os_prio=31 tid=0x00007fc168112800 nid=0x2803 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"NGServer(localhost/127.0.0.1, 65319,27633241-7c9d-4458-9831-527f37863f1d)" #9 prio=5 os_prio=31 tid=0x00007fc169095800 nid=0x4703 runnable [0x000070000eb88000]
   java.lang.Thread.State: RUNNABLE
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
	at com.martiansoftware.nailgun.NGServer.run(NGServer.java:418)
	at java.lang.Thread.run(Thread.java:745)

"Service Thread" #8 daemon prio=9 os_prio=31 tid=0x00007fc168002000 nid=0x3a03 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C1 CompilerThread2" #7 daemon prio=9 os_prio=31 tid=0x00007fc16883f000 nid=0x3803 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" #6 daemon prio=9 os_prio=31 tid=0x00007fc169816800 nid=0x3603 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #5 daemon prio=9 os_prio=31 tid=0x00007fc16900f000 nid=0x3503 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" #4 daemon prio=9 os_prio=31 tid=0x00007fc16803b800 nid=0x3403 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" #3 daemon prio=8 os_prio=31 tid=0x00007fc168026800 nid=0x2f03 in Object.wait() [0x000070000e473000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x00000007aab08e98> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x00000007aab08e98> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

"Reference Handler" #2 daemon prio=10 os_prio=31 tid=0x00007fc169020000 nid=0x2e03 in Object.wait() [0x000070000e370000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x00000007aab06b40> (a java.lang.ref.Reference$Lock)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
	- locked <0x00000007aab06b40> (a java.lang.ref.Reference$Lock)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

"VM Thread" os_prio=31 tid=0x00007fc169001000 nid=0x2c03 runnable

"GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007fc168004800 nid=0x1d07 runnable

"GC task thread#1 (ParallelGC)" os_prio=31 tid=0x00007fc168005800 nid=0x1e03 runnable

"GC task thread#2 (ParallelGC)" os_prio=31 tid=0x00007fc169002000 nid=0x5403 runnable

"GC task thread#3 (ParallelGC)" os_prio=31 tid=0x00007fc169003000 nid=0x5303 runnable

"VM Periodic Task Thread" os_prio=31 tid=0x00007fc16804d000 nid=0x3c03 waiting on condition

JNI global references: 47
```

#### Jstat

jstat用法如下:

```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```

- -option 是可选选项
  - -class 
  
    跟class 加载，占用消耗有关的状态.
  
    ```java
    hadoop@spark6:~$ jstat -class 131649
    Loaded  Bytes  Unloaded  Bytes     Time
     21038 42844.6      904  1394.4      28.32
    ```
  
  - -compiler
  
    应该是 即时编译有关吧.
  
    ```java
    hadoop@spark6:~$ jstat -compiler 131649
    Compiled Failed Invalid   Time   FailedType FailedMethod
       39298      3       0   216.39          1 org/apache/spark/ContextCleaner$$anonfun$org$apache$spark$ContextCleaner$$keepCleaning$1 apply$mcV$sp
    ```
  
  - -gc 垃圾回收统计
  
    ```java
    hadoop@spark6:~$ jstat -gc 131649
    Warning: Unresolved Symbol: sun.gc.generation.2.space.0.capacity substituted NaN
    Warning: Unresolved Symbol: sun.gc.generation.2.space.0.used substituted NaN
     S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
    186240.0 186240.0 186240.0  0.0   1490240.0 326860.8 5587220.0  3700631.8    �      �     32818 2921.766 11684 4006.977 6928.743
    ```
  
    
  
  - -gccapacity 堆内存统计
  
  - -gccause
  
  - -gcnew
  
  - -gcnewcapacity
  
  - -gcold
  
  - -gcoldcapacity
  
  - -gcpermcapacity 
  
  - -gcutil 总结垃圾回收统计
  
  -  -printcompilation
  
- -t 是用于显示timeStamp，示例如下。

  - ```
    hadoop@spark6:~$ jstat -gc 131649
    Warning: Unresolved Symbol: sun.gc.generation.2.space.0.capacity substituted NaN
    Warning: Unresolved Symbol: sun.gc.generation.2.space.0.used substituted NaN
     S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
    186240.0 186240.0 186240.0  0.0   1490240.0 1341107.0 5587220.0  3388251.2    �      �     32748 2914.187 11656 3990.174 6904.362
    hadoop@spark6:~$ jstat -gc -t 131649
    Warning: Unresolved Symbol: sun.gc.generation.2.space.0.capacity substituted NaN
    Warning: Unresolved Symbol: sun.gc.generation.2.space.0.used substituted NaN
    Timestamp        S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
           247190.2 186240.0 186240.0 186240.0  0.0   1490240.0 1341242.8 5587220.0  3102583.1    �      �     32748 2914.187 11657 3990.174 6904.362
    ```

- -h 是每行之间的样本数，用于求平均值吧

- vmid 是java进程Id

- interval 采样间距，单位为毫秒

- 采样次数

#### Jmap

用法如下:

```java
hadoop@spark6:~$ jmap
Usage:
    jmap [option] <pid>
        (to connect to running process)
    jmap [option] <executable <core>
        (to connect to a core file)
    jmap [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    <none>               to print same info as Solaris pmap
    -heap                to print java heap summary
    -histo[:live]        to print histogram of java object heap; if the "live"
                         suboption is specified, only count live objects
    -permstat            to print permanent generation statistics
    -finalizerinfo       to print information on objects awaiting finalization
    -dump:<dump-options> to dump java heap in hprof binary format
                         dump-options:
                           live         dump only live objects; if not specified,
                                        all objects in the heap are dumped.
                           format=b     binary format
                           file=<file>  dump heap to <file>
                         Example: jmap -dump:live,format=b,file=heap.bin <pid>
    -F                   force. Use with -dump:<dump-options> <pid> or -histo
                         to force a heap dump or histogram when <pid> does not
                         respond. The "live" suboption is not supported
                         in this mode.
    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system
```

​	首先介绍一下core dump。当程序运行的过程中异常终止或崩溃，操作系统会将程序当时的内存状态记录下来，保存在一个文件中，这种行为就叫做Core Dump。我们可以认为 core dump 是“内存快照”，但实际上，除了内存信息之外，还有些关键的程序运行状态也会同时 dump 下来，例如寄存器信息（包括程序指针、栈指针等）、内存管理信息、其他处理器和操作系统状态和信息。core dump 对于编程人员诊断和调试程序是非常有帮助的，因为对于有些程序错误是很难重现的，例如指针异常，而 core dump 文件可以再现程序出错时的情景。在linux中可以使用 `ulimit -c`查看目前是否打开core dump 功能，如果显示为0则代表未打开，可以使用`ulimit -c unlimited`打开core dump功能，当core dump是打开时，才会在程序崩溃时保存内存快照。

##### mat

在实际生产中，我们通常使用 `jmap -dump:live,format=b,file=heap.bin <pid>`命令来将core  dump到文件中，然后使用 mat(eclipse memory analyzer)来分析这个dump文件，会生成一些html文件，然后将这些文件下载下来，点击其index.html来查看分析结果。

mat的下载地址为:https://www.eclipse.org/mat/downloads.php。

解压之后是一个`mat`文件夹，进入这个文件夹.

```java
 ./ParseHeapDump.sh jmap.info  org.eclipse.mat.api:suspects org.eclipse.mat.api:overview org.eclipse.mat.api:top_components

```

结果会生产如下三个zip文件，很小可以直接拷贝到本机.

```
jmap_Leak_Suspects.zip
jmap_System_Overview.zip
jmap_Top_Components.zip
```

之后就可以解压查看其对应的index.html.

#### Jinfo(Java Configuration Info)

使用方法如下:

```java
hadoop@spark6:~$ jinfo
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message
```

看到上面可以更改一些参数，那么哪些参数是可以动态更改?

JVM官方文档说明如下，也就是说，标记为manageable的参数或者通过`com.sun.management.HotSpotDiagnosticMXBean`这个类的接口得到；

```
Flags marked as manageable are dynamically writeable through the JDK management interface (com.sun.management.HotSpotDiagnosticMXBean API) and also through JConsole.
```

通过manageable方法更加方便，命令如下:

```java
hadoop@spark6:~$ java -XX:+PrintFlagsInitial | grep manageable
     intx CMSAbortablePrecleanWaitMillis            = 100             {manageable}
     intx CMSWaitDuration                           = 2000            {manageable}
     bool HeapDumpAfterFullGC                       = false           {manageable}
     bool HeapDumpBeforeFullGC                      = false           {manageable}
     bool HeapDumpOnOutOfMemoryError                = false           {manageable}
    ccstr HeapDumpPath                              =                 {manageable}
    uintx MaxHeapFreeRatio                          = 70              {manageable}
    uintx MinHeapFreeRatio                          = 40              {manageable}
     bool PrintClassHistogram                       = false           {manageable}
     bool PrintClassHistogramAfterFullGC            = false           {manageable}
     bool PrintClassHistogramBeforeFullGC           = false           {manageable}
     bool PrintConcurrentLocks                      = false           {manageable}
     bool PrintGC                                   = false           {manageable}
     bool PrintGCDateStamps                         = false           {manageable}
     bool PrintGCDetails                            = false           {manageable}
     bool PrintGCTimeStamps                         = false           {manageable}
```

所以只有这几个参数是可以通过 `jinfo -flag  [+|-]<name>  pid`或者`jinfo -flag <name>=<value> pid`动态更改的.

### References

[JVM内部运行线程介绍](http://ifeve.com/jvm-thread/)

[PS -aux 命令详解](https://www.cnblogs.com/dion-90/articles/9048627.html)

[Jmap使用](https://www.jianshu.com/p/a4ad53179df3)

[mat使用方法]([http://moheqionglin.com/site/blogs/84/detail.html](http://moheqionglin.com/site/blogs/84/detail.html))

[jinfo](https://www.jianshu.com/p/c321d0808a1b)