# cpu性能优化

## 1. cpu平衡负载

单位时间里的活跃进程数，包含了可运行状态和不可中断状态的平均进程数，一般认为平均进程数和cpu的个数保持一致是最理想状态。

### 查看cpu核数

```bash
$ grep 'model name' /proc/cpuinfo | wc -l
```



### 平均负载和cpu使用率

- CPU密集开，平均负载和CPU使用率是一致的
- I/O密集型，等待I/O会产生不可中断进程，此时平均负载升高，CPU使用率不一定很高
- 大量等待CPU的进程调度也会导致平均负载升高，此时CPU使用率也会比较高



## 模拟实验

我们模拟CPU密集型、I/O密集型、和大量进程的场景分别来看CPU的使用情况。

### 工具：

```bash
$ yum stress sysstat -y
```

stress   linux压测工具

sysstat    包含mpstat和pidstat， mpstat是多核分析工具，来用实时查看每个CPU的使用情况。pidstat是一个进程分析工具，实时查看进程的CPU、I/O、内存以及上下文切换指标



### 实验一：CPU密集型

```bash
# 一个CPU使用率100%
$ stress --cpu 1 --timeout 600 
```

```bash
# -d 表示高亮变化的部分
$ watch -d uptime 
 17:19:32 up 133 days, 19:03,  3 users,  load average: 0.01, 2.46, 3.58
```

查看CPU使用率变化情况

```bash
# -P ALL 表示监控所有CPU, 5表示5秒刷新一次
$ mpstat -P ALL 5
Linux 4.18.0-193.6.3.el8_2.x86_64 (seepre) 	04/19/2021 	_x86_64_	(2 CPU)

05:25:48 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:25:53 PM  all   50.15    0.00    0.30    0.00    0.20    0.00    0.00    0.00    0.00   49.35
05:25:53 PM    0   99.60    0.00    0.00    0.00    0.40    0.00    0.00    0.00    0.00    0.00
05:25:53 PM    1    0.60    0.00    0.60    0.00    0.00    0.00    0.00    0.00    0.00   98.80
```

找出使用率高的进程

```bash
# 隔5秒输出结果,输出1次
$ pidstat -u 5 1 
Linux 4.18.0-193.6.3.el8_2.x86_64 (seepre) 	04/19/2021 	_x86_64_	(2 CPU)

05:26:15 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
05:26:20 PM     0    114567    0.20    0.00    0.00    0.00    0.20     0  tuned
05:26:20 PM   989    178452    0.00    0.20    0.00    0.00    0.20     0  mysqld
05:26:20 PM     0   3115279    0.40    0.20    0.00    0.00    0.60     1  hosteye
05:26:20 PM     0   3116185   99.60    0.00    0.00    0.00   99.60     0  stress
```



### 实验二：I/O密集型

```bash
$ stress -i 1 --timeout 600
```

```bash
# -d 表示高亮变化的部分
$ watch -d uptime 
17:28:31 up 133 days, 19:12,  3 users,  load average: 1.28, 1.00, 2.22
```

查看CPU使用情况

```bash
$ mpstat -P ALL 5 
Linux 4.18.0-193.6.3.el8_2.x86_64 (seepre) 	04/19/2021 	_x86_64_	(2 CPU)

05:31:21 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:31:26 PM  all    1.80    0.00   48.50    0.40    0.30    0.10    0.00    0.00    0.00   48.90
05:31:26 PM    0    2.00    0.00   94.80    0.00    0.40    0.00    0.00    0.00    0.00    2.80
05:31:26 PM    1    1.60    0.00    2.20    0.80    0.20    0.20    0.00    0.00    0.00   95.00
```

查看是哪个进程导致的

```bash
$ pidstat -u 5 1
Linux 4.18.0-193.6.3.el8_2.x86_64 (seepre) 	04/19/2021 	_x86_64_	(2 CPU)

05:32:26 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
05:32:31 PM     0       462    0.00    0.20    0.00    0.00    0.20     0  jbd2/vda1-8
05:32:31 PM     0      1272    0.00    0.20    0.00    0.00    0.20     1  bcm-agent
05:32:31 PM   989    178452    0.00    0.20    0.00    0.00    0.20     1  mysqld
05:32:31 PM     0   3115279    0.80    0.20    0.00    0.00    1.00     1  hosteye
05:32:31 PM     0   3116697    1.60   97.01    0.00    0.20   98.60     0  stress
```



### 实验三：大量进程

