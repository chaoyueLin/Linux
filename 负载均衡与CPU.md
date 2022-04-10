# CPU性能
## 负载均衡
 平均负载,使用uptime

    watch -d uptime
    02:34:03 up 2 days, 20:14,  1 user,  load average: 0.63, 0.83, 0.88

* 运行状态 Running, Runable 
* 不可中断状态 uninterruptible 

平均负载其实就是平均活跃进程数比如当平均负载为 2 时，意味着什么呢？
* 在只有 2 个 CPU 的系统上，意味着所有的 CPU 都刚好被完全占用。
* 在 4 个 CPU 的系统上，意味着 CPU 有 50% 的空闲。
* 而在只有 1 个 CPU 的系统中，则意味着有一半的进程竞争不到 CPU。

平均负载不仅包括了正在使用 CPU 的进程，还包括等待 CPU 和等待 I/O 的进程。而 CPU 使用率，是单位时间内 CPU 繁忙情况的统计，跟平均负载并不一定完全对应。比如：
* CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
* I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；
* 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。

## cup使用率
* 正在使用cpu,使用mpstat
 
        # -P ALL 表示监控所有CPU，后面数字5表示间隔5秒后输出一组数据
        $ mpstat -P ALL 5
        Linux 4.15.0 (ubuntu) 09/22/18 _x86_64_ (2 CPU)
        13:30:06     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
        13:30:11     all   50.05    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   49.95
        13:30:11       0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
        13:30:11       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
* 查看进程，使用pidstat

        # 间隔5秒后输出一组数据
        $ pidstat -u 5 1
        13:37:07      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
        13:37:12        0      2962  100.00    0.00    0.00    0.00  100.00     1  stress


## 软中断
Linux 将中断处理过程分成了两个阶段，也就是上半部和下半部：
* 上半部用来快速处理中断，它在中断禁止模式下运行，主要处理跟硬件紧密相关的或时间敏感的工作。
* 下半部用来延迟处理上半部未完成的工作，通常以内核线程的方式运行。
* 
这两个阶段你也可以这样理解：
* 上半部直接处理硬件请求，也就是我们常说的硬中断，特点是快速执行；
* 而下半部则是由内核触发，也就是我们常说的软中断，特点是延迟执行。

Linux 中的软中断包括网络收发、定时、调度、RCU 锁等各种类型，我们可以查看 proc 文件系统中的 /proc/softirqs  ，观察软中断的运行情况。

  	$ watch -d cat /proc/softirqs
                      CPU0 CPU1
            HI: 0 0
         TIMER: 811613 1972736
        NET_TX: 49 7
        NET_RX: 1136736 1506885
         BLOCK: 0 0
      IRQ_POLL: 0 0
       TASKLET: 304787 3691
         SCHED: 689718 1897539
       HRTIMER: 0 0
           RCU: 1330771 1354737


watch -d cat /proc/softirqs，通过 /proc/softirqs 文件内容的变化情况，你可以发现， TIMER（定时中断）、NET_RX（网络接收）、SCHED（内核调度）、RCU（RCU 锁）等这几个软中断都在不停变化。

sar  可以用来查看系统的网络收发情况，还有一个好处是，不仅可以观察网络收发的吞吐量（BPS，每秒收发的字节数），还可以观察网络收发的 PPS，即每秒收发的网络帧数。


        # -n DEV 表示显示网络收发的报告，间隔1秒输出一组数据
        $ sar -n DEV 1
        15:03:46        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
        15:03:47         eth0  12607.00   6304.00    664.86    358.11      0.00      0.00      0.00      0.01
        15:03:47      docker0   6302.00  12604.00    270.79    664.66      0.00      0.00      0.00      0.00
        15:03:47           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
        15:03:47    veth9f6bbcd   6302.00  12604.00    356.95    664.66      0.00      0.00      0.00      0.05
* 第一列：表示报告的时间。
* 第二列：IFACE 表示网卡。
* 第三、四列：rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是  PPS。
* 第五、六列：rxkB/s 和 txkB/s 分别表示每秒接收、发送的千字节数，也就是  BPS。

