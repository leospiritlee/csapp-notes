# 动态内存分配（L19—L20）

> 自己写一个 malloc：隐式/显式空闲链表、分离空闲链表、合并与对齐。做完这个 lab，才算真懂指针。

### Lecture 19 Dynamic Memory Allocation Basic Concepts（66.5 分钟）
- **主旨**：说明为什么需要在运行时从 heap（堆）申请虚拟内存，并给出以隐式空闲链表（implicit free list）为基础的分配器设计——块格式、放置策略、分裂与合并，以及吞吐率与内存利用率之间的取舍。
- **关键点**：
  1. 动态内存分配器（如 malloc）管理进程虚拟内存中称为 heap 的区域，堆顶由 brk 指针标示，用于大小到运行期才确定的数据结构；`void *malloc(size_t size)` 成功时返回至少 size 字节、按 8 字节（x86）或 16 字节（x86-64）边界对齐的块指针，size==0 返回 NULL，失败返回 NULL 并设置 errno。
  2. 两个性能目标常相互冲突：throughput（吞吐率，单位时间完成的请求数，例：10 秒内 5000 次 malloc + 5000 次 free 即 1000 operations/second）与 peak memory utilization（峰值利用率，以 aggregate payload Pk 即当前已分配 payload 总和衡量）。
  3. internal fragmentation（内部碎片）指块大小大于 payload，源于维护堆数据结构的开销、对齐填充和显式策略（如用大块满足小请求）；external fragmentation（外部碎片）指空闲总量足够却没有单个足够大的空闲块——p1=malloc(4)、p2=malloc(5)、p3=malloc(6)、free(p2) 后 p4=malloc(6) 就会失败。
  4. implicit free list 为每块保存 size 与 allocated 两种信息：header（头部）存块大小（字节），最低位作分配标志，因此块按双字（double-word）对齐；示例头部序列 8/0、16/1、32/0、16/1、0/1，分配块加阴影、空闲块不加阴影。
  5. 放置策略：first fit 从头扫描取第一个够大的空闲块（`p = p + (*p & -2)` 跳到下一块，最坏线性时间，易在表头产生 splinters）；next fit 从上一次搜索结束处继续（更快但碎片更糟）；best fit 取剩余字节最少者（利用率高但更慢）。
  6. free 的最简实现只清分配位（`*p = *p - 2`），不合并会产生 false fragmentation；coalescing（合并）后继块用 `next = p + *p` 并检查其分配位，而与前驱块合并必须借助 boundary tag（脚部 footer），才能把四种前后分配状态都变成常数时间（Case 1—4）。
- **易错/考点**：隐式链表分配为线性时间，实际 malloc/free 并不采用，但 splitting（分裂）与 boundary tag coalescing 的思想对所有分配器通用；边界标记带来额外内部碎片，只有需要向前合并的块才需要 footer。
### Lecture 20 Dynamic Memory Allocation Advanced Concepts（82.7 分钟）
- **主旨**：在隐式链表基础上引入显式空闲链表（explicit free list）与分离空闲链表（segregated free list）以提升吞吐率，并延伸到垃圾回收（garbage collection）与 C 语言中常见的内存错误及其检测工具。
- **关键点**：
  1. explicit free list 只维护空闲块的双向链表：空闲块在 Size/a 与 Size/a 之间放 Next、Prev 指针（借用 payload 区），逻辑上 A→B→C 有序而物理上块可任意分布，同时仍保留 boundary tag 以便合并。
  2. 释放时的插入策略：LIFO（last-in-first-out，插到表头，简单且常数时间，但研究显示碎片更严重）与 address-ordered（保证 addr(prev) < addr(curr) < addr(next)，需搜索但碎片更少）。
  3. LIFO 释放按前后块状态分情况：Case 1 前后均分配只需把块插入表头；Case 2 前驱空闲则 splice out（摘除）前驱、合并两块后插入表头；Case 4 前驱与后继都空闲则摘除两者、合并三个块再插入表头。
  4. segregated free list（seglist，分离空闲链表）为每个 size class 各建一条空闲链表（示例分级 1–2、3、4、5–8、9–inf，小尺寸常一尺寸一类，大尺寸按区间或 2 的幂归并），其 first fit 近似全堆 best fit，从而同时提高吞吐率与内存利用率。
  5. 垃圾回收自动回收堆内存、应用无需 free（Python、Ruby、Java、Perl、ML、Lisp、Mathematica 常用，C/C++ 只有 conservative 变体且不能保证回收全部垃圾）；mark and sweep（标记-清除）在块头加 mark bit，从 roots 出发标记所有可达块，再扫描释放未标记块。
  6. C 的保守式回收器靠 `is_ptr()` 判断某字是否指向已分配块，但 C 指针可指向块中部，于是必须解决"如何找到块起始"；配套考点是 C 声明优先级（`->`、`()`、`[]` 最高，`*` 与 `&` 次之，故 `int *p[13]` 是 13 个 int 指针的数组、`int (*p)[13]` 是指向 13 个 int 数组的指针）与 `*size--` 这类把指针当对象引用的越界写 bug。
  7. 内存错误工具：gdb 擅长定位错误指针解引用但难查其他内存 bug；data structure consistency checker 平时静默、出错才报，可当探针缩小范围；valgrind 通过二进制翻译重写可执行文件 text 段，运行时逐次检查引用（坏指针、越界写、引用已释放块）；glibc malloc 自带检查，可用 `setenv MALLOC_CHECK_ 3` 打开。
- **易错/考点**：显式链表仍需边界标记、只有空闲块才用 payload 存指针；`int **p; p = malloc(N*sizeof(int *))` 后逐行 malloc 才是二维数组，写错尺寸或把指针当对象引用是最典型的内存覆写来源。
---
