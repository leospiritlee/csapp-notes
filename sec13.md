# 并发编程（L23—L26）

> 进程、线程、I/O 多路复用三条并发路线，然后是同步：信号量、生产者-消费者、读者-写者、线程安全与可重入。

### Lecture 23 Concurrent Programming（71.6 分钟）
- **主旨**：解释为什么服务器必须靠并发（concurrency）同时服务多个客户端，并对比 process-based、event-based、thread-based 三条实现路线及其取舍。
- **关键点**：
  1. 并发程序的经典问题分为三类：race（竞态，结果取决于系统别处任意的调度顺序）、deadlock（死锁，资源分配不当导致无法推进）、livelock / starvation / fairness（活锁、饥饿、公平性，外部事件或调度决策使子任务无法前进）。
  2. 迭代服务器（iterative server）一次只处理一个请求：客户 1 connect 后服务器阻塞在 read 等数据，客户 2 只能阻塞在 connect 上，正如进度图所示"用户去吃午饭，客户 1 阻塞、服务器阻塞、客户 2 阻塞"。
  3. 三种并发方案：process-based（内核自动交错多个逻辑流，各自独立地址空间）、event-based（程序员用 select / epoll 这类 I/O multiplexing 手工交错，所有流共享同一地址空间）、thread-based（内核自动交错，共享同一地址空间）。
  4. process-based echo server 的骨架是 Accept 后 Fork：子进程 Close(listenfd) 后 echo(connfd) 再 exit(0)，父进程 Close(connfd)；父子各持 listenfd 与 connfd 的副本，进程间无共享状态，交换数据要靠 IPC（FIFO、System V shared memory、sockets）。
  5. event-based 服务器维护一个 connfd 数组，反复用 select / epoll 找出有输入待处理的描述符；优点是单一控制流、可单步调试、无进程/线程控制开销（Node.js、nginx、Tornado 即此设计），缺点是代码复杂、难做细粒度并发（如只收到部分 HTTP 请求头）、无法利用多核。
  6. thread-based 服务器的线程各有独立逻辑流与栈（栈上局部变量不受其他线程保护）、共享 code / data / kernel context，创建与回收比进程便宜约两倍（进程约 20K cycles）；线程必须 Pthread_detach 成 detached 以免内存泄漏，且 connfd 必须 malloc 后把指针传给线程，否则多个线程会读到同一个共享的 connfd 变量（data race）。
- **易错/考点**：把 connfd 的地址（主线程栈上的变量）直接传给线程会引发 data race；joinable 线程不 join 就泄漏资源，detached 线程不能被 join 也不能被杀。
### Lecture 24 Synchronization Basics（56.5 分钟）
- **主旨**：用 badcnt.c 演示共享计数器在并发下丢失更新，并用 progress graph（进度图）解释临界区，进而引入 semaphore（信号量）实现互斥。
- **关键点**：
  1. 共享变量的严格定义：当且仅当多个线程引用某个变量的同一实例时该变量才是共享的；global 变量只有 1 个实例，每个线程栈上各有 1 个局部变量实例，函数内 static 局部变量在虚拟内存中只有 1 个实例。
  2. badcnt.c 用 `volatile long cnt = 0`，两个线程各执行 niters 次 `cnt++`，最后检查 `cnt != 2*niters` 即报 "BOOM"。
  3. 汇编把 `cnt++` 拆成三步：L 阶段 `movq cnt(%rip), %rdx` 载入、U 阶段 `addq $1, %rdx` 更新、S 阶段 `movq %rdx, cnt(%rip)` 写回，三步之间都可被调度打断。
  4. 交错 H1, L1, U1, H2, L2, U2, S1, S2 时两个线程都读到 0，最终 cnt = 1 而不是 2，这就是丢失更新。
  5. progress graph 的两轴分别是两个线程的指令序列，点 (I1, I2) 表示线程 1 完成 I1、线程 2 完成 I2 的离散执行状态；trajectory（轨迹）是状态转移序列，进入 unsafe region（两个临界区重叠的部分）即出错。
  6. semaphore 是非负全局整数，只由 P、V 操作；P(s) 在 s > 0 时把 s 减 1 立即返回（测试与减 1 原子完成），s = 0 时挂起线程直到被 V 唤醒并重新减 1；V(s) 把 s 加 1，若有线程阻塞在 P 上则精确唤醒其中一个。mutex 初值为 1，P 即加锁、V 即解锁。
