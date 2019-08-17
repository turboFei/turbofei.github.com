---
layout: post
category: bigData
tagline: "Linux Ops"
summary: 作为一个大数据平台从业人员，会操作线上服务器是必备的技能，因此必须要会一些常见的Linux运维命令。
tags: [linux, Ops]
---
{% include JB/setup %}
目录
* toc
{:toc}
{{ page.summary }} 本文记录一下最近用到的一些线上操作命令。

### 前言

由于之前对线上操作不多，而最近开始接触线上操作，因此也在这个过程中学习到了一些Linux命令。

- telnet
- ss  & somaxconn
- netstat
- top
- sysctl & systemctl

### telnet

在linux系统中，通常需要判断一些网络问题。比如某个节点是否是联通的，可以使用Ping 命令。

```bash
ping hostname
```

而如果要判断一个host的port是否互通就需要使用telnet命令。

```bash
telnet host port
```

嗯，感觉这个就够了...

### ss & somaxconn

说实话这个命令之前都没听过。

ss 是socket statistics的简称。用于查看socket相关的统计信息。

```bash
Usage: ss [ OPTIONS ]
       ss [ OPTIONS ] [ FILTER ]
   -h, --help           this message      #显示帮助菜单
   -V, --version        output version information      #输出版本信息
   -n, --numeric        don't resolve service names    #不解析服务名
   -r, --resolve       resolve host names   #解析主机名
   -a, --all            display all sockets     #显示所有的套接字
   -l, --listening      display listening sockets   #显示监听状态的socket
   -o, --options       show timer information   #显示计时器信息
   -e, --extended      show detailed socket information #展示详细的socket信息
   -m, --memory        show socket memory usage #展示socket的内存使用
   -p, --processes      show process using socket   #展示使用socket的进程
   -i, --info           show internal TCP information   #展示tcp内部信息
   -s, --summary        show socket usage summary   #展示socket使用汇总

   -4, --ipv4          display only IP version 4 sockets    #只显示ipv4的sockets
   -6, --ipv6          display only IP version 6 sockets    #只显示ipv6的sockets
   -0, --packet display PACKET sockets  #显示包经过的网络接口
   -t, --tcp            display only TCP sockets    #显示tcp套接字
   -u, --udp            display only UDP sockets    #显示udp套接字
   -d, --dccp           display only DCCP sockets   #显示dccp套接字
   -w, --raw            display only RAW sockets    #显示raw套接字
   -x, --unix           display only Unix domain sockets    #显示unix套接字
   -f, --family=FAMILY display sockets of type FAMILY   #显示指定类型的套接字

   -A, --query=QUERY, --socket=QUERY    #查看某种类型
       QUERY := {all|inet|tcp|udp|raw|unix|packet|netlink}[,QUERY]

   -D, --diag=FILE      Dump raw information about TCP sockets to FILE  #将关于TCP套接字的原始信息转储到文件中
   -F, --filter=FILE   read filter information from FILE #使用此参数指定的过滤规则文件，过滤某种状态的连接
       FILTER := [ state TCP-STATE ] [ EXPRESSION ]
```

这里讲一下 -n 参数，如果不加 `-n`参数，那么会显示服务名，而如果使用`-n`，那么则不解析服务名。示例如下。

```bash
[fwang12@fwang-dev1-3474144 ~]$ ss -t
State       Recv-Q Send-Q  Local Address:Port                   Peer Address:Port
ESTAB       0      168    10.194.228.167:ssh                   10.242.101.88:56151
ESTAB       0      0      10.194.228.167:38570                 10.232.128.78:ldaps
ESTAB       0      0      10.194.228.167:54564                10.169.164.189:4505
[fwang12@fwang-dev1-3474144 ~]$ ss -tn
State       Recv-Q Send-Q    Local Address:Port                   Peer Address:Port
ESTAB       0      168      10.194.228.167:22                    10.242.101.88:56151
ESTAB       0      0        10.194.228.167:38570                 10.232.128.78:636
ESTAB       0      0        10.194.228.167:54564                10.169.164.189:4505
```