```bash
$ stress -c 10 --timeout 600
```

```bash
$ watch -d uptime
 17:33:48 up 133 days, 19:17,  3 users,  load average: 5.25, 2.16, 2.25
```

查看CPU使用情况

```bash
$ mpstat -P ALL 5
Linux 4.18.0-193.6.3.el8_2.x86_64 (seepre) 	04/19/2021 	_x86_64_	(2 CPU)

05:34:11 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:34:16 PM  all   99.50    0.00    0.10    0.00    0.40    0.00    0.00    0.00    0.00    0.00
05:34:16 PM    0   99.60    0.00    0.00    0.00    0.40    0.00    0.00    0.00    0.00    0.00
05:34:16 PM    1   99.40    0.00    0.20    0.00    0.40    0.00    0.00    0.00    0.00    0.00
```

查看是哪个进程导致的

```bash
$ pidstat -u 5 1
Linux 4.18.0-193.6.3.el8_2.x86_64 (seepre) 	04/19/2021 	_x86_64_	(2 CPU)

05:34:53 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
05:34:58 PM     0      1272    0.00    0.20    0.00    0.00    0.20     1  bcm-agent
05:34:58 PM   989    178452    0.00    0.20    0.00    0.00    0.20     1  mysqld
05:34:58 PM     0   3113781    0.20    0.00    0.00    0.40    0.20     0  watch
05:34:58 PM     0   3116889   19.76    0.00    0.00   79.24   19.76     0  stress
05:34:58 PM     0   3116890   19.76    0.00    0.00   79.44   19.76     1  stress
05:34:58 PM     0   3116891   19.76    0.00    0.00   80.24   19.76     1  stress
05:34:58 PM     0   3116892   19.96    0.00    0.00   80.84   19.96     0  stress
05:34:58 PM     0   3116893   19.96    0.00    0.00   79.64   19.96     1  stress
05:34:58 PM     0   3116894   19.96    0.00    0.00   80.04   19.96     0  stress
05:34:58 PM     0   3116895   19.76    0.00    0.00   80.44   19.76     1  stress
05:34:58 PM     0   3116896   19.76    0.00    0.00   79.44   19.76     0  stress
05:34:58 PM     0   3116897   19.56    0.00    0.00   80.24   19.56     1  stress
05:34:58 PM     0   3116898   19.76    0.00    0.00   80.04   19.76     0  stress
```



## 2. CPU使用率

### CPU节拍率

为了维护CPU时间，Linux通过事先定义的节拍率（HZ），触发时间中断，并使用全局变量Jiffies记录了开机以来的节拍数。每发生一次中断，Jiffies的值就加1.

查看内核中节拍配置项

```bash
$ grep 'CONFIG_HZ' /boot/config-$(uname -r)
# CONFIG_HZ_PERIODIC is not set
# CONFIG_HZ_100 is not set
# CONFIG_HZ_250 is not set
# CONFIG_HZ_300 is not set
CONFIG_HZ_1000=y
CONFIG_HZ=1000
```

节拍率是内核选项，所以用户空间程序不能直接访问。为了方便用户空间程序，内核提供了一个用户空间节拍率USER_HZ，它总是固定为100，也就是1/100秒。这样，用户空间看到的总是USER_HZ也就是1/100秒。

### /proc查看CPU使用情况

```bash
$ cat /proc/stat | grep ^cpu
cpu  7776693 821287 7134682 2750356771 1303495 1924335 1399907 2 0 0
cpu0 3744638 239473 3058699 1376999690 112509 902365 763062 1 0 0
cpu1 4032054 581814 4075983 1373357080 1190985 1021970 636844 0 0 0
```

第一行是所有CPU的总和，第一列是CPU的编号。

#### 可以使用man  proc查看所有列的含义，下面是常见列的含义说明：

user（通常缩写为 us），代表用户态 CPU 时间。注意，它不包括下面的 nice 时间，但包括了 guest 时间。

nice（通常缩写为 ni），代表低优先级用户态 CPU 时间，也就是进程的 nice 值被调整为 1-19 之间时的 CPU 时间。这里注意，nice 可取值范围是 -20 到 19，数值越大，优先级反而越低。

system（通常缩写为 sys），代表内核态 CPU 时间。

idle（通常缩写为 id），代表空闲时间。注意，它不包括等待 I/O 的时间（iowait）。

iowait（通常缩写为 wa），代表等待 I/O 的 CPU 时间。

