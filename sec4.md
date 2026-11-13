# 机器级编程：汇编与栈帧（L5—L9）

> 从 C 到汇编，重点不是会写汇编，而是能读懂：过程调用的栈帧怎么建、控制流怎么跳、数组和结构体怎么寻址、缓冲区溢出怎么发生。

### Lecture 05 Machine Level Programming I Basics（73.5 分钟）
- **主旨**：从 C 代码走到机器码——先交代 x86 演化史与 ISA/microarchitecture 概念，再讲汇编的数据类型、寄存器与 mov/leaq/算术指令的操作数和寻址语义。
- **关键点**：
  1. Intel x86 演化里程碑：8086（1978，29K 晶体管，5–10 MHz，首款 16 位，IBM PC/DOS 基础，1 MB 地址空间）→ 386（1985，275K，首款 32 位即 IA32，加 flat addressing 可跑 Unix）→ Pentium 4E（2004，125M，首款 64 位即 x86-64）→ Core 2（2006，291M，首款多核）→ Core i7（2008，731M，1700–3900 MHz）。
  2. x86-64 的由来：Intel 2001 年推 IA64/Itanium 彻底换架构、只能 legacy 跑 IA32 且性能失望；2003 年 AMD 以演进式 x86-64（AMD64）抢先，Intel 2004 年以 EM64T 跟进，二者几乎相同，如今除低端外所有 x86 处理器都支持 x86-64。
  3. 概念区分：architecture/ISA 是写汇编或机器码必须理解的规范部分（指令集、寄存器），microarchitecture 是其实现（cache、主频）；machine code 是处理器执行的字节级程序，assembly code 是它的文本表示。
  4. 编译流水线：p1.c/p2.c 经 `gcc -Og -S` 得 .s，经 assembler 得 .o，再与 static libraries (.a) 一起由 linker 生成可执行文件 p；`-Og` 是较新 GCC 的基础优化级别。
  5. 汇编层的数据类型极简：只有 1/2/4/8 字节的 "integer"（既表示 data value 也表示无类型指针地址）与 4/8/10 字节浮点，没有 aggregate type（数组/结构体只是连续分配的字节）。
  6. `movq Source, Dest` 三类操作数：immediate（`$0x400`、`$-533`，编码 1/2/4 字节）、register（16 个整数寄存器 %rax/%rcx/%rdx/%rbx/%rsi/%rdi/%rsp/%rbp/%r8–%r15，%rsp 保留特殊用途）、memory（`(%rax)` 或 `D(R)` 即 Mem[Reg[R]+D]）；地址计算实例：`0x8(%rdx)`=0xf008、`(%rdx,%rcx)`=0xf100、`(%rdx,%rcx,4)`=0xf400、`0x80(,%rdx,2)`=0x1e080。
  7. 算术指令统一为 `Dest = Dest op Src`（addq/subq/imulq/salq/sarq/shrq/xorq/andq/orq），参数顺序与 mov 相反；指令对 signed/unsigned 不加区分；`leaq` 只做地址运算不访存，故 `y*48` 编译为 `leaq (%rsi,%rsi,2),%rdx`（3y）再 `salq $4`（×16）。
  8. 机器码落地示例：`*dest = t` 对应 `movq %rax,(%rbx)`，目标码 `48 89 03`（3 字节）位于 0x40059e；`objdump -d`／`gdb disassemble` 可反汇编任意可执行字节流（连 WINWORD.EXE 也行，但反汇编器需要知道起点）。
