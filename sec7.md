# 链接（L13—L13）

> 一个 .c 文件变成可执行文件中间发生了什么：符号解析、重定位、静态库与动态库的取舍。

### Lecture 13 Linking（81.5 分钟）
- **主旨**：讲 linker（链接器）如何把多个可重定位目标文件合并成可执行文件，核心是 symbol resolution（符号解析）与 relocation（重定位），以及静态库、共享库与 interposition（插桩）机制。
- **关键点**：
  1. ELF 目标文件含 ELF header（字长、字节序、文件类型 .o/exec/.so）、segment header table、.text、.rodata（只读数据，如跳转表）、.data（已初始化全局变量）、.bss（未初始化全局变量，"Block Started by Symbol"）、.symtab、.rel.text/.rel.data 等节。
  2. 链接器符号分三类：global（本模块定义、可被其它模块引用）、external（本模块引用、他处定义）、local（static 函数与 static 全局变量，仅本模块可见，与程序内的局部变量无关）。
  3. 符号分 strong（过程与已初始化全局变量）与 weak（未初始化全局变量）：两个同名 strong 直接 link time error，strong 会覆盖 weak。
  4. 重定位把 .o 内的相对位置改成最终绝对地址：main.o 中 `mov $0x0,%edi`（R_X86_64_32 array）被填为 0x601018，`callq` 的 R_X86_64_PC32 sum 填为 0x4004f8（= 0x4004e3 + 5）。
  5. 静态库 .a 由 ar 归档，链接器按命令行顺序扫描、只抽取能解决当前未解析符号的成员，故库须放在命令行末尾（`gcc -L. -lmine libtest.o` 报 undefined reference）。
  6. 共享库用 dynamic linking：`gcc -shared` 生成 .so，链接期只做部分链接（partially linked），运行时由 ld-linux.so 重定位；也可 dlopen/dlsym/dlclose 动态加载，或用 LD_PRELOAD 做 run-time interposition。
- **易错/考点**：同名全局变量类型不一致（`int x` 与 `double x` 并存）会互相覆盖内存（幻灯片标为 Evil/Nasty），应尽量 static、定义时初始化、引用外部变量用 extern。
---