irq（通常缩写为 hi），代表处理硬中断的 CPU 时间。

softirq（通常缩写为 si），代表处理软中断的 CPU 时间。

steal（通常缩写为 st），代表当系统运行在虚拟机中的时候，被其他虚拟机占用的 CPU 时间。

guest（通常缩写为 guest），代表通过虚拟化运行其他操作系统的时间，也就是运行虚拟机的 CPU 时间。

guest_nice（通常缩写为 gnice），代表以低优先级运行虚拟机的时间。



### CPU使用率计算

通常说的cpu使用率是除了空闲时间外的其他时间占总CPU时间的百分比，公式如下：

```
CPU使用率 = 1 - （空闲时间/总CPU时间）
```

上面的公式可以从/proc/stat中很容易计算出来，但proc/stat是开机以来的总的节拍数，实际参考意义不大。事实上，计算CPU使用率一般都是取一段时间内（比如3秒）的两次值，相减之后再计算这段时间内的平均CPU使用率，公式如下：

```
平均CPU使用率 = 1 - ((空间时间new - 空间时间old)  /  (总cpu时间new  -  总cpu时间old))
```



### 某个进程CPU使用率

/proc/[pid]/stat



查看CPU使用率

```bash
$ top
top - 11:05:02 up 160 days, 12:48,  2 users,  load average: 0.00, 0.00, 0.00
Tasks:  99 total,   2 running,  97 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us,  0.8 sy,  0.0 ni, 98.3 id,  0.2 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3938.7 total,    177.1 free,    637.0 used,   3124.6 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3016.8 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
3118044 root      20   0 1290732  78764  13756 S   1.0   2.0 205:47.49 hosteye
      1 root      20   0   95352  11288   8388 S   0.3   0.3  21:39.66 systemd
    694 dbus      20   0   76928   4144   2984 S   0.3   0.1  32:41.96 dbus-daemon
 178452 mysql     20   0 1804408 378316  14004 S   0.3   9.4 396:09.11 mysqld
      2 root      20   0       0      0      0 S   0.0   0.0   0:02.73 kthreadd
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par_gp
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/0:0H-kblockd
      8 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 mm_percpu_w
     ...
```

按1可以看每个核的使用率

```bash
top - 11:05:39 up 160 days, 12:49,  2 users,  load average: 0.00, 0.00, 0.00
Tasks:  99 total,   1 running,  98 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.3 us,  0.3 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.7 us,  0.3 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.3 hi,  0.0 si,  0.0 st
MiB Mem :   3938.7 total,    176.5 free,    637.6 used,   3124.6 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3016.2 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
3118044 root      20   0 1290732  78752  13756 S   1.0   2.0 205:47.74 hosteye
      1 root      20   0   95352  11288   8388 S   0.0   0.3  21:39.66 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:02.73 kthreadd
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par_gp
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/0:0H-kblockd
      8 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 mm_percpu_wq
      9 root      20   0       0      0      0 S   0.0   0.0   0:22.45 ksoftirqd/0
     10 root      20   0       0      0      0 I   0.0   0.0  30:32.79 rcu_sched
     11 root      rt   0       0      0      0 S   0.0   0.0   0:01.03 migration/0
```

##### top命令是包含了内核态、用户进程空间、就绪队列等待运行的CPU，在虚拟化环境中，它还包括了运行虚拟机占用的CPU。所以，top并不适合分析进程cpu使用率。

### pidstat

```bash
$ pidstat 1 5   // 间隔1秒展示5组CPU使用率
Linux 4.18.0-193.6.3.el8_2.x86_64 (seepre) 	05/16/2021 	_x86_64_	(2 CPU)

11:09:00 AM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
11:09:01 AM     0        10    0.00    1.00    0.00    0.00    1.00     1  rcu_sched
11:09:01 AM     0   3118044    1.00    0.00    0.00    0.00    1.00     1  hosteye
...
Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0      1291    0.00    0.20    0.00    0.00    0.20     -  bcm-si
Average:        0    115580    0.00    0.20    0.00    0.00    0.20     -  rsyslogd
Average:      989    178452    0.00    0.20    0.00    0.00    0.20     -  mysqld
Average:        0   3118044    0.40    0.40    0.00    0.00    0.80     -  hosteye
Average:        0   3944860    0.00    0.20    0.00    0.00    0.20     -  sshd
Average:        0   3945391    0.00    0.20    0.00    0.00    0.20     -  pidstat
```

