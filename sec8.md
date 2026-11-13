# 异常控制流（L14—L15）

> 程序不是顺序执行的。异常、中断、进程、信号、非本地跳转——这是 OS 与应用程序的交界。

### Lecture 14 Exceptional Control Flow: Exceptions and Processes（72.1 分钟）
- **主旨**：讲跳转/调用之外的 control flow（控制流）——exception（异常）与 process（进程）如何由硬件和 OS 协作实现，以及 fork/execve/wait/exit 构成的进程控制接口。
- **关键点**：
  1. 同步异常（synchronous exception）由执行某条指令引起，分 trap（有意，如 system call、断点，返回 "next" 指令）、fault（可能可恢复，重新执行出错指令，如 page fault）、abort（不可恢复，终止程序）。
  2. 每类事件有唯一 exception number k，k 作为 exception table（又称 interrupt vector，中断向量表）的下标，handler 在每次异常 k 发生时被调用。
  3. x86-64 每个 system call 有唯一编号：read=0、write=1、open=2、close=3、stat=4、fork=57、execve=59、exit=60、kill=62。
  4. 进程提供两个抽象：logical control flow（看似独占 CPU，由 context switch 上下文切换实现）与 private address space（看似独占主存，由 virtual memory 虚拟内存实现）。
  5. fork 调用一次返回两次、复制出独立地址空间，父子各自把 x 从 1 改成 2 和 0 互不影响；执行顺序不可预测，须用 process graph（进程图）的 topological sort 判定 feasible total ordering。
  6. 子进程终止后未被回收即成 zombie（`ps` 显示 defunct），父进程用 wait/waitpid 回收并取 exit status，父进程先退出则由 init（pid = 1）收养；execve 在当前进程加载新程序，argv[0] 约定为文件名。
- **易错/考点**：系统级函数出错返回 -1 并置全局 errno，必须逐个检查返回值（返回 void 的少数函数除外），且要用 WIFEXITED/WEXITSTATUS 才能取到真正的退出码。
### Lecture 15 Exceptional Control Flow Signals and Nonlocal Jumps（79.4 分钟）
- **主旨**：讲进程如何通过 signal（信号）接收异步事件通知，以及如何安全地在处理程序与主程序之间协作，避免竞态。
- **关键点**：
  1. Linux 进程层次以 init（pid 1）为祖先，daemon（如 httpd）挂在 init 下，login shell 再 fork 出 child/grandchild；shell 用 fork + execve 跑作业，用 waitpid 回收前台作业。
  2. signal 是内核发给进程的 small message，编号为小整数 ID（1–30）；内核在检测到系统事件（除零 SIGFPE、子进程终止 SIGCHLD）或另一进程调用 kill 系统调用时发送。
  3. 每个进程恰好属于一个进程组（getpgrp/setpgid）；键入 Ctrl-C 使内核向前台进程组的每个作业发 SIGINT（默认终止），Ctrl-Z 发 SIGTSTP（默认挂起），后台作业不受影响。
  4. pending 指已发送未接收，blocked 指信号可被阻塞、保持 pending 直到 unblock；同类信号最多只有一个 pending，信号不排队，因此不能用信号去计数子进程终止次数。
  5. 处理程序是独立逻辑流，与主程序并发执行且可被其他处理程序打断（嵌套）；编写规则 G0–G4 要求保持简单、只调用 async-signal-safe 函数（write/wait/_exit 安全，printf/sprintf/malloc/exit 不安全）、保存并恢复 errno、用 sigprocmask 保护共享数据、共享全局变量加 volatile。
  6. 消除竞态的做法是在 fork 前 sigprocmask(SIG_BLOCK, &mask_one, &prev_one) 阻塞 SIGCHLD、父进程 addjob 后再 SIG_SETMASK 解除；等待子进程用 sigsuspend(&prev) 原子地解除阻塞，替代 while(!pid) 忙等和 sleep(1)，可移植版本用 sigaction 配 SA_RESTART（被中断的慢速系统调用返回 EINTR）。
- **易错/考点**：`count--` 式的信号计数会漏掉并发到达的 SIGCHLD（信号不排队），必须靠 waitpid 循环回收；printf 不是 async-signal-safe，处理程序里只能用 write。
---
