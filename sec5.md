# 程序优化（L10—L10）

> 性能不是靠猜。这一讲给的是方法论：减少函数调用、消除内存别名、循环展开、提高指令级并行。

### Lecture 10 Program Optimization（73.6 分钟）
- **主旨**：讲如何写出编译器友好的代码，通过消除优化阻碍、利用指令级并行（ILP）把程序性能提升数倍。
- **关键点**：
  1. 编译器的根本约束是**不得改变程序行为**（除非用了非标准语言特性），且分析只在单个过程内、只基于静态信息，因此遇到疑问时必须保守（conservative），很多优化要程序员自己做。
  2. 通用优化包括 code motion（把 `a[n*i+j]` 中的 `n*i` 提到循环外）、strength reduction（强度削弱）、公共子表达式消除（4 个邻居求和从 3 次乘法 `n*n`、`(n-1)*n`、`(n+1)*n` 减到 1 次 `n*n`）、消除不必要的函数调用。
  3. 两大阻碍：一是过程调用，`lower()` 里 `for (i=0;i<strlen(s);i++)` 每轮重算 strlen（strlen 必须扫到 `'\0'` 才知长度），复杂度退化为 O(N²)，补救是 inline（-O1 起单文件内生效）或手动提出循环；二是内存别名（memory aliasing），`sum_rows(double *a, double *b, long n)` 中若 `b` 与 `a` 重叠（如 `double B[3] = A+3`），写 `b[i]` 会改到正在读的 `a[i*n+j]`，编译器不敢把 `b[i]` 留在寄存器，B 依次变为 {3,0,16}、{3,22,16}、{3,22,45}。
  4. 性能用 CPE（cycles per element）刻画：`T = CPE*n + Overhead`，CPE 是拟合直线的斜率；现代 CPU 是 superscalar（超标量，1993 年 Pentium 起）且功能单元流水化，3 次乘法可在 7 个周期内完成。
  5. 打破数据依赖才能突破 latency bound（延迟下界）：combine4 的 `imulq (%rax,%rdx,4),%rcx` 形成串行链、整数乘 CPE 为 3.01；2×1 展开 + 重新结合（`x = x OP (d[i] OP d[i+1])`）或双累加器（`x0`、`x1`）可降到 1.51/2.51，接近 throughput bound（1.00/0.50）。
  6. 进一步用 SIMD：`vaddss %ymm0,%ymm1,%ymm1` 一次并行加 8 个 float、`vaddpd` 一次加 4 个 double；分支靠分支预测，向量长度 100 时 i=99 预测「继续跳转」会错（misprediction），需清空并重载流水线（reload pipeline），代价是多个时钟周期。
- **易错/考点**：浮点加法/乘法不满足结合律与分配律，所以「重新结合」这类变换编译器不能自动做，必须由程序员显式改写（这正是本讲反复强调的点）。
---