用户态CPU使用率（%usr）

内核态CPU使用率（%system）

运行虚拟机CPU使用率（%guest）

等待CPU使用率（%wait）

总的CPU使用率（%CPU）

最后的Average部分计算了5组数据的平均值



### Perf分析CPU使用率过高的问题

```bash
$ perf top
Samples: 75K of event 'cycles', 4000 Hz, Event count (approx.): 27075371904 lost: 0/0 drop: 0/0
Overhead  Shared Object                       Symbol
  30.79%  [kernel]                            [k] copy_p4d_range
   9.98%  [kernel]                            [k] unmap_page_range
   6.49%  [kernel]                            [k] page_remove_rmap
   3.01%  [kernel]                            [k] release_pages
   2.87%  [kernel]                            [k] free_pages_and_swap_cache
   1.70%  [kernel]                            [k] async_page_fault
	 ...
```

第一行，采样数（Samples）、事件类型（event）、事件总数（Event count）

下一行开始是一个表格，有以下几列：

Overhead，是该符号的性能事件在所有采样中的比例，用百分比表示

Shared，是该函数或指令所在的动态共享对象（Dynamic Shared Object），如内核、进程名、动态链接库名、内核模块名等

Object，是动态共享对象的类型，比如[.]表示用户空间的可执行程序、或者动态链接库，而[k]表示内核空间

Symbol，是符号名，也就是函数名。当函数名未知时，用十六进制的地址来表示

```bash
$ perf record  # 按Ctrl + c终止采样
```

终止采样会保存为pref.data，然后可以使用perf report来分析

```bash
$ perf report 
Samples: 2K of event 'cycles', Event count (approx.): 747922397
Overhead  Command          Shared Object                       Symbol
   2.24%  hosteye          libc-2.28.so                        [.] _int_malloc
   1.87%  hosteye          libc-2.28.so                        [.] malloc
   1.84%  hosteye          [kernel.kallsyms]                   [k] __d_lookup
   1.78%  hosteye          libc-2.28.so                        [.] __memmove_sse2_unaligned_erms
   1.50%  hosteye          libmodwebshell.so                   [.] boost::detail::function::function_obj_invoker2<boost::algorithm::detail::token_finderF<boost::algorithm::detail::is_any_ofF<char> >, boost::iterator_range<__gnu_cxx::__normal_iterator<char*, std::string> >
   1.36%  hosteye          libc-2.28.so                        [.] _int_free
   1.00%  hosteye          [kernel.kallsyms]                   [k] entry_SYSCALL_64
   1.00%  hosteye          liblogger.so                        [.] std::basic_string<char, std::char_traits<char>, std::allocator<char> >::basic_string
   0.79%  swapper          [kernel.kallsyms]                   [k] apic_timer_interrupt
   0.73%  sshd             ld-2.28.so                          [.] do_lookup_x
   0.71%  hosteye          libmodwebshell.so                   [.] utils::process::(anonymous namespace)::get_proc_cmdline
   0.66%  hosteye          hosteye                             [.] std::binary_search<char const*, char>
   0.64%  swapper          [kernel.kallsyms]                   [k] _raw_spin_lock_irqsave
   0.63%  hosteye          [kernel.kallsyms]                   [k] do_task_stat
   0.61%  hosteye          [kernel.kallsyms]                   [k] pid_revalidate
   ...
```

使用 -g 开启调用关系的采样

```bash
$ perf top -g
$ perf record -g
```



实验

准备两个台机器，一台安装ab压测工具，一台run两个docker，docker如下：

```bash
$ docker run --name nginx -p 10000:80 -itd feisky/nginx
$ docker run --name phpfpm -itd --network $ container:nginx feisky/php-fpm
```

然后在另一台机器测试

```bash
$ curl http://192.168.1.111:10000
It works!
```

然后安装ab压测工具

```bash
$ yum install httpd-tools
```

进行压测

```bash
# 并发10 测试100个请求
$ ab -c 10 -n 100 http://192.168.1.111:10000/   
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.1.226 (be patient).....done


Server Software:        nginx/1.15.4
Server Hostname:        192.168.1.226
Server Port:            10000

Document Path:          /
Document Length:        9 bytes

Concurrency Level:      10
Time taken for tests:   11.759 seconds
Complete requests:      100
Failed requests:        0
Write errors:           0
Total transferred:      17200 bytes
HTML transferred:       900 bytes
Requests per second:    8.50 [#/sec] (mean)
Time per request:       1175.903 [ms] (mean)
Time per request:       117.590 [ms] (mean, across all concurrent requests)
Transfer rate:          1.43 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.2      1       1
Processing:   268 1108 200.5   1129    1519
Waiting:      267 1107 200.5   1128    1518
Total:        269 1108 200.4   1129    1519

Percentage of the requests served within a certain time (ms)
  50%   1129
  66%   1182
  75%   1214
  80%   1245
  90%   1305
  95%   1371
  98%   1453
  99%   1519
 100%   1519 (longest request)
```

