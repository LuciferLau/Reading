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

之所以说不可靠，是因为内核对于前31个的信号处理方式决定的：

对于信号的 mgr 来说，有一个位图 sigset 和一个等待(pending)列表 list；

在进程阻塞(sleep)时收到不可靠信号会先检查 set 里面有没有，没有则设1并且塞入list，但如果有了，就不处理直接丢弃；

但可靠信号则会每次都塞入list，直到上限（ulimit pending signals）

<img width="635" height="206" alt="image" src="https://github.com/user-attachments/assets/2b8e4842-95f2-49cf-bc54-ec8ebec164ca" />

## 信号的安装
func | 新旧 | desc
| :-: | :-: | -
singal | old | 功能比较单一，无法设置标志位，触发一次后重置handler
sigaction | new | 可以设置各种 SA_* 标志，屏蔽除当前信号外多个信号

## 信号的发送
func | desc
| :-: | -
kill | 给指定 pid 发送不同的 signal，0用于判断进程存活，衍生出 tkill/tgkill，指定线程发送，后者根据 tgid 发送防止 tid 复用发错
raise | 类似 kill，省略 pid（当前进程）
sigqueue | 也类似 kill，加入第3参数 sigval 可以传输1个int/指针给目标进程（自定义handler时使用）

## 信号的等待
> SIGKILL/SIGSTOP 无法等待，若尝试等待，操作将被忽略

func | desc
| :-: | -
pause | 进程进入可中断的睡眠状态，等待收到1个信号，因为不是原子的，可能导致signal丢失从而永远不被触发
siguspend | 等待指定信号，并将解除信号阻塞和pause封装成一个原子操作
sigwait | 降低 siguspend 的使用成本，优雅地等待指定信号，进阶还有 sigwaitinfo/sigtimedwait，捕获额外信息和超时功能
signalfd | 类似 sigwaitinfo，但可以通过 epoll 等方式监视就绪后 read 到来的信号，较为灵活

POSIX标准规定一个进程下的所有线程共用主进程(线程)的 sigHandler，在 fork 线程的时候会进行 ref++ 的操作，共用一块内存地址；

并且规定 kill/sigqueue 的信号都发送给所有线程，保证在如收到 SIGKILL 这种信号时所有线程都正常退出(exit_group)；

对于多个未决信号在 pending list 内，优先级为：不可靠 > 可靠，可靠内的为：编号小 > 编号大，不可靠内从高到低的为：同步信号->非实时信号->编号 

---

# 第7章 理解Linux线程（1）
> 线程又被称为轻量级进程(Light Weighted Process)，每一个用户态线程都有一个内核调度实体和自己的进程描述符(Task Struct)
> 而 Task Struct 内的 pid 其实是线程ID，tgid 才是进程ID（线程组ID），同一线程组内线程无层级关系

## POSIX标准线程API
func | desc
| :-: | -
pthread_create |	创建一个新的线程。需要提供线程函数（线程将执行的代码）、线程属性以及传递给线程函数的参数
pthread_join |	阻塞当前线程，直到指定的线程执行结束。可以获取被等待线程的返回值
pthread_exit |	终止调用该函数的线程，并可以返回一个值给等待它的线程（通过 pthread_join 获取）
pthread_detach |	将一个线程设置为分离状态。分离后，线程在终止时会自动释放其资源，无需其他线程对其调用 pthread_join()
pthread_self |	获取当前线程的线程 ID
pthread_equal |	比较两个线程 ID 是否相等

prctl：进程/线程属性设置，但它是Linux特有的，类似fcntl，ioctl，但fcntl是POSIX标准，ioctl也是非标准的

线程栈默认大小：ulimit -s / ulimit stack size

Linux 使用 futex(fast userspace mutex) 系统调用控制互斥锁，避免了在无线程竞争状态下不必要的系统调用消耗(用户态<->内核态)

如果临界区竞争不激烈，忙等待（自旋锁）的性能表现可能会好于互斥锁，因为可以减少系统调用的次数

<img width="722" height="364" alt="image" src="https://github.com/user-attachments/assets/e10d1698-6d67-405b-8b2a-0bec33be8b6f" />

---

# 第8章 理解Linux线程（2）
> 永远不要在多线程程序里面调用fork，因为只有调用fork线程会被复制，其它线程会进入 unexpected 状态（如果加锁则死锁，等）

pthread_cancel 是一个风险系数较大的操作，非必要不使用，根据线程的属性可能立即执行，可能异步执行，可能延迟到取消点执行

但无论怎么执行，都需要设计者关注线程的当前状态。

## 线程局部变量 Thread-Local Storage(TLS) 
> 这里讨论在线程外声明的变量，在线程函数声明的变量自然也是属于线程栈的空间，当然也是TLS

