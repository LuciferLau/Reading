> 工作中经常用的都是脚本语言，久而久之，对较为底层的一些基础知识会慢慢淡化，遂通过此书做补全与复习。

# 第1章 文件I/O
本章主要介绍了内核提供的一些系统调用及其用法，介绍了unix系统文件的结构，为何说“一切皆fd”，

还有 File Struct 及其 mgr，强调了文件内置的偏移量可能带来的坑。

# 第2章 标准I/O库
本章延续上一章，介绍了 glibc 如何包装系统调用成为 c 库函数，以及函数的用法。

lsof：用于查看进程相关的fd。

fcntl：改变fd的属性。

# 第3章 进程环境
本章先是从编译原理方面入手，简述了程序在实际运行时可能经过的路，在main函数之前做的事情（通过 __attribute__ 设置)；

然后介绍了 exit 和 _exit 的区别，如何使用 atexit 注册在程序运行结束时执行的钩子；
<img width="792" height="456" alt="image" src="https://github.com/user-attachments/assets/b031d571-3144-467f-be2d-dd20dc655872" />

简述了程序如何执行动态库的热更新（自定义struct管理库的对外接口，以及 dlxxx 函数的使用）；

从内存管理常见的问题(数组越界，野指针，内存泄漏，二次释放)引出了 Valgrind 的使用简介；

最后是充满争议但在内核代码里无处不在的goto，如何设置跳点（setjmp)执行跳跃（longjmp）。

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

# 第5章 进程控制：状态、调度和优先级
传统的5个状态可以分为：创建、就绪、运行、阻塞、终止，而本书中提及7个状态（不包含创建）。

这些状态都会随内核版本不同而发生变化，不必过分纠结。

<img width="805" height="598" alt="image" src="https://github.com/user-attachments/assets/86c99074-96aa-4e60-9bd9-3cef8e2dd825" />

在实际生产环境中，我们常见的状态有 R（不必多说），S（I/O多路复用），T（strace/gbd调试）；

比较少见是 Z（代码有误没正确回收子进程），几乎没见过的是 X 和 D。
<img width="762" height="577" alt="image" src="https://github.com/user-attachments/assets/12900f83-0996-4f3b-bea8-b92bd39da230" />

书中提及了 CPU密集 和 I/O密集 2种不同风格的程序，前者可发挥CPU多核心优势，后者反之（同步）。

pidstat：查看进程用户态和系统态的执行时长占比，user+sys/real 越大，说明越是CPU密集型。


# 第6章 信号
# 第7章 理解Linux线程（1）
# 第8章 理解Linux线程（2）
# 第9章 进程间通信：管道
# 第10章 进程间通信：System V IPC
# 第11章 进程间通信：POSIX IPC
# 第12章 网络通信：连接的建立
# 第13章 网络通信：数据报文的发送
# 第14章 网络通信：数据报文的接收
# 第15章 编写安全无错代码