然后继续增加请求数，并切换到第一台机器，查看cpu使用率

```bash
$ ab -c 10 -n 10000 http://192.168.0.111:10000/
```

我们在另一台机器上使用top看一下cpu使用情况

```bash
$ top 
top - 14:35:04 up  1:44,  2 users,  load average: 2.21, 0.62, 1.28
Tasks: 233 total,   6 running, 177 sleeping,   0 stopped,   0 zombie
%Cpu0  : 99.0 us,  1.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 99.7 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu2  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  : 99.7 us,  0.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3918420 total,   195164 free,   953824 used,  2769432 buff/cache
KiB Swap:  2097148 total,  2096368 free,      780 used.  2493696 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 3698 daemon    20   0  336692  15332   7656 R  83.5  0.4   0:25.67 php-fpm
 3696 daemon    20   0  336692  13080   5404 R  79.9  0.3   0:25.90 php-fpm
 3700 daemon    20   0  336692  13016   5340 R  78.5  0.3   0:24.33 php-fpm
 3699 daemon    20   0  336692  13016   5340 R  78.2  0.3   0:23.27 php-fpm
 3697 daemon    20   0  336692  15368   7692 R  76.6  0.4   0:23.97 php-fpm
```

可以看到，php-fpm进程几乎将cpu占满了，接着用perf来找出具体是哪个函数

```bash
$ perf top -g -p 3696 
Tasks: 233 total,   6 running, 177 sleeping,   0 stopped,   0 zombie
Samples: 16K of event 'cycles', 4000 Hz, Event count (approx.): 4544996381 lost: 0/0 drop: 0/0
  Children      Self  Shared Object       Symbol
+   40.16%     0.00%  [unknown]           [.] 0x6cb6258d4c544155
+   40.16%     0.00%  libc-2.24.so        [.] 0x00007fa07c7f42e1
+   40.13%     0.00%  php-fpm             [.] 0x0000559b6d7b9642
+   40.13%     0.00%  php-fpm             [.] 0x0000559b6d56a6fc
+   40.13%     0.00%  php-fpm             [.] 0x0000559b6d619f94
+   40.13%     0.00%  php-fpm             [.] 0x0000559b6d6b1323
+   35.22%     0.00%  php-fpm             [.] 0x0000559b6d6b096e
+    6.26%     0.00%  php-fpm             [.] 0x0000559b6d77bea3
+    5.71%     0.00%  php-fpm             [.] 0x0000559b6d6b2a7c
```

然后找到php-fpm那一行，ente键进入

```bash

Samples: 23K of event 'cycles', Event count (approx.): 7210948027
  Children      Self  Command  Shared Object       Symbol
-   99.98%     0.00%  php-fpm  [unknown]           [k] 0x6cb6258d4c544155                                                             ◆
     0x6cb6258d4c544155                                                                                                               ▒
   - __libc_start_main                                                                                                                ▒
      - 99.94% 0x559b6d7b9642                                                                                                         ▒
         - 99.93% php_execute_script                                                                                                  ▒
              zend_execute_scripts                                                                                                    ▒
            - zend_execute                                                                                                            ▒
               - 91.54% execute_ex                                                                                                                                                                                               ▒
                  - 14.73% 0x559b6d6b2a7c                                                                                             ▒
                       1.15% sqrt                                                                                                     ▒
                       0.77% 0x559b6d46fab8                                                                                           ▒
                       0.52% 0x559b6d46fa5c                                                                                           ▒
                    0.88% 0x559b6d6b2add                                                                                              ▒                                                                                           ▒
                 1.60% 0x559b6d74618b                                                                                                 ▒
                 0.89% 0x559b6d73f39d                                                                                                 ▒
                 0.70% 0x559b6d6b29f1                                                                                                 ▒
                 0.63% 0x559b6d73cda8                                                                                                 ▒
                 0.59% 0x559b6d73f4dc                                                                                                 ▒
                 0.50% 0x559b6d6bba1f                                                                                                 ▒
+   99.98%     0.00%  php-fpm  libc-2.24.so        [.] __libc_start_main                                                              ▒
+   99.94%     0.00%  php-fpm  php-fpm             [.] 0x0000559b6d7b9642                                                             ▒
+   99.93%     0.00%  php-fpm  php-fpm             [.] php_execute_script                                                             ▒
+   99.93%     0.00%  php-fpm  php-fpm             [.] zend_execute_scripts                                                           ▒
+   99.93%     0.00%  php-fpm  php-fpm             [.] zend_execute                                                                   ▒
+   93.48%     5.13%  php-fpm  php-fpm             [.] execute_ex                                                                     ▒
+   15.92%     0.00%  php-fpm  php-fpm             [.] 0x0000559b6d77bea3                                                             ▒
+   14.73%     0.00%  php-fpm  php-fpm             [.] 0x0000559b6d6b2a7c
```