tcpdump

        #-i eth0 只抓取eth0网卡，-n不解析协议名和主机名
        #tcp port 80表示只抓取tcp协议并且端口号为80的网络帧
  
        tcpdump -i eth0 -n tcp port 80

## CPU上下文切换
* 进程上下文切换
    * 进程的时间片耗尽
    * 系统资源耗尽，比如内存不足
    * sleep主动睡眠
    * 更高优先级进程
    * 硬件中断
* 线程上下文切换
* 中断上下文切换

查看CPU上下文切换，使用vmstat

        # 每隔5秒输出1组数据
        $ vmstat 5
        procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
        r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
        0  0      0 7005360  91564 818900    0    0     0     0   25   33  0  0 100  0  0


* cs（context switch）是每秒上下文切换的次数。
* in（interrupt）则是每秒中断的次数。
* r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。
* b（Blocked）则是处于不可中断睡眠状态的进程数。

查看进程详细情况，使用pidstat

        # 每隔5秒输出1组数据
        $ pidstat -w 5
        Linux 4.15.0 (ubuntu)  09/23/18  _x86_64_  (2 CPU)

        08:18:26      UID       PID   cswch/s nvcswch/s  Command
        08:18:31        0         1      0.20      0.00  systemd
        08:18:31        0         8      5.40      0.00  rcu_sched
        ...

* cswch  ，表示每秒自愿上下文切换（voluntary context switches）的次数，所谓自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。
* nvcswch  ，表示每秒非自愿上下文切换（non voluntary context switches）的次数。而非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。


详细的中断信息，调度中断（RES），这个中断类型表示，唤醒空闲状态的 CPU 来调度新的任务运行。这是多处理器系统（SMP）中，调度器用来分散任务到不同 CPU 的机制，通常也被称为处理器间中断（Inter-Processor Interrupts，IPI）。

        # -d 参数表示高亮显示变化的区域
        $ watch -d cat /proc/interrupts
                CPU0       CPU1
        ...
        RES:    2450431    5279697   Rescheduling interrupts
        ...

**总结：首先通过uptime查看系统负载，然后使用mpstat结合pidstat来初步判断到底是cpu计算量大还是进程争抢过大或者是io过多，接着使用vmstat分析切换次数，以及切换类型，来进一步判断到底是io过多导致问题还是进程争抢激烈导致问题。**

## CPU使用率
使用top

        # 默认每3秒刷新一次
        $ top
        top - 11:58:59 up 9 days, 22:47,  1 user,  load average: 0.03, 0.02, 0.00
        Tasks: 123 total,   1 running,  72 sleeping,   0 stopped,   0 zombie
        %Cpu(s):  0.3 us,  0.3 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
        KiB Mem :  8169348 total,  5606884 free,   334640 used,  2227824 buff/cache
        KiB Swap:        0 total,        0 free,        0 used.  7497908 avail Mem

        PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
        1 root      20   0   78088   9288   6696 S   0.0  0.1   0:16.83 systemd
        2 root      20   0       0      0      0 S   0.0  0.0   0:00.05 kthreadd
        4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H
        ...

* user（通常缩写为 us），代表用户态 CPU 时间。注意，它不包括下面的 nice 时间，但包括了 guest 时间。
* nice（通常缩写为 ni），代表低优先级用户态 CPU 时间，也就是进程的 nice 值被调整为 1-19 之间时的 CPU 时间。这里注意，nice 可取值范围是 -20 到 19，数值越大，优先级反而越低。
* system（通常缩写为 sys），代表内核态 CPU 时间。
* idle（通常缩写为 id），代表空闲时间。注意，它不包括等待 I/O 的时间（iowait）。
* iowait（通常缩写为 wa），代表等待 I/O 的 CPU 时间。
* irq（通常缩写为 hi），代表处理硬中断的 CPU 时间。
* softirq（通常缩写为 si），代表处理软中断的 CPU 时间。
* steal（通常缩写为 st），代表当系统运行在虚拟机中的时候，被其他虚拟机占用的 CPU 时间。
* guest（通常缩写为 guest），代表通过虚拟化运行其他操作系统的时间，也就是运行虚拟机的 CPU 时间。
* guest_nice（通常缩写为 gnice），代表以低优先级运行虚拟机的时间。

