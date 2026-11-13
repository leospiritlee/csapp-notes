# 系统级 I/O（L16—L16）

> read/write 的语义、短计数、缓冲的必要性，以及为什么标准 I/O 比系统调用快。

### Lecture 16 System Level I_O（71.0 分钟）
- **主旨**：把 Linux 的 Unix I/O 抽象（一切皆文件 + 五个系统调用）讲清，并给出应对 short count（短计数）的 RIO 包和内核文件表示模型。
- **关键点**：
  1. Linux file 就是 m 个字节的序列 B₀, B₁, …, Bₘ₋₁，普通文件分 text 与 binary 两类但内核不区分；所有 I/O 设备都表示为文件（/dev/sda2 为 /usr 磁盘分区、/dev/tty2 为终端），内核本身也是文件（/boot/vmlinuz-3.13.0-55-generic、/proc）。
  2. Unix I/O 的最小接口是 open/close、read/write、lseek，每个打开的文件维护一个 current file position（当前文件位置）k，指示下次读写的偏移。
  3. 目录是 link（链接）的数组，每个 link 把文件名映射到文件，至少含 `.`（指向自身）与 `..`（指向父目录）两项，操作命令为 mkdir/ls/rmdir。
  4. open 返回一个小的非负 file descriptor（fd，文件描述符），出错返回 -1；read/write 返回实际传输字节数，可能少于请求，即 short count——它出现在读遇 EOF、从终端读文本行、读写 network socket 时，读磁盘文件（除 EOF）和写磁盘文件不会出现。
  5. RIO 包分两类：无缓冲的 rio_readn/rio_writen（遇 EINTR 置 nread=0 后重试 read，nread=0 视为 EOF 跳出）、有缓冲的 rio_readlineb/rio_readnb（用 rio_buf、rio_bufptr、rio_cnt 管理已读未用字节），线程安全且可在同一描述符上交错调用。
  6. 内核用三张表表示打开文件：descriptor table（每进程一张）、open file table（全局共享，含 file pos 与 refcnt）、v-node table（含 file access/size/type）；fork 后子进程复制描述符表并共享同一 open file entry，refcnt+1，dup2 用于 I/O 重定向；标准 I/O 则在 buf 中攒字节，遇 `\n`、fflush、exit 或 main 返回才 flush，最终等价于 write(1, buf, 6)。
- **易错/考点**：short count 是常态而非错误，单字节拷贝 `while (Read(...) != 0)` 必须处理返回值；重定向要在子进程 execve 之前用 dup2 完成。
---