- **易错/考点**：mov 是「源在前、目的在后」而算术是「目的在前、源在后」，且 `leaq` 计算的是地址而非取内存内容（画面中部分反汇编字节不清晰）。
### Lecture 6 Machine Level Programming II Control（73.8 分钟）
- **主旨**：讲 x86-64 如何用条件码（condition code）和跳转/条件传送把 C 的 if、循环、switch 翻译成机器级控制流。
- **关键点**：
  1. 四个单比特条件码由算术指令**隐式**设置：CF（进位，无符号溢出）、ZF（结果为零）、SF（结果为负）、OF（补码有符号溢出），如 `addq Src,Dest` 对应 `t = a+b`，判断式写作 `(a>0 && b>0 && t<0) || (a<0 && b<0 && t>=0)`。
  2. `cmpq Src2,Src1` 计算 `Src1-Src2` 但不写目的操作数，`testq Src2,Src1` 计算 `a&b` 只置条件码，常用掩码方式判零/判负。
  3. SetX 指令只把目的寄存器的**低字节**置 0/1（`sete`←ZF、`setne`←~ZF、`setg`←~(SF^OF)&~ZF、`setl`←SF^OF、`seta`←~CF&~ZF），其余 7 字节不变，所以 `int gt(long x,long y)` 需 `cmpq %rsi,%rdi; setg %al; movzbl %al,%eax` 补齐。
  4. 老式条件分支用 `jle` 等实现 absdiff：`cmpq %rsi,%rdi; jle .L4; movq %rdi,%rax; subq %rsi,%rax; ret`，`.L4` 分支做 `subq %rdi,%rax`。
  5. `cmovXX`（条件传送）把 `if (Test) Dest ← Src` 变成无控制转移的指令，原因是分支对流水线（pipeline）破坏极大；GCC 只在能证明安全时才用。
  6. 循环统一改写：do-while 用 `loop: ... if(x) goto loop`；while 用 jump-to-middle（`goto test` 在 -Og 下）；for 先展开成 `Init; while(Test){Body; Update;}`；switch 用 jump table（跳转表），如 `jmp *.L4(,%rdi,8)` 配 `.quad` 表项，case 2 的 fall-through 用 `goto merge` 合并代码块。
- **易错/考点**：`setg/setl` 看 SF^OF（有符号），`seta/setb` 只看 CF 和 ZF（无符号），混用是经典错误。
### Lecture 7 Machine Level Programming III Procedures（66.7 分钟）
- **主旨**：讲 x86-64 过程（procedure）调用机制的三个要素——控制传递、数据传递、内存管理，以及栈帧（stack frame）与调用约定。
- **关键点**：
  1. x86-64 栈向**低地址**增长，`%rsp` 始终指向栈顶（最低地址）；`popq Dest` 先从 `%rsp` 取值、再 `%rsp` 加 8、最后写入目的寄存器（目的必须是寄存器）。
  2. `call label` 把返回地址（call 之后那条指令的地址）压栈并跳转，`ret` 弹出该地址并跳回；例子中 `400544: callq 400550 <mult2>` 压入的返回地址是 `0x400549`，`%rsp = 0x118`、`%rip = 0x400550`。
  3. 栈帧内容自上而下为：参数构造区（argument build）、局部变量、保存的寄存器上下文、可选旧 `%rbp`；caller 帧里放返回地址和参数，`call` 指令隐式完成压栈。
  4. `call_incr` 用 `subq $16,%rsp` 分配、`movq $15213,8(%rsp)` 存局部变量、`leaq 8(%rsp),%rdi` 传指针、`addq 8(%rsp),%rax` 取回、`addq $16,%rsp; ret` 释放，`%rax` 始终是返回值。
  5. 寄存器分为 caller-saved 与 callee-saved 两类：`call_incr2` 用 `pushq %rbx` 保存、结尾 `popq %rbx` 恢复，把跨调用要用的 x 放在 `%rbx` 里。
  6. 递归无需特殊机制：`pcount_r` 的 `pushq %rbx; movq %rdi,%rbx; andl $1,%ebx; shrq %rdi; call pcount_r; addq %rbx,%rax; popq %rbx` 靠栈帧天然获得私有存储，遵循后进先出（LIFO），互递归同样成立。
