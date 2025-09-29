> 工作中经常用的都是脚本语言，久而久之，对较为底层的一些基础知识会慢慢淡化，遂通过此书做补全与复习。

# 第1章 文件I/O
本章主要介绍了内核提供的一些系统调用及其用法，介绍了unix系统文件的结构，为何说“一切皆fd”，

还有 File Struct 及其 mgr，强调了文件内置的偏移量可能带来的坑。

---

# 第2章 标准I/O库
本章延续上一章，介绍了 glibc 如何包装系统调用成为 c 库函数，以及函数的用法。

lsof：用于查看进程相关的fd。

fcntl：改变fd的属性。

---

# 第3章 进程环境
本章先是从编译原理方面入手，简述了程序在实际运行时可能经过的路，在main函数之前做的事情（通过 __attribute__ 设置)；

然后介绍了 exit 和 _exit 的区别，如何使用 atexit 注册在程序运行结束时执行的钩子；
<img width="792" height="456" alt="image" src="https://github.com/user-attachments/assets/b031d571-3144-467f-be2d-dd20dc655872" />

简述了程序如何执行动态库的热更新（自定义struct管理库的对外接口，以及 dlxxx 函数的使用）；

从内存管理常见的问题(数组越界，野指针，内存泄漏，二次释放)引出了 Valgrind 的使用简介；

最后是充满争议但在内核代码里无处不在的goto，如何设置跳点（setjmp)执行跳跃（longjmp）。

---

# 第4章 进程控制：进程的一生
本章介绍了 PID，随之而来的便是 fork，Task Struct，父子的关系（执行顺序不定，CFS完全公平调度）；

COW: 父进程内存通过多级页表管理，子进程只有在修改内存内容时触发缺页异常，此时内核执行拷贝；

vfork：COW未出现前的代替品，就是基本不复制父进程，靠指针引用父进程相关数据；

一些进程常见的属性（sid，gid，pid），以及一些操作他们的函数，
记住Seesion，Group，Process应该很容易区分它们的含义；

随之自然而然就会想到守护进程daemon，实现一次基本可以搞懂这些接口是干什么用的。

exec：基于 execve 系统调用的封装，共6个接口，通过参数形式（数组，列表）和环境变量（当前，指定）来区分
<img width="772" height="275" alt="image" src="https://github.com/user-attachments/assets/463990c9-f6bc-4f86-be3a-cdd024fe812c" />

system：父进程通过创建 shell子进程，子进程创建孙子进程，并且执行 bash cmd；

需要注意的是父进程对 SIGCHLD 的处理，因为 system 的返回值会受影响，

若 signal(SIGCHLD, SIG_IGN)，则必返回-1，无法知道 cmd 执行结果，成功返回0；

这会导致竞争状态，这个信号谁来消费，是shell子进程？还是父进程？这是不可知的。

父进程拿到信号 signal(SIGCHLD, sigHandler) 会导致 system 函数内部的 waitpid 失效（僵尸进程）；

shell子进程的 waitpid 正常处理信号，孙子正常清理，但父进程无事可做，拿到信号后 errno 被设为 ECHILD。
<img width="762" height="338" alt="image" src="https://github.com/user-attachments/assets/602f97b0-e0c7-432e-a262-40c1459d9609" />

sysctl：内核常量的读取和管理。

ps：进程的状态监控指令。

---

# 第5章 进程控制：状态、调度和优先级
传统的5个状态可以分为：创建、就绪、运行、阻塞、终止，而本书中提及7个状态（不包含创建）。

这些状态都会随内核版本不同而发生变化，不必过分纠结。

<img width="805" height="598" alt="image" src="https://github.com/user-attachments/assets/86c99074-96aa-4e60-9bd9-3cef8e2dd825" />

在实际生产环境中，我们常见的状态有 R（不必多说），S（I/O多路复用），T（strace/gbd调试）；