这样我们就可以看到服务所使用的端口号，这样就可以通过一些端口号来查看其socket 信息。例如我们熟知的ExternalShuffleService就是默认以7337作为端口，那么就可以使用`ss -ln |grep 7337`来查看其socket信息。

#### Recv-Q & Send-Q

1. 当 client 通过 connect 向 server 发出 SYN 包时，client 会维护一个 socket 等待队列，而 server 会维护一个 SYN 队列
2. 此时进入半链接的状态，如果 socket 等待队列满了，server 则会丢弃，而 client 也会由此返回 connection time out；只要是 client 没有收到 SYN+ACK，3s 之后，client 会再次发送，如果依然没有收到，9s 之后会继续发送
3. 半连接 syn 队列的长度为 max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog)  决定
4. 当 server 收到 client 的 SYN 包后，会返回 SYN, ACK 的包加以确认，client 的 TCP 协议栈会唤醒 socket 等待队列，发出 connect 调用
5. client 返回 ACK 的包后，server 会进入一个新的叫 accept 的队列，该队列的长度为 min(backlog, somaxconn)，默认情况下，`somaxconn` 的值为 128，表示最多有 129 的 ESTAB 的连接等待 accept()，而 backlog 的值则由 [int listen(int sockfd, int backlog)](http://http//linux.die.net/man/2/listen) 中的第二个参数指定，listen 里面的 backlog 的含义请看这里。需要注意的是，[一些 Linux 的发型版本可能存在对 somaxcon 错误 truncating 方式](http://serverfault.com/questions/518862/testifying-rasing-net-core-somaxconn-can-make-a-difference)。
6. 当 accept 队列满了之后，即使 client 继续向 server 发送 ACK 的包，也会不被相应，此时，server 通过 /proc/sys/net/ipv4/tcp_abort_on_overflow 来决定如何返回，0 表示直接丢丢弃该 ACK，1 表示发送 RST 通知 client；相应的，client 则会分别返回 read timeout 或者 connection reset by peer。上面说的只是些理论，如果服务器不及时的调用 accept()，当 queue 满了之后，服务器并不会按照理论所述，不再对 SYN 进行应答，返回 ETIMEDOUT。根据[这篇](http://www.douban.com/note/178129553/)文档的描述，实际情况并非如此，服务器会随机的忽略收到的 SYN，建立起来的连接数可以无限的增加，只不过客户端会遇到延时以及超时的情况。

 

可以看到，整个 TCP stack 有如下的两个 queue:
1. 一个是 half open(syn queue) queue(max(tcp_max_syn_backlog, 64))，用来保存 SYN_SENT 以及 SYN_RECV 的信息。
2. 另外一个是 accept queue(min(somaxconn, backlog))，保存 ESTAB 的状态，但是调用 accept()。

#### somaxconn

前面提到，client返回ack之后，server进入一个新的叫accept的队列，队列长度为min(backlog, somaxconn)。

somaxconn定义了系统中每一个端口最大的监听队列的长度,这是个全局的参数,默认值为128.限制了每个端口接收新tcp连接侦听队列的大小。针对线上一些服务，比如说Spark 的External Shuffle Service，默认的128太小，必须设置很大才能满足生产需求。

而somaxconn 是在文件`/etc/sysctl.conf`中。

保存之后使用`sysctl -p`立即生效。

### netstat

netstat的作用其实与 ss 类似。

```bash
[fwang12@fwang-dev1-3474144 ~]$ netstat -h
usage: netstat [-vWeenNcCF] [<Af>] -r         netstat {-V|--version|-h|--help}
       netstat [-vWnNcaeol] [<Socket> ...]
       netstat { [-vWeenNac] -I[<Iface>] | [-veenNac] -i | [-cnNe] -M | -s [-6tuw] } [delay]

        -r, --route              display routing table
        -I, --interfaces=<Iface> display interface table for <Iface>
        -i, --interfaces         display interface table
        -g, --groups             display multicast group memberships
        -s, --statistics         display networking statistics (like SNMP)
        -M, --masquerade         display masqueraded connections

        -v, --verbose            be verbose
        -W, --wide               don't truncate IP addresses
        -n, --numeric            don't resolve names
        --numeric-hosts          don't resolve host names
        --numeric-ports          don't resolve port names
        --numeric-users          don't resolve user names
        -N, --symbolic           resolve hardware names
        -e, --extend             display other/more information
        -p, --programs           display PID/Program name for sockets
        -o, --timers             display timers
        -c, --continuous         continuous listing

        -l, --listening          display listening server sockets
        -a, --all                display all sockets (default: connected)
        -F, --fib                display Forwarding Information Base (default)
        -C, --cache              display routing cache instead of FIB
        -Z, --context            display SELinux security context for sockets

  <Socket>={-t|--tcp} {-u|--udp} {-U|--udplite} {-S|--sctp} {-w|--raw}
           {-x|--unix} --ax25 --ipx --netrom
  <AF>=Use '-6|-4' or '-A <af>' or '--<af>'; default: inet
  List of possible address families (which support routing):
    inet (DARPA Internet) inet6 (IPv6) ax25 (AMPR AX.25)
    netrom (AMPR NET/ROM) ipx (Novell IPX) ddp (Appletalk DDP)
    x25 (CCITT X.25)
```

例如可以使用`netstat -antlp`列出所有非解析服务名字的tcp监听状态的server sockets，并打出pid.

### top

输入top打印如下。

```bash
top - 11:05:18 up 9 days, 15:59,  1 user,  load average: 0.00, 0.01, 0.05
Tasks: 184 total,   1 running, 183 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32778616 total, 31754528 free,   350816 used,   673272 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 31972344 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9198 fwang12   20   0  172408   2424   1632 R   0.3  0.0   0:00.01 top
```

其中显示的列信息，可以在进入top之后，按F进入编辑列，使用空格决定是否显示列信息。

```
* PID     = Process Id
* USER    = Effective User Name
* PR      = Priority
* NI      = Nice Value  //PR(new) = PR(old) + NI
* VIRT    = Virtual Image (KiB)
* RES     = Resident Size (KiB)
* SHR     = Shared Memory (KiB)
* S       = Process Status
* %CPU    = CPU Usage
* %MEM    = Memory Usage (RES)
* TIME+   = CPU Time, hundredths
* COMMAND = Command Name/Line
  PPID    = Parent Process pid
  UID     = Effective User Id
  RUID    = Real User Id
  RUSER   = Real User Name
  SUID    = Saved User Id
  SUSER   = Saved User Name
  GID     = Group Id
  GROUP   = Group Name
  PGRP    = Process Group Id
  TTY     = Controlling Tty
  TPGID   = Tty Process Grp Id
  SID     = Session Id
  nTH     = Number of Threads
  P       = Last Used Cpu (SMP)
  TIME    = CPU Time
  SWAP    = Swapped Size (KiB)
  CODE    = Code Size (KiB)
  DATA    = Data+Stack (KiB)
  nMaj    = Major Page Faults
  nMin    = Minor Page Faults
  nDRT    = Dirty Pages Count
  WCHAN   = Sleeping in Function
  Flags   = Task Flags <sched.h>
  CGROUPS = Control Groups
  SUPGIDS = Supp Groups IDs
  SUPGRPS = Supp Groups Names
  TGID    = Thread Group Id
  ENVIRON = Environment vars
  vMj     = Major Faults delta
  vMn     = Minor Faults delta
  USED    = Res+Swap Size (KiB)
  nsIPC   = IPC namespace Inode
  nsMNT   = MNT namespace Inode
  nsNET   = NET namespace Inode
  nsPID   = PID namespace Inode
  nsUSER  = USER namespace Inode
  nsUTS   = UTS namespace Inode
```

```bash
fwang12@fwang-dev1-3474144 ~]$ top -h
  procps-ng version 3.3.10
Usage:
  top -hv | -bcHiOSs -d secs -n max -u|U user -p pid(s) -o field -w [cols]
```

可以指定用户，PID等信息.

进入top之后输入`?`可以查看帮助，查看可以使用什么命令。

```bash
Help for Interactive Commands - procps-ng version 3.3.10
Window 1:Def: Cumulative mode Off.  System: Delay 3.0 secs; Secure mode Off.

  Z,B,E,e   Global: 'Z' colors; 'B' bold; 'E'/'e' summary/task memory scale
  l,t,m     Toggle Summary: 'l' load avg; 't' task/cpu stats; 'm' memory info
  0,1,2,3,I Toggle: '0' zeros; '1/2/3' cpus or numa node views; 'I' Irix mode
  f,F,X     Fields: 'f'/'F' add/remove/order/sort; 'X' increase fixed-width

  L,&,<,> . Locate: 'L'/'&' find/again; Move sort column: '<'/'>' left/right
  R,H,V,J . Toggle: 'R' Sort; 'H' Threads; 'V' Forest view; 'J' Num justify
  c,i,S,j . Toggle: 'c' Cmd name/line; 'i' Idle; 'S' Time; 'j' Str justify
  x,y     . Toggle highlights: 'x' sort field; 'y' running tasks
  z,b     . Toggle: 'z' color/mono; 'b' bold/reverse (only if 'x' or 'y')
  u,U,o,O . Filter by: 'u'/'U' effective/any user; 'o'/'O' other criteria
  n,#,^O  . Set: 'n'/'#' max tasks displayed; Show: Ctrl+'O' other filter(s)
  C,...   . Toggle scroll coordinates msg for: up,down,left,right,home,end

  k,r       Manipulate tasks: 'k' kill; 'r' renice
  d or s    Set update interval
  W,Y       Write configuration file 'W'; Inspect other output 'Y'
  q         Quit
          ( commands shown with '.' require a visible task display window )
Press 'h' or '?' for help with Windows,
Type 'q' or <Esc> to continue
```

常用可以使用`1`查看CPU信息。



### sysctl & systemctl

sysctl用于运行时配置内核参数，这些参数位于/proc/sys目录下。sysctl配置与显示在/proc/sys目录中的内核参数．可以用sysctl来设置或重新设置联网功能，如IP转发、IP碎片去除以及源路由检查等。用户只需要编辑/etc/sysctl.conf文件，即可手工或自动执行由sysctl控制的功能。

systemctl用于管理系统服务，systemd(system dameon),是相对于service和chkconfig的新命令.

|       daemon命令       |         systemctl命令         |   说明   |
| :--------------------: | :---------------------------: | :------: |
|  service [服务] start  |  systemctl start [unit type]  | 启动服务 |
|  service [服务] stop   |  systemctl stop [unit type]   | 停止服务 |
| service [服务] restart | systemctl restart [unit type] | 重启服务 |

|      daemon命令      |         systemctl命令         |         说明         |
| :------------------: | :---------------------------: | :------------------: |
| chkconfig [服务] on  | systemctl enable [unit type]  |   设置服务开机启动   |
| chkconfig [服务] off | systemctl disable [unit type] | 设备服务禁止开机启动 |

```bash
[fwang12@fwang-dev1-3474144 ~]$ sysctl -h

Usage:
 sysctl [options] [variable[=value] ...]

Options:
  -a, --all            display all variables
  -A                   alias of -a
  -X                   alias of -a
      --deprecated     include deprecated parameters to listing
  -b, --binary         print value without new line
  -e, --ignore         ignore unknown variables errors
  -N, --names          print variable names without values
  -n, --values         print only values of a variables
  -p, --load[=<file>]  read values from file
  -f                   alias of -p
      --system         read values from all system directories
  -r, --pattern <expression>
                       select setting that match expression
  -q, --quiet          do not echo variable set
  -w, --write          enable writing a value to variable
  -o                   does nothing
  -x                   does nothing
  -d                   alias of -h

 -h, --help     display this help and exit
 -V, --version  output version information and exit
```



常用的几个， `sysctl -w`可以直接写入一个值到sys内核参数中。

Sysctl -p 可以立即读取/etc/sysctl.conf中的参数并生效。

sysctl -a可以列出所有的内核参数。

### To Be Continued



### References

[ss与 Recv-Q Send-Q](https://www.cnblogs.com/leezhxing/p/5329786.html)

[linux Top Usage](https://blog.csdn.net/yjclsx/article/details/81508455)