- **易错/考点**：`ret` 的地址来自栈顶而非某个寄存器，因此栈被破坏（如缓冲区溢出）时 `ret` 会跳到攻击者指定的地址。
### Lecture 8 Machine Level Programming IV Data（79.8 分钟）
- **主旨**：讲数组、嵌套数组、多级数组、结构体与浮点在机器级的表示与访问方式，重点是地址计算和内存对齐。
- **关键点**：
  1. 数组名即指向首元素的指针：`int val[5] = {1,5,2,1,3}`，元素地址为 `x+4i`，故 `val+1` 是 `x+4`、`&val[2]` 是 `x+8`、`val[5]` 已越界（值不可知）；取元素编译成 `movl (%rdi,%rsi,4), %eax`。
  2. 嵌套数组 `T A[R][C]` 按 row-major（行主序）连续存放，总大小 `R*C*K` 字节；`zip_dig pgh[4]` 等价于 `int pgh[4][5]`，先排完第 0 行再排第 1 行。
  3. 多级数组 `int *univ[3]` 是指针数组（每元素 8 字节），访问 `univ[index][digit]` 需**两次内存读**：`salq $2,%rsi; addq univ(,%rdi,8),%rsi; movl (%rsi),%eax`，即 `Mem[Mem[univ+8*index]+4*digit]`。
  4. 变长 n×n 数组必须做整数乘法：元素地址 `A + i*(C*K) + j*K`，编译为 `imulq %rdx,%rdi`（n*i）、`leaq (%rsi,%rdi,4),%rax`、`movl (%rax,%rcx,4),%eax`；定长 `fix_matrix` 因 N=16 可在编译期算好。
  5. 结构体按字段顺序排列，`struct rec {int a[3]; int i; struct rec *next;}` 中 i 在偏移 16、next 在偏移 24，链表遍历 `movslq 16(%rdi),%rax; movl %esi,(%rdi,%rax,4); movq 24(%rdi),%rdi`。
  6. 对齐规则是「地址必须是类型大小 K 的倍数」，`struct S1 {char c; int i[2]; double v;}` 实际偏移为 0、4、8、16、总大小 24，把大类型放前面（`int i; char c; char d;`）可把 12 字节压到 8 字节；浮点则走 SSE3 的 XMM 寄存器（16 个、每个 16 字节），`dincr` 用 `movapd %xmm0,%xmm1; movsd (%rdi),%xmm0; addsd %xmm1,%xmm0; movsd %xmm0,(%rdi)`。
- **易错/考点**：多级数组是「指针数组 + 各自独立的行」，与二维数组的连续布局不同，前者多一次内存读且行间不保证相邻。
### Lecture 9 Machine Level Programming V Advanced Topics（78.8 分钟）
- **主旨**：讲进程内存布局、缓冲区溢出（buffer overflow）漏洞的原理与代码注入攻击，以及 NX、栈金丝雀、ROP 等攻防手段，最后讲 union 与字节序。
- **关键点**：
  1. x86-64 Linux 内存布局自高到低为：Stack（运行时栈，8MB 限制，放局部变量）、Heap（`malloc/calloc/new` 动态分配）、Data（全局变量、static 变量、字符串常量）、Text/Shared Libraries（只读的可执行指令）。
  2. 溢出根源是 `gets()` 无法限制读入长度：`void echo() { char buf[4]; gets(buf); puts(buf); }` 编译成 `subq $24,%rsp; movq %rsp,%rdi; call gets`，buf 只占 4 字节，其后还有 20 字节未用空间和 8 字节返回地址；攻击者把可执行代码字节写进 buf 并一路覆盖 caller 帧的返回地址 A，`Q` 执行 `ret` 时就跳到 exploit code（输入超过 23 字节即可观察到溢出）。
  3. 代码级防御：用 `fgets(buf,4,stdin)` 替代 `gets`、`strncpy` 替代 `strcpy`，不要用 `scanf` 的 `%s`（改用 `fgets` 或 `%ns` 限定长度）。
  4. 系统级防御：x86-64 增加显式「execute」权限，栈被标记为不可执行（non-executable），注入的代码无法运行；`-fstack-protector` 在缓冲区与返回地址之间放 8 字节 canary，函数返回前用 `sub %fs:0x28,%rax` 比对，不符则 `call __stack_chk_fail`。
  5. ROP（return-oriented programming）绕过前两者：利用已有代码中以单字节 `0xc3`（ret）结尾的 gadget，如 `setval` 里 `4004d9: c7 07 d4 48 89 c7` 从中间 `0x4004dc` 进入即得到 `movq %rax,%rdi; retq`，把 gadget 地址串在栈上依次执行。
  6. `union` 按最大成员分配空间且同一时刻只能用其中一个字段（`char c; int i[2]; double v` 共 8 字节，三字段重叠在 up+0）；x86-64 是小端（little endian），最低有效字节放在最低地址。
- **易错/考点**：栈金丝雀只防「改到返回地址」，挡不住 ROP；NX 只防「执行栈上代码」，挡不住复用已有代码，两者互补而非替代。
---
