# 数据表示：整数与浮点（L1—L4）

> 先破除「整数就是整数」的直觉——溢出、补码、字节序、IEEE 754 舍入，这些不是边角料，而是后面所有 bug 的源头。

### Lecture 01 Course Overview（75.1 分钟）
- **主旨**：交代 15-213 的课程定位、教学与评分规则，并用「整数不是整数、浮点不是实数、内存会越界」等真实反例说明为什么必须理解机器级系统行为。
- **关键点**：
  1. Great Reality #1：`40000*40000` 得 1600000000（溢出），而浮点 `(1e20 + -1e20) + 3.14` 得 3.14、`1e20 + (-1e20 + 3.14)` 得 0，说明整数算术是模运算、浮点加法不满足结合律。
  2. Great Reality #2：实际几乎不手写汇编，但只有懂汇编才能在出 bug 时看清「high-level language models break down」，并支撑性能调优、编译器/OS 实现与安全漏洞分析。
  3. Memory Referencing Bug 示例：`typedef struct { int a[2]; double d; } struct_t;` 中执行 `s.a[i] = 1073741824` 越界写，`fun(4)` 起会覆盖 `d` 的值，结果 system dependent（画面不清晰，具体返回值未看清）。
  4. Memory System Performance：2048×2048 的 int 数组，`copyij` 按行遍历耗时 4.3 ms、按列遍历 81.8 ms（2.0 GHz Intel Core i7 Haswell），20 倍差距来自 locality（局部性）。
  5. C/C++ 不提供内存保护（越界数组引用、非法指针、malloc/free 滥用），后果可能是 action at a distance——被破坏的对象与被访问对象逻辑上无关，且 bug 可能在很久之后才显现；可用 Valgrind 等工具检测。
  6. 课程由 lectures（高层概念）+ recitations（工具与实验细节）+ 7 个 lab（L1 datalab 位操作、L2 bomblab 拆炸弹、L3 attacklab 代码注入、L4 cachelab 写缓存模拟器并优化局部性、L6 malloclab 写 malloc 包、L7 proxylab 写 Web proxy）以及 midterm + final 组成。
  7. 规则：lab 通过 Autolab 电子提交，迟交每天扣 15%、逾期 3 天后不再接受；一学期 5 个 grace day（每个 lab 最多自动用 2 个）；考试在 network-isolated clusters 上、可自选时间；课堂禁止任何录音。
- **易错/考点**：越界写为何能改到结构体里"无关"的成员（栈上布局 + 无边界检查），以及 grace day／迟交扣分的具体规则。
### Lecture 02 Bits, Bytes, and Integer（71.1 分钟）
- **主旨**：建立「一切皆比特」的表示观，讲清 unsigned 与 two's complement（补码）的编码公式、数值范围、位运算，以及有/无符号之间的转换、扩展与截断。
- **关键点**：
  1. byte = 8 bits，二进制 00000000₂–11111111₂ 对应十进制 0–255、十六进制 00–FF；C 里用 0x 前缀写十六进制（如 0xFA1D37B8），因为按位书写 32/64 位太长易错。
  2. 位运算的集合语义：w 位向量表示 {0,…,w−1} 的子集，`a_i = 1` 表示 i∈A；01101001 & 01010101 = 01000001 即交集 {0,6}，| 为并集 {0,2,3,4,5,6}，^ 为 symmetric difference（对称差）{2,3,4,5}。
  3. C 的 `&& || !` 视非零为 "True"、结果恒为 0/1 且有 early termination（短路）；`& | ~` 是按位逻辑运算，如 `!0x41 = 0x00`、`!!0x41 = 0x01`、`0x69 && 0x55 = 0x01`。
  4. 移位：`x << y` 左移丢弃高位、右侧补 0；`x >> y` 分 logical（补 0）与 arithmetic（复制符号位），对 10100010 而言 log>>2 得 00101000、arith>>2 得 11101000；移位量 <0 或 ≥ 字长属 undefined behavior（未定义行为）。
  5. 编码公式：B2U(X)=Σx_i·2^i，B2T(X)=−x_{w−1}·2^{w−1}+Σx_i·2^i，最高位是 sign bit（符号位）；`short` 下 15213 = 3B 6D、−15213 = C4 93（2 字节）。
  6. 数值范围（W=16）：UMax = 2^w−1 = 65535 (FF FF)、TMax = 2^{w−1}−1 = 32767 (7F FF)、TMin = −2^{w−1} = −32768 (80 00)、−1 = FF FF；同一 bit pattern 在两种解释下唯一但数值不同。
  7. 转换与扩展：signed↔unsigned 保持 bit pattern 只重新解释（T2U 对负数是加 2^w 的 ordering inversion）；混合表达式里 signed 被隐式转成 unsigned，导致 `-1 < 0U` 为真、`2147483647U < -2147483647-1` 为真；扩展时 unsigned 补 0、signed 复制 MSB，截断则丢弃高位并重新解释。