- **易错/考点**：`volatile` 只保证每次从内存读，不保证 L/U/S 三步原子，解决不了竞态；mutex 把 unsafe region 变成 forbidden region，任何轨迹都无法进入。
### Lecture 25 Synchronization Advanced（80.7 分钟）
- **主旨**：把信号量用到 producer-consumer（生产者-消费者）、readers-writers（读者-写者）和有界缓冲区上，并讨论 thread safety（线程安全）、reentrant（可重入）与 deadlock（死锁）的成因与避免。
- **关键点**：
  1. producer-consumer 的固定模式：producer 等 empty slot（空槽）、插入元素后通知 consumer；consumer 等 item、取出后通知 producer；典型例子是多媒体处理（producer 生成 MPEG 视频帧、consumer 渲染）和事件驱动的图形界面。
  2. sbuf 结构含 `buf` 数组、`n`（最大槽数）、`front`、`rear`、`mutex`（初值 1）、`slots`（初值 n）、`items`（初值 0）；`sbuf_remove` 的顺序是 P(items) → P(mutex) → 取 `buf[(++sp->front)%(sp->n)]` → V(mutex) → V(slots)。
  3. readers-writers 是互斥问题的推广：reader 只读、writer 修改、writer 必须独占、读者数量不限；第一类读者优先（写者可能饥饿），第二类写者优先（后到的读者必须等），两类都可能 starvation；第一类的解法是用 `mutex` 保护 `readcnt`、用信号量 `w` 保护对象，第一个进入的读者 P(w)、最后一个离开的读者 V(w)。
  4. thread-unsafe 函数分四类：不保护共享变量；跨多次调用保持状态（`rand` 用 `static unsigned int next`）；返回指向 static 变量的指针（`ctime`）；调用其他 thread-unsafe 函数；而 reentrant（可重入）函数不访问共享变量、无需同步，是 thread-safe 的真子集，class 2 只能改成可重入（`rand_r` 由调用者传 seed），class 3 可用 lock-and-copy（加锁后把结果拷进调用者缓冲区）。
  5. prethreaded server 由 master 线程 accept 后 `sbuf_insert`、worker 线程 `sbuf_remove` 处理；`echo_cnt` 用 `pthread_once` 初始化 `byte_cnt` 与 `mutex`，累加时 P(mutex)/V(mutex) 包住。
  6. deadlock 是"等待一个永远不为真的条件"：线程 1 持 A 等 B、线程 2 持 B 等 A 即永久等待；避免办法是按相同顺序获取资源（两线程都先 P(s0) 再 P(s1)），此时进度图上不再出现交叉的 forbidden region。
- **易错/考点**：`sbuf_remove` 中 P(items) 与 P(mutex) 顺序颠倒会死锁；`readcnt` 本身也是共享变量必须加锁；`rand`、`ctime` 不是线程安全的，多线程下要用 `rand_r` / `ctime_r`。
### Lecture 26 Thread Level Parallelism（65.6 分钟）
- **主旨**：讲如何在单核与多核上把程序拆成并行任务（求和、快排），用加速比与 Amdahl 定律衡量收益，并解释多核下的内存一致性（memory consistency）模型。
- **关键点**：
  1. 乱序（out-of-order）处理器由 instruction control 动态把指令流重排成操作流，同时执行多条指令；hyperthreading（超线程）复制 PC 与寄存器组到译码/控制部分，使同一套 functional units（功能单元）可交错执行两条线程的指令。
  2. 并行求和把 0…n−1 分成 t 个区间、每个线程处理一个（假设 n 是 t 的倍数）；psum-mutex 版本在 8 核 Shark 机器上 1 线程 51 秒，2 线程反而 456 秒、4 线程 790 秒——加锁/解锁本身开销极大。
  3. psum-array 让 peer 线程 i 把部分和累加到全局数组 `psum[i]`，主线程等所有线程结束后再对 psum 求和，从而消除 mutex；但实测耗时随核数增加反而上升（1.98 秒 → 5.86 秒），而 psum-local 版本"显著更快"，随核数增加从 1.14 秒降到 0.32 秒。
  4. 加速比 S_p = T1/Tp（Tp 为用 p 核的运行时间），效率 E_p = S_p/p = T1/(pTp)，取值 0～100%；Amdahl 定律 T_k = pT/k + (1−p)T，例：T = 10、p = 0.9、k = 9 时 T9 = 2.0，理论下限为 1.0。
  5. 并行快排：子数组规模 N ≤ Nthresh 时退化为串行快排，否则选 pivot 分区成 L（≤ p）与 R（≥ p）后递归 spawn 两个线程分别排序；对 2^27（134,217,728）个随机数实测最佳加速比 6.84X，最佳线程数取决于串行比例（从 1 试到 4096）。
  6. 内存一致性：snoopy cache 给每个 cache block 打 Invalid / Shared / Exclusive 标记，写必须拿到独占（exclusive）副本；顺序一致性（sequential consistency）下 `a=1; b=100;` 两个线程分别写读，打印结果不可能出现 100,1 或 1,100。
- **易错/考点**：Amdahl 定律说明串行比例 p 决定加速上限，加核无法突破；psum-array 的耗时随核数增加反而上升（1.98 → 5.86 秒），说明"去掉锁"并不自动等于加速。
---