可以看到，有一个sqrt函数，然后我们将将代码拷贝出来分析一下

```bash
$ docker cp phpfpm:/app .
```

然后全局搜一下sqrt函数

```bash
$ grep sqrt -r app/
app/index.php:  $x += sqrt($x);
```

打开index.php文件

```bash
$ grep app/index.php
<?php
// test only.
$x = 0.0001;
for ($i = 0; $i <= 1000000; $i++) {
  $x += sqrt($x);
}

echo "It works!"
?>
```


## 系统CPU使用率很高，找不到高CPU应用

这种情况一般是短时应用导致的问题，可以有以下两种思路：

1. 应用里直接调用了其他二进制程序，这些程序通常运行时间比较短，通过top工具也不容易发现
2. 应用本身不停的崩溃重启，启动过程可能会占用相当多的CPU

我们可以使用pstree和execsnoop 找到他们的父进程，再从父进程入手。



### 实验

环境准备：

```bash
$ apt install docker.io sysstat linux-tools-common apache2-utils
$ docker run --name nginx -p 10000:80 -itd feisky/nginx:sp
$ docker run --name phpfpm -itd --network container:nginx feisky/php-fpm:sp
```

对php-fpm进行压测

```bash
$ ab -t 5 -t 600 http://192.168.1.226:10000/  # 5个并发 持续5分钟
```

然后查看cpu负载

```bash
$top 
top - 14:07:38 up 10 min,  2 users,  load average: 2.42, 0.91, 0.39
Tasks: 242 total,   3 running, 188 sleeping,   0 stopped,   0 zombie
%Cpu0  : 82.1 us, 10.0 sy,  0.0 ni,  6.3 id,  0.0 wa,  0.0 hi,  1.7 si,  0.0 st
%Cpu1  : 85.3 us,  8.7 sy,  0.0 ni,  5.7 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu2  : 80.4 us, 11.6 sy,  0.0 ni,  7.6 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu3  : 83.1 us, 10.0 sy,  0.0 ni,  7.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3918424 total,  1697448 free,   929496 used,  1291480 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  2553288 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4524 root      20   0   12324   5724   4888 S   4.3  0.1   0:02.56 containerd-shim
 4609 systemd+  20   0   33100   3812   2392 S   3.3  0.1   0:02.10 nginx
 8085 daemon    20   0  336692  15888   8212 S   3.0  0.4   0:01.35 php-fpm
 8067 daemon    20   0  336692  15888   8212 S   2.6  0.4   0:01.35 php-fpm
 8071 daemon    20   0  336692  15888   8212 S   2.6  0.4   0:01.34 php-fpm
 8073 daemon    20   0  336692  15888   8212 S   2.6  0.4   0:01.34 php-fpm
 8093 daemon    20   0  336692  15888   8212 S   2.6  0.4   0:01.33 php-fpm
17932 daemon    20   0    8180   1360    544 R   1.7  0.0   0:00.05 stress
  935 root      20   0 1213656 105304  52204 S   1.0  2.7   0:08.11 dockerd
 9891 root      20   0   46204   4504   3748 R   1.0  0.1   0:00.31 top
  913 root      20   0  961012  41596  23444 S   0.7  1.1   0:00.80 containerd
   11 root      20   0       0      0      0 I   0.3  0.0   0:00.54 rcu_sched
 4610 systemd+  20   0   33100   3812   2392 S   0.3  0.1   0:00.11 nginx
17935 daemon    20   0    8180    828    532 R   0.3  0.0   0:00.01 stress
    1 root      20   0  225448   9200   6628 S   0.0  0.2   0:04.86 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kthreadd
    3 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 rcu_gp
    4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 rcu_par_gp
    5 root      20   0       0      0      0 I   0.0  0.0   0:00.13 kworker/0:0-eve
    6 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H-kb
```

