# 虚拟内存（L17—L18）

> 全课的高潮：虚拟地址如何翻译、页表怎么组织、缺页怎么处理、malloc 如何在虚拟内存之上管理堆。

### Lecture 17 Virtual Memory Concepts（71.2 分钟）
- **主旨**：建立 VM（虚拟内存）的三重身份——磁盘数据的 DRAM cache、内存管理工具、保护与共享机制，并推出地址翻译的位级拆分。
- **关键点**：
  1. 物理寻址（CPU 直接发 PA，用于嵌入式微控制器、汽车）对比虚拟寻址（CPU 发 VA，由 MMU 翻译成 PA），后者是现代服务器、笔记本和手机的共同基础。
  2. VM 的三大作用：把 DRAM 当作虚拟地址空间的 cache 以高效利用主存、简化内存管理（每进程获得统一的线性地址空间）、隔离地址空间（进程无法访问彼此内存与内核特权信息）。
  3. VM 作为 cache 时，虚拟内存是磁盘上 N 个连续字节的数组，块称为 page（页，大小 P = 2^p 字节）；DRAM 比 SRAM 慢约 10 倍、磁盘比 DRAM 慢约 10,000 倍，因此页取大值（典型 4 KB，有时 4 MB）、采用 fully associative（全相联）与 write-back（写回）。
  4. 页表是 PTE（页表项）的数组，每项含 valid bit 与物理页号或磁盘地址，是每进程一份、存放在 DRAM 的内核数据结构；访问未驻留页触发 page fault（缺页异常），由内核从磁盘调页。
  5. 地址翻译：VA 拆成 VPN（n−p 位）+ VPO（p 位），PTBR（页表基址寄存器）定位页表，PTE 给出 PPN，PA = PPN ‖ PPO 且 PPO 直接复用 VPO；TLB 进一步用 VPN 索引 T=2^t 个组，把 VA 拆成 TLBT + TLBI + VPO。
  6. 多级页表把 VPN 拆成 VPN1…VPNk，因为 4 KB 页、48 位地址、8 字节 PTE 的平坦页表需 2^48 × 2^3 / 2^12 = 2^39 字节即 512 GB；同一机制还简化链接加载（各程序 .text/.data 起始地址相同，execve 只建标 invalid 的 PTE、按需逐页复制）、实现保护（PTE 的 SUP/READ/WRITE/EXEC 位由 MMU 每次访问检查）与共享（不同进程虚拟页映射到同一物理页）。
- **易错/考点**：页内偏移 VPO 在翻译前后不变（PPO = VPO），只有 VPN 需要查表；缺页不等于非法访问，非法地址或权限不符才产生 SIGSEGV。
### Lecture 18 Virtual Memory Systems（77.8 分钟）
- **主旨**：用一个小型内存系统手算 TLB/页表/缓存的全流程，再落到真实 Intel Core i7 的四级页表与 Linux 的 mmap、COW 机制。
- **关键点**：
  1. 小系统模型：TLB 16 entries、4-way associative，L1 为 16 lines、4-byte block size、physically addressed、direct-mapped，配合页表可手算 0x03d4 等虚拟地址的 VPN/VPO 与 TLBI/TLBT 拆分，最终得到物理地址。
  2. Intel Core i7 存储层次为 L1 i-cache 与 L1 d-cache 各 32 KB 8-way、L2 256 KB 8-way、L3 8 MB 16-way 共享；地址翻译用 4 级页表，VPN 的 36 位均分为 4 组、每组 9 位，依次索引 PML4、PDPT、PD、PT，PTE 中含 XD（禁止执行）位等标志。
  3. 加速技巧是把 VPO 在地址翻译进行的同时直接送进 L1（virtually indexed, physically tagged），因为页内偏移翻译前后相同，从而省掉等待 MMU 的延迟。
  4. Linux 缺页处理：CPU 发 VA → MMU 查 PTE 失败 → 触发 page fault → 内核 trap handler 判定是合法缺页、权限错误还是非法地址，合法则从磁盘调入页面并更新 PTE，否则发 SIGSEGV。
  5. 内存映射（memory mapping）用 vm_area_struct 把虚拟区域与磁盘对象关联：可映射普通文件、匿名页（stack、heap）、/dev/zero（demand-zero page），脏页在内存与 swap 文件之间来回搬运；用户级接口是 `void *mmap(void *start, int len, int prot, int flags, int fd, int offset)`，flags 可取 MAP_PRIVATE/MAP_SHARED/MAP_ANON，返回映射区起始指针。
  6. Copy-on-write（COW，写时复制）：fork 时父子进程的 PTE 都标记 read-only、区域标记为 private copy-on-write，任一方写入即触发 protection fault，handler 为其复制出新页并改为可读写，然后重启指令；execve 则释放旧区域、为新映像的 .text/.data 建 area 并标 private COW，最后把 PC 设为 .text 入口。
- **易错/考点**：COW 的页在 fork 后是只读的，写操作走的是 protection fault 而非缺页；用 mmap 拷贝文件（`write(1, p, stat.st_size)`）可避免数据在用户空间与内核之间来回搬运。
---