1、NPTL设计的接口：允许最多1024个线程内的全局变量的声明，并且可以设计他们各自的释放钩子（类似析构函数），用法较为繁杂，略；

2、__thread 关键字：GCC的标准，带有这个关键字的变量，在每个线程都会有独立的拷贝，生命周期直到线程退出

C11标准还引入了 thread_local 关键字

## 线程与信号

在多线程程序中，遵循一个原则：不要使用信号作为进程间通信的手段。因为其带来的收益，与其产生的风险不成比例

没有一个线程相关函数是异步信号安全的，sigHandler 里面也不能调用任何 pthread_* 函数

子线程会共享主线程的 sigHandler，但是 pthread_sigmask 可以自定义每个线程的信号掩码

pthread_kill/pthread_sigqueue 可以对线程发信号（区别见 Chap.6）

---

# 第9章 进程间通信：管道
> 在实际应用中，为了更加实现更加复杂的功能，一般不会使用原生的pipe，而是通过自己组装各种库的数据结构建立一个
 管道默认大小在 /proc/sys/fs/pipe-max-size

进程通信的方法，根据其目的可以大致分成2大类：通信 和 同步

<img width="568" height="772" alt="image" src="https://github.com/user-attachments/assets/987de1be-73a4-42df-9677-27a057e4ff15" />

管道是最早出现的进程间通信方法，常见就是在 shell 上通过 '|' 将1个进程的结果作为另1个进程的参数传入

但管道一边是单向的，且是阅后即焚的，如果需要双向通信，需要双方各自建立1个单向管道

为了不阻塞，一般会让一端的进程只写，关闭管道读端，另一边反之

popen 函数做的就是：fork子进程执行shell命令，然后创建 pipe 将结果返回给父进程

命名管道(FIFO)类似无名管道，现代实际工程中，基本都用万金油 socket 通信，这里先略过

---

# 第10章 进程间通信：System V IPC
> 这种对象不遵循 Unix 的"一切皆文件"的设计逻辑，没有fd，无法使用I/O多路复用
> 
> 基本就宣告了在高并发方面的死亡，注定是一些小众玩法，值得仔细品尝的应该就只有共享内存
> 
> Gemeni 锐评：
> 
> <img width="722" height="128" alt="image" src="https://github.com/user-attachments/assets/8e3e9781-29c3-4010-afe6-efc2efb32419" />

## 消息队列 Message Queue <sys/msg.h>
工程中一般自己实现，略

## 信号量 Semaphore <sys/sem.h>
condition更香，略

## 共享内存 Shared Memory <sys/shm.h>
多个进程共同将1块实际的内存空间映射到自己的虚拟地址空间中，零拷贝，听着就香，但我还是看隔壁POSIX标准吧

综上所述，略

---

# 第11章 进程间通信：POSIX IPC
> 熟悉的fd出现了，头文件的名字都变得正式了起来

## 消息队列 Message Queue <mqueue.h>
工程中依然一般自己实现，略

## 信号量 Semaphore <semaphore.h>
信号量的使用需要掌握 PV原语，

P：wait 阻塞等待，当然也有 trywait 非阻塞，timedwait 超时不等

V：post 通知唤醒阻塞进/线程

作者锐评：

<img width="847" height="196" alt="image" src="https://github.com/user-attachments/assets/dd20bc99-0e1c-4672-9d34-9a4dbba45457" />

<img width="708" height="482" alt="image" src="https://github.com/user-attachments/assets/684df0a2-fe76-41fe-95af-5aed33a9d4a7" />

## mmap：Memory-Map 内存映射，POSIX标准高性能的秘密武器

根据有无文件关联，分为基于文件的映射和匿名映射(malloc)

根据是否在进程间共享，可分为私有(fork)和共享(IPC)

Usage | desc
:-: | -
代替read/write操作文件 | 大部分情况下性能可能不如前者，因为可能引发大量的缺页中断，而前者一般有独立缓存
IPC | 通过 fcntl 提供的记录锁做到读写请求分离，零拷贝的操作同一款内存数据，前提是共享的内存映射

内存映射是通过页缓存（page cache）实现的，但具体的逻辑得深究内核代码，这里只做大致了解，核心是COW

## 共享内存 Shared Memory <sys/mman.h>
可以动态调整内存空间，因为是fd，所以可以使用文件相关的方法控制

本质是在 tmpfs 下（挂载位于/dev/shm）创建文件，通过 mmap 使用内存区域

> mount打印：tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)

---

# 第12章 网络通信：连接的建立
# 第13章 网络通信：数据报文的发送
# 第14章 网络通信：数据报文的接收
# 第15章 编写安全无错代码