进程都有一个 %CPU 列，表示进程的 CPU 使用率。它是用户态和内核态 CPU 使用率的总和，包括进程用户空间使用的 CPU、通过系统调用执行的内核空间 CPU 、以及在就绪队列等待运行的 CPU。在虚拟化环境中，它还包括了运行虚拟机占用的 CPU

使用pidstat


        # 每隔1秒输出一组数据，共输出5组
        $ pidstat 1 5
        15:56:02      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
        15:56:03        0     15006    0.00    0.99    0.00    0.00    0.99     1  dockerd

        ...

        Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
        Average:        0     15006    0.00    0.99    0.00    0.00    0.99     -  dockerd

使用perf top


        $ perf top
        Samples: 833  of event 'cpu-clock', Event count (approx.): 97742399
        Overhead  Shared Object       Symbol
        7.28%  perf                [.] 0x00000000001f78a4
        4.72%  [kernel]            [k] vsnprintf
        4.32%  [kernel]            [k] module_get_kallsym
        3.65%  [kernel]            [k] _raw_spin_unlock_irqrestore
        ...

* 第一列 Overhead ，是该符号的性能事件在所有采样中的比例，用百分比来表示。
* 第二列 Shared ，是该函数或指令所在的动态共享对象（Dynamic Shared Object），如内核、进程名、动态链接库名、内核模块名等。
* 第三列 Object ，是动态共享对象的类型。比如 [.] 表示用户空间的可执行程序、或者动态链接库，而 [k] 则表示内核空间。
* 最后一列 Symbol 是符号名，也就是函数名。当函数名未知时，用十六进制的地址来表示。

调用关系

        # -g开启调用关系分析，-p指定php-fpm的进程号21515
        $ perf top -g -p 21515

## 进程状态
top查看

        $ top
        PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
        28961 root      20   0   43816   3148   4040 R   3.2  0.0   0:00.01 top
        620 root      20   0   37280  33676    908 D   0.3  0.4   0:00.01 app
        1 root      20   0  160072   9416   6752 S   0.0  0.1   0:37.64 systemd
        1896 root      20   0       0      0      0 Z   0.0  0.0   0:00.00 devapp
        2 root      20   0       0      0      0 S   0.0  0.0   0:00.10 kthreadd
        4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H
        6 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 mm_percpu_wq
        7 root      20   0       0      0      0 S   0.0  0.0   0:06.37 ksoftirqd/0

* R 是 Running 或 Runnable 的缩写，表示进程在 CPU 的就绪队列中，正在运行或者正在等待运行。
* D 是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uninterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断。
* Z 是 Zombie 的缩写，如果你玩过“植物大战僵尸”这款游戏，应该知道它的意思。它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源（比如进程的描述符、PID 等）。
* S 是 Interruptible Sleep 的缩写，也就是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。
* I 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。前面说了，硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。要注意，D 状态的进程会导致平均负载升高， I 状态的进程却不会。
* T 或者 t，也就是 Stopped 或 Traced 的缩写，表示进程处于暂停或者跟踪状态。向一个进程发送 SIGSTOP 信号，它就会因响应这个信号变成暂停状态（Stopped）；再向它发送 SIGCONT 信号，进程又会恢复运行（如果进程是终端里直接启动的，则需要你用 fg 命令，恢复到前台运行）。而当你用调试器（如 gdb）调试一个进程时，在使用断点中断进程后，进程就会变成跟踪状态，这其实也是一种特殊的暂停状态，只不过你可以用调试器来跟踪并按需要控制进程的运行。
* X，也就是 Dead 的缩写，表示进程已经消亡，所以你不会在 top 或者 ps 命令中看到它。