可以看到CPU的总负载很高，但却没有看到占CPU很高的应用

使用pidstat，还是看不出来

```bash
$ pidstat 1   # 每秒输出一组
Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0        10    0.00    0.12    0.00    0.00    0.12     -  ksoftirqd/0
Average:        0        11    0.00    0.12    0.00    0.12    0.12     -  rcu_sched
Average:        0       110    0.00    0.25    0.00    0.00    0.25     -  kworker/u16:3-events_unbound
Average:        0       808    0.12    0.00    0.00    0.00    0.12     -  thermald
Average:        0       913    0.00    0.12    0.00    0.00    0.12     -  containerd
Average:        0       935    0.75    0.50    0.00    0.00    1.25     -  dockerd
Average:        0      4524    0.37    3.74    0.00    0.00    4.11     -  containerd-shim
Average:      101      4609    0.75    2.62    0.00    0.87    3.36     -  nginx
Average:        0      4807    0.12    0.12    0.00    0.37    0.25     -  php-fpm
Average:        0      4823    0.00    0.12    0.00    0.00    0.12     -  kworker/u16:0-events_unbound
Average:        1      8067    0.37    2.24    0.00    0.62    2.62     -  php-fpm
Average:        1      8071    0.50    2.12    0.00    0.50    2.62     -  php-fpm
Average:        1      8073    0.50    2.12    0.00    0.50    2.62     -  php-fpm
Average:        1      8085    0.37    2.24    0.00    0.50    2.62     -  php-fpm
Average:        1      8093    0.25    2.37    0.00    0.50    2.62     -  php-fpm
Average:        0     13524    1.12    1.49    0.00    0.25    2.62     -  pidstat
Average:        1     14871    0.50    0.00    0.00    0.00    0.50     -  stress
Average:        1     14874    0.37    0.00    0.00    0.00    0.37     -  stress
```

还是看不出来占用CPU高的应用

然后，我们看top输出的信息，发现php-fpm都处理s状态，stress处理于r状态，我们查看stress的信息

```bash
$ pidstat -p 17935
Linux 5.4.0-74-generic (heads-Lenovo-Yoga-2-11) 	2021年06月13日 	_x86_64_	(4 CPU)

14时14分27秒   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
```

发现啥也没有，然后我们使用pstree查看stress的父进程 

```bash
$ pstree | grep stress
        |                 |         `-php-fpm---sh---stress---stress
```

可以看到，stress的父进程是php-fpm，然后我们就可以分析源码了，将源码拷贝出来

```bash
$ docker cp phpfpm:/app .
$ grep stress -r app
app/index.php:// fake I/O with stress (via write()/unlink()).
app/index.php:$result = exec("/usr/local/bin/stress -t 1 -d 1 2>&1", $output, $status);
```

发现在index.php文件里面调用了stress命令

```php
$ cat app/index.php
<?php
// fake I/O with stress (via write()/unlink()).
$result = exec("/usr/local/bin/stress -t 1 -d 1 2>&1", $output, $status);
if (isset($_GET["verbose"]) && $_GET["verbose"]==1 && $status != 0) {
  echo "Server internal error: ";
  print_r($output);
} else {
  echo "It works!";
}
?>
```

但是，就算是调用了stress命令我们也能够看出是stress命令占用CPU比较高，但实际情况确不是。接着，加一个参数verbose=1

```bash
$ curl http://192.168.1.226:10000?verbose=1
Server internal error: Array
(
    [0] => stress: info: [7401] dispatching hogs: 0 cpu, 0 io, 0 vm, 1 hdd
    [1] => stress: FAIL: [7402] (563) mkstemp failed: Permission denied
    [2] => stress: FAIL: [7401] (394) <-- worker 7402 returned error 1
    [3] => stress: WARN: [7401] (396) now reaping child worker processes
    [4] => stress: FAIL: [7401] (400) kill error: No such process
    [5] => stress: FAIL: [7401] (451) failed run completed in 0s
)
```

发现mkstemp failed: Permission denied，原来是没有权限执行stress命令，导致一直在报错，没有真正运行。

然后使用perf工具找出热点函数，看看导致CPU飙高具体是什么原因

```bash
$ perf record -g 
$ perf report 
\Samples: 282K of event 'cycles', Event count (approx.): 91373110412
  Children      Self  Command          Shared Object               Symbol