比较少见是 Z（代码有误没正确回收子进程），几乎没见过的是 X 和 D。
<img width="762" height="577" alt="image" src="https://github.com/user-attachments/assets/12900f83-0996-4f3b-bea8-b92bd39da230" />

书中提及了 CPU密集 和 I/O密集 2种不同风格的程序，前者可发挥CPU多核心优势，后者反之（同步）。

pidstat：查看进程用户态和系统态的执行时长占比，user+sys/real 越大，说明越是CPU密集型。

并发：多任务在单处理器上轮流占用CPU资源，并行：多任务在多处理器系统同时占用不同CPU资源。

内核使用工作队列为每个CPU核心安排任务，称为 per cpu run queue，但这可能导致旱的旱死涝的涝死；

于是设计者通过 load_balance 将任务从繁忙队列移动到较为空闲的队列，实现任务调度，从而达到负载均衡。

## CFS (Completely Fair Scheduler)：完全公平调度
首先，时间的长短是相对的，对于应用程序来说，一个操作执行1ms甚至1s都很正常，但对于内核来说，颗粒度显然是更低的ns。

在实际的生产环境中的机器，可能有各种优先级不同的进程，在这个算法里，优先级由 nice 值决定；

这个值范围是 [-20, 19] 值越高，优先级越低，进程的权重越小，默认为0。(越nice越受欺负？）
### 调度周期的计算
要计算调度周期，就得先计算调度延迟，公式为：理想周期(sched_latency_ns)/最小粒度(sched_min_granularity_ns)
<img width="757" height="449" alt="image" src="https://github.com/user-attachments/assets/1f0dc7b3-d44b-41b4-a36d-2b3293661d8b" />

而调度周期是一个动态的值，取决于当前进程数量(nr_running)

| 进程数与调度延迟的关系 | 调度周期 |
| - | - |
|  <=  | 理想周期 |
|  >  | 进程数 * 最小粒度 |

### 不考虑优先级情况下的时间片计算
进程获得的运行时间片 = 调度周期 / nr_running

假设理想周期为 20ms，最小粒度为 5ms，那么此时调度延迟为4：

| nr_running | 调度周期 | 每个进程获得的实际时间片 |
| - | - | - |
|  3(<4)  | 3*5ms < 20ms | 20 / 3 ~= 6ms |
|  5(>4)  | 5*5ms > 20ms | 3ms |

### 优先级情况下的时间片计算
假设该进程的权重占比为 K = weight / totalweight(<=1)，进程获得的运行时间片 = 调度周期 * K；

可以通过 getpriority() / setpriority() 这组函数控制进程优先级。

进程运行相关信息保存在 /proc/PID/sched 内，se_sum_exec_runtime 为实际运行物理时间。

### 既然有权重，为什么还叫完全公平？
既然有因为优先级各不相同 rumtime，自然会有一个不受优先级大幅影响的 vruntime，

vruntime 的增加量 ≈ 实际运行时间 / 进程权重，内核只需要不断调度 vrumtime 最小的任务，既可保证加权公平。

CPU亲和力：ps命令中PSR说明进程在某个CPU上运行，如果要绑定到某个CPU上，则称之为设置其亲和力。

---

# 第6章 信号
> 一个古早的通信机制，1-31为不可靠信号，[SIGRTMIN,SIGRTMAX]为可靠信号。
<img width="635" height="206" alt="image" src="https://github.com/user-attachments/assets/2b8e4842-95f2-49cf-bc54-ec8ebec164ca" />






---

# 第7章 理解Linux线程（1）
# 第8章 理解Linux线程（2）
# 第9章 进程间通信：管道
# 第10章 进程间通信：System V IPC
# 第11章 进程间通信：POSIX IPC
# 第12章 网络通信：连接的建立
# 第13章 网络通信：数据报文的发送
# 第14章 网络通信：数据报文的接收
# 第15章 编写安全无错代码
