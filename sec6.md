# 存储层次与缓存（L11—L12）

> 为什么按行遍历比按列快 20 倍？答案在 cache。这两讲是全课最需要动笔算的部分——地址怎么拆成 tag/set/offset。

### Lecture 11 The Memory Hierarchy（75.8 分钟）
- **主旨**：把寄存器、SRAM/DRAM、磁盘到远程存储按每字节成本与访问延迟排成存储金字塔，并用 locality（局部性）解释上层如何充当下层的 cache（缓存）来弥合 CPU–memory gap（CPU 与内存的速度差）。
- **关键点**：
  1. 非易失存储器（nonvolatile memory）含 ROM、PROM、EPROM、EEPROM 与 flash，flash 按 block 级擦除、约 10 万次擦写后磨损，常用于存 BIOS 固件、SSD 与磁盘缓存。
  2. 总线（bus）是一组并行导线，承载地址、数据与控制信号且被多设备共享；`movq %rax, A` 就是 CPU 经 bus interface、I/O bridge、memory bus 把数据写到主存地址 A 的 store 事务。
  3. 磁盘几何上同一半径的 track（磁道）构成 cylinder（柱面）；现代盘把磁道切成 recording zone，同一 zone 内扇区数相同（由最内圈周长决定），故容量按平均每道扇区数计算。
  4. 访问时间 = 寻道 + 旋转延迟 + 传输：7200 RPM、平均寻道 9 ms、每道 400 扇区时 Tavg rotation = 4 ms、Tavg transfer = 0.02 ms，Taccess ≈ 13.02 ms，开销几乎全在寻道与旋转。
  5. 磁盘控制器把 logical block（逻辑块号）翻译成 (surface, track, sector) 并用 DMA（直接内存访问）写入主存；SSD 以 page 为单位读写、以 block 为单位擦除，改一页需搬走整块其余页，故随机写慢于顺序写。
  6. 三类 cache miss：cold/compulsory（缓存为空）、conflict（多个块只能映射到同一组，如 block i 只能放在 block i mod 4）、capacity（working set 超过 cache 容量）。
- **易错/考点**：`sum_array_cols` 按列遍历（stride = N）破坏 spatial locality，与行优先 `sum_array_rows` 的对比是局部性分析的必考判断。
### Lecture 12 Cache Memories（78.7 分钟）
- **主旨**：把 cache 抽象成 (S, E, B) 的组相联结构，讲地址如何切成 tag / set index / block offset 判命中，并用 memory mountain（内存山）与 blocking（分块）说明怎样写 cache 友好的代码。
- **关键点**：
  1. 通用 cache 有 S = 2^s 个 set（组），每组 E = 2^E 条 line（行），每行含 B = 2^b 字节的 block，另有 valid bit 与 tag。
  2. 读 cache 时地址切成 t 位 tag、s 位 set index、b 位 block offset：低 b 位定块内偏移，中间 s 位选组，高 t 位与组内 tag 比较；直接映射示例中地址 0_01_100 的 set index 为 1。
  3. 直接映射（E = 1）4 位地址、B = 2、S = 4 的例子中，轨迹 0,1,7,8,0 依次 miss、miss、miss、miss、hit；2-way 组相联则同时比较同组两条 line 的 tag，valid 且匹配即 hit。
  4. 写策略两维正交：write-through（写直达）对 write-back（写回，需 dirty bit 记录脏行）；write-allocate（写分配，先装入 cache）对 no-write-allocate（直写内存不装入）。
  5. Intel Core i7：L1 i-cache/d-cache 32 KB、8-way、4 cycles，L2 256 KB、8-way、10 cycles，L3 8 MB、16-way、40–75 cycles，block size 64 bytes；L1 miss rate 约 3–10%，miss penalty 约 50–200 cycles。
  6. memory mountain 以 read throughput（读带宽 MB/s）对 stride 与 working set 作图：陡坡是 spatial locality 的斜率，山峰是 temporal locality 的脊线，stride-1 访问最快。
- **易错/考点**：分块矩阵乘把 miss 从 (9/8)·n³ 降到 1/(4B)·n³，前提是块大小满足 3B² < C（cache 容量），且 (kij) 版内层循环 A 的 miss 为 0.0、B 与 C 各 0.25。
---