+   30.46%    28.49%  stress           libc-2.24.so                [.] random_r
+   21.68%    19.51%  stress           libc-2.24.so                [.] random
+   11.36%     6.09%  stress           libc-2.24.so                [.] rand
     4.92%     4.92%  stress           stress                      [.] 0x0000000000002efc
+    3.29%     0.00%  php-fpm          [kernel.kallsyms]           [k] entry_SYSCALL_64_after
+    3.28%     0.04%  php-fpm          [kernel.kallsyms]           [k] do_syscall_64
+    3.05%     0.00%  swapper          [kernel.kallsyms]           [k] secondary_startup_64
+    3.05%     0.00%  swapper          [kernel.kallsyms]           [k] cpu_startup_entry
+    3.05%     0.01%  swapper          [kernel.kallsyms]           [k] do_idle
+    2.94%     0.00%  swapper          [kernel.kallsyms]           [k] call_cpuidle
+    2.94%     0.00%  swapper          [kernel.kallsyms]           [k] cpuidle_enter
+    2.93%     0.01%  swapper          [kernel.kallsyms]           [k] cpuidle_enter_state
     2.79%     2.79%  stress           stress                      [.] 0x0000000000002f06
+    2.76%     2.75%  swapper          [kernel.kallsyms]           [k] intel_idle
     2.55%     2.55%  stress           stress                      [.] 0x0000000000002eef
     2.55%     2.55%  stress           stress                      [.] 0x0000000000000e80
     2.35%     2.35%  stress           stress                      [.] 0x0000000000002ee0
+    2.23%     0.00%  swapper          [kernel.kallsyms]           [k] start_secondary
+    2.14%     0.00%  php-fpm          [unknown]                   [k] 0x6cb6258d4c544155
+    2.14%     0.00%  php-fpm          libc-2.24.so                [.] __libc_start_main
+    1.96%     0.01%  stress           [kernel.kallsyms]           [k] page_fault
     1.87%     1.87%  stress           stress                      [.] 0x0000000000002f14
     1.86%     1.86%  stress           stress                      [.] 0x0000000000002f21
     1.69%     1.69%  stress           stress                      [.] 0x0000000000002f25
+    1.67%     0.01%  stress           [kernel.kallsyms]           [k] do_page_fault
+    1.67%     0.00%  stress           [kernel.kallsyms]           [k] entry_SYSCALL_64_after
+    1.66%     0.05%  stress           [kernel.kallsyms]           [k] do_syscall_64
Cannot load tips.txt file, please install perf!
```

发现random函数是random函数，然后修复权限并减少stress的调用就能解决这个问题了



### 大量瞬时进程

大量瞬时进程可以使用一个工具：https://github.com/brendangregg/perf-tools/blob/master/execsnoop

```bash
[root@slave ~]# ./execsnoop
Tracing exec()s. Ctrl-C to end.
Instrumenting sys_execve
   PID   PPID ARGS
 29450      0               sh-29450 [000] d... 80931.947422: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29452  29433 gawk -v o=1 -v opt_name=0 -v name= -v opt_duration=0 [...]
 29454  29451 cat -v trace_pipe
 29453  29450 /usr/local/bin/stress -t 1 -d 1
 29455   8445               sh-29455 [001] d... 80931.964506: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29457  29455 /usr/local/bin/stress -t 1 -d 1
 29459   8434               sh-29459 [000] d... 80931.967556: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29460  29459 /usr/local/bin/stress -t 1 -d 1
 29462   8439               sh-29462 [001] d... 80931.996296: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29463  29462 /usr/local/bin/stress -t 1 -d 1
 29466   8434               sh-29466 [001] d... 80932.005191: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29465   8446               sh-29465 [000] d... 80932.008042: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29467  29466 /usr/local/bin/stress -t 1 -d 1
 29470   8441               sh-29470 [001] d... 80932.011944: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29471   8445               sh-29471 [001] d... 80932.012629: execsnoop_sys_execve: (SyS_execve+0x0/0x30)
 29472  29470 /usr/local/bin/stress -t 1 -d 1
 29473  29471 /usr/local/bin/stress -t 1 -d 1
 29469  29465 /usr/local/bin/stress -t 1 -d 1
 ....
```