- **易错/考点**：混合有/无符号比较一律按 unsigned 求值（`-1 > 0U` 成立、`(unsigned)-1 > -2` 成立），以及 arithmetic vs logical right shift 的区别。
### Lecture 03 Bits, Bytes, and Integers cont（76.6 分钟）
- **主旨**：继续整数——加、乘、除在有限字长下的位级行为与溢出规律，并转到内存层面看字节序、字符串表示和一组整数谜题。
- **关键点**：
  1. Unsigned addition 丢弃最高进位（discard carry），即 UAdd_w(u,v) = u + v mod 2^w，天然实现模运算；two's complement addition 与 unsigned 的位级行为完全相同，`s = (int)((unsigned)u + (unsigned)v)` 与 `t = u+v` 必得 s == t。
  2. 乘法精确结果需要 2w 位：unsigned 最大 (2^w−1)² = 2^{2w} − 2^{w+1} + 1；two's complement 最负可达 −2^{w−1}(2^{w−1}−1)，只有 TMin² 需要 2w 位；C 的乘法只保留低 w 位，signed 与 unsigned 的高位不同但低位相同。
  3. 用移位代替乘除 2 的幂：`u << k` 等于 u·2^k（signed/unsigned 都成立）；unsigned 的 `u >> k` 用 logical shift 得 ⌊u/2^k⌋。
  4. unsigned 倒序循环陷阱：`for (i = cnt-2; i >= 0; i--)` 永不终止，因为 0−1 回绕成 UMax；正确写法是 `i < cnt`（C 标准保证 unsigned 加法是模运算），更稳妥用 `size_t`（长度等于 word size，cnt = UMax 也正确）。
  5. 字节序：big-endian（Sun、PPC Mac、Internet）把最低有效字节放在最高地址，little-endian（x86、Android/iOS/Windows 上的 ARM）把它放在最低地址；0x01234567 存于 &x = 0x100 时分别为 01 23 45 67 与 67 45 23 01。
  6. 字长与寻址：word size 是整数与地址的标称宽度，32 位限制地址空间为 4 GB (2^32)，64 位可达 18.4×10^19 字节（约 18 PB）；机器仍支持字长的分数或倍数数据格式，但总是整数个字节。
  7. 字符串用字符数组表示、ASCII 7 位编码（'0' = 0x30，数字 i 为 0x30+i）、以 0 结尾，故 byte ordering 对字符串无影响；`show_bytes` 靠把指针 cast 成 `unsigned char *` 逐字节打印（%p 打印指针、%x 打印十六进制），Linux x86-64 上 15213 显示为 6d 3b 00 00。
  8. 整数谜题（`int x=foo(); int y=bar(); unsigned ux=x; unsigned uy=y;`）：`x<0 ⇒ x*2<0` 假（TMin 反例）、`ux>=0` 恒真、`x&7==7 ⇒ x<<30<0` 真、`ux>-1` 假（−1 转成 UMax）、`x*x>=0` 假（溢出）、`(x|-x)>>31 == -1` 对 x≠0 成立。
- **易错/考点**：`x*2` 对 TMin 是溢出 UB、unsigned 下 `i>=0` 恒真导致死循环，以及 little-endian 下 `show_bytes` 输出顺序与书写顺序相反。
### Lecture 04 Floating Point（71.5 分钟）
- **主旨**：从二进制小数推进到 IEEE 754 浮点格式，讲清 normalized/denormalized/∞/NaN 的编码、舍入规则，以及由此导致的浮点运算代数性质缺失。
- **关键点**：
  1. 二进制小数 b_i…b_0 . b_−1 b_−2 … b_−j 的位权为 2 的幂（1/2、1/4、1/8…），5 3/4 = 101.11₂、2 7/8 = 10.111₂、1 7/16 = 1.0111₂；0.111111…₂ = 1.0 − ε，即「刚好小于 1」的一族数。
  2. IEEE Standard 754 于 1985 年确立，此前格式五花八门；它由数值分析学家主导制定（规范 rounding、overflow、underflow），代价是硬件实现变难。
  3. 三种精度位宽：single 32 位 = 1 符号 + 8 exp + 23 frac，double 64 位 = 1 + 11 + 52，extended 80 位（仅 Intel）= 1 + 15 + 63/64。
  4. Normalized（exp ≠ 000…0 且 ≠ 111…1）：V = (−1)^s M 2^E，E = Exp − Bias，Bias = 2^{k−1}−1（single 127，Exp 1…254 对应 E −126…127；double 1023，E −1022…1023），M = 1.xxx…x₂ 隐含前导 1（frac 全 0 时 M = 1.0，全 1 时 M = 2.0−ε）。
  5. 编码实例：float F = 15213.0 = 1.1101101101101₂ × 2^13，E = 13、Exp = 13 + 127 = 140 = 10001100₂，frac = 11011011011010000000000₂，结果为 `0 10001100 11011011011010000000000`。
  6. Denormalized（exp = 000…0）：E = 1 − Bias，M = 0.xxx…x₂，用于表示 0（+0 与 −0 是两个不同值）与等间距的、最接近 0 的数；exp = 111…1 时 frac = 0 表示 ±∞、否则为 NaN；8 位示例（k=4,n=3）中最小 denorm 为 1/512、最大 denorm 7/512、最小 norm 8/512、最大 norm 240。
  7. 舍入默认 round-to-nearest-even（就近舍入到偶数）：其他模式（towards zero、down、up）统计上有偏，会系统性高估或低估；二进制下以「舍入位右侧为 100…₂」判 half way，例如舍到 1/4：2 3/32→2、2 3/16→2 1/4、2 7/8→3、2 5/8→2 1/2。
  8. 浮点乘法：符号 s1^s2、有效数 M1×M2、指数 E1+E2，若 M ≥ 2 需右移并进位，再 round 到 frac 精度；加法满足交换律但不满足结合律（(3.14+1e10)−1e10 = 0 vs 3.14+(1e10−1e10) = 3.14），乘法不满足分配律；C 中 double→int 截断（向零舍入）、越界或 NaN 结果未定义（通常置 TMin），int→double 在 int ≤ 53 位时精确。
- **易错/考点**：denormalized 用 E = 1 − Bias 而非 0 − Bias（保证与最小 norm 平滑衔接），以及 round-to-even 只在恰好 half way 时才看奇偶。
---
