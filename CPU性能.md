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

cup使用率
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

总结：
首先通过uptime查看系统负载，然后使用mpstat结合pidstat来初步判断到底是cpu计算量大还是进程争抢过大或者是io过多，接着使用vmstat分析切换次数，以及切换类型，来进一步判断到底是io过多导致问题还是进程争抢激烈导致问题。