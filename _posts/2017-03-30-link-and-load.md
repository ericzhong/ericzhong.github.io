---
layout: post
title: 链接与装载
tags: 计算机系统
category: 计算机
---



#### C 源码的编译过程

预处理 (preprocess) > 编译 (compile) > 汇编 (assembly) > 链接 (link)

预处理：处理 `#include` 指令、展开 `#define` 宏定义、删除注释等等。

```bash
# 以下语句效果相同
gcc -E hello.c -o hello.i     # -E 仅预处理
cpp hello.c > hello.i
```

编译：产生汇编代码。

```bash
# 以下语句效果相同
gcc -S hello.i -o hello.s         # -S 仅预处理与编译
gcc -S hello.c -o hello.s
```

汇编：生成机器指令。

```bash
# 以下语句效果相同
as hello.s -o hello.o
gcc -c hello.s -o hello.o       # -c 编译&汇编但不链接 
gcc -c hello.c -o hello.o
```

链接：产生可执行文件。

```bash
ld -static /usr/lib/crt1.o /usr/lib/crti.o /usr/lib/gcc/i686-linux-gnu/x.x.x/crtbeginT.o -L/usr/lib/gcc/i686-linux-gnu/x.x.x -L/usr/lib -L/lib hello.o --start-group -lgcc -lgcc_eh -lc --end-group /usr/lib/gcc/i686-linux/gnu/x.x.x/crtend.o /usr/lib/crtn.o
```



#### 可执行文件的格式

Windows 的 `PE` (Portable Executable)、Linux 的 `ELF` (Executeable Linkable Format) 都是 `COFF` (Common File Format) 的变种。

还有 Linux 的 `a.out` 格式和 MS-DOS 的 `.COM` 格式。

目标文件 (Object File) 就是编译后还未链接的中间文件，比如 Windows 下的 `.obj` 和 Linux 下的 `.o` 文件。一般与可执行文件采用相同的格式存储。



#### ELF 文件类型、结构

ELF 文件类型：可重定位文件 (`.o`)、可执行文件、共享目标文件 (`.so`)、核心转储文件。使用 `file` 命令查看的信息中会包含 `relocatable|executable|shared object`。

ELF 文件结构：ELF 头、文本段、数据段、BSS段、... 、段表、段表字符串表、字符串表、符号表、... 。

ELF 头：ELF 魔数（数据存储方式、版本、运行平台、ABI 版本）、ELF 文件类型、CPU 平台类型、程序入口地址和长度、段表位置及包含的描述符数量、段表字符串表所在段在段表中的位置。

BSS (Block Started by Symbol) 段 ：只是为未初始化的全局变量和局部静态变量预留位置，没有内容，在文件中不占空间。

代码和数据分开存放的原因：1.权限不同。 2.代码要共享，而数据要私有。 3.提高代码的局部性，可提升缓存命中率。

段表 (Section Header Table)：是包含所有段的描述符的数组。段名的长度无法固定，所以全部存放到段表字符串表中，描述符中的 name 字段是段名在该表中的偏移。

符号表 (Symbol Table)：在链接中，函数和变量都叫符号。

字符串表 (String Table)：保存普通的字符串。

段表字符串表 (Section Header String Table)：保存段表中用到的字符串，比如段名。

`.init` 和 `.fini` 段：Glibc 初始化和退出时执行的代码，比如调用 C++ 的构造和析构函数。

```sh
readelf -S a.out          # -S Sections, 查看可执行文件的段
```



#### 特殊符号

ld 会用到的符号：

```c
__executable_start         // 程序起始地址（非入口地址）
__extext, _etext, extext   // 代码段结束地址
_edata, edata              // 数据段结束地址
_end, end                  // 程序结束地址
```



#### 符号修饰、函数签名

符号修饰 (Name Decoration)：早期 C 语言为了防止与现有库中的符号产生冲突，编译器都会在符号名前面加上类似下划线这样的修饰符号。Linux 下的 GCC 默认不修饰，可通过选项 `-fleading-underscore, -fno-leading-underscore` 来开关。

函数签名：面向对象语言的编译器为了防止符号冲突，会在符号中加入一些标识符，用来区别比如函数的返回值、参数类型、父类名称等等。



#### 在 C++ 中调用 C 库函数

```c++
#ifdef __cpluscplus
extern "C" {
#endif

...

#ifdef __cpluscplus
}
#endif
```

C++ 的编译器都会定义 `__cpluscplus`，然后 `extern "C" {...}` 中的符号就不会被修饰或签名。



#### 强符号、弱符号、COMMON 机制

强弱符号用于告诉链接器如果符号被重复定义了怎么办。

如果强符号碰到强符号会报错，强符号碰到弱符号会使用强符号，弱符号碰到弱符号则使用占用空间大的那一个。

对于 C/C++ 语言，编译器默认函数和初始化的全局变量为强符号 (Strong Symbol)，未初始化的全局变量为弱符号 (Weak Symbol)。

```c
# GCC
extern int ext;     // 外部符号既不是强符号也不是弱符号
int weak1;
int strong = 1;
__attribute__((weak)) weak2 = 2;
```

COMMON 机制：未初始化的全局变量被当做弱符号，符号类型显示为 COMMON。如果希望被当做强符号，可以使用 GCC 选项 `-fno-common` 关闭该机制，或者使用扩展：

```c
int global_var __attribute__((nocommon));
```



#### 强引用、弱引用

强弱引用用于告诉链接器如果符号没找到怎么办。

符号没找到默认是会报错的，如果被定义为弱引用 (Weak Reference)，则值为 0。使用前先判断一下，这样即便相应模块不存在，程序也可以运行。弱引用一般用来写插件。

```c
__attribute__((weakref)) void foo();

int main()
{
	if (foo) foo();
}
```



#### 调试信息格式

ELF 使用 `DWARF` (Debug With Arbitrary  Record Format) 格式，微软使用 `CodeView` 格式。

```sh
gcc -g         # 编译时产生调试信息
strip foo      # 去除 ELF 文件中的调试信息
```



#### BFD 库

Binary File Descriptor library，它是 binutils 项目的一个子项目。它是目标文件的一个抽象模型，通过它的统一接口可以访问不同格式的目标文件。GCC、ld、GDB 以及 binutils 的其它工具都是通过 BFD 库来处理目标文件的。



#### 链接步骤

* 空间与地址分配
* 符号解析与重定位。

```bash
ld a.o b.o -e main -o hello      # -e entry (即程序起始符号)
```

所以，起始函数不一定要用 main，可以自定义。

程序起始符号一般是 main，也是程序员视角的入口函数。但是链接器视角的入口符号是 `_start`，它会执行一些代码然后才调用 main() 函数。



#### 空间与地址分配

将所有目标文件的相似段合并，并将所有符号放到一个符号表中，重定位信息都放到重定位表中。

VMA、LMA：虚拟地址、加载地址，一般值是相同的。在嵌入式或 ROM 系统中可能不相同。



#### 符号解析与重定位

对所有符号进行重定位，也就是确定它们的虚拟地址。

在 Linux 下，ELF 可执行文件的地址默认从 `0x08048000` 开始分配。

```sh
objdump -r a.o         # -r relocation 查看重定位表
readelf -s a.o         # -s symbols 查看符号表 (UND 表示未定义符号，都是需要重定位的)
```



#### 静态库、静态链接

静态库是一组目标文件的组合，后缀一般为 `.a`。

静态链接就是从静态库中找到需要的所有符号并将内容全部链接到可执行文件中。

```sh
ar -t libc.a      # 列出静态库中的目标文件
ar -x libc.a      # -x extract 解压静态库
ar -cr libx.a a.o b.o c.o    # 生成静态库

gcc -static --verbose -fno-builtin hello.c    # 编译&静态链接

gcc -c -fno-builtin hello.c                   # 1.编译但不链接
ld -static -e main -o hello hello.o           # 2.静态链接
```

`-fno-builtin` 表示关闭内置优化，比如 GCC 为了提升运行速度会将 `printf` 替换为 `puts` 。



#### 链接控制

可以使用自定义的链接控制脚本来控制链接器的行为。

```sh
ld -verbose      # 查看默认链接控制脚本
ld -T link.lds   # 使用自定义的链接控制脚本
```



#### 建立进程的过程

1. 创建独立的虚拟地址空间
2. 读取可执行文件头，将可执行文件映射到虚拟地址空间。
3. 跳转到程序入口开始执行。



#### ELF 可执行文件映射到虚拟地址空间

链接程序会将具有相同权限的不同 Section 放到一起，然后将它们以一个 Segment 映射到一个虚拟内存区域 (VMA)。

ELF 可执行文件中有一个程序头表 (`Program Header Table`) 专门用来保存 Segment 的信息。比如每一个 Segment 的类型，偏移、大小、装载地址 (LMA)、权限、对齐属性等等。

环境变量与程序参数被压人栈的底部。栈内数据（从低到高）：参数数量、参数指针表、环境变量指针表、字符串。栈指针指向参数数量。

堆、栈都在单独的 VMA 中。

```sh
readelf -l hello     # 查看 segments
```



#### 动态链接程序的加载过程

1. 启动动态链接器
2. 装载所有需要的共享对象
3. 重定位和初始化



#### 地址无关代码 (PIC)

PIC，Position-Independent Code。

PIE，Position-Independent Executable。

静态编译是链接时重定位，而动态编译是装载时重定位，因此地址要到装载时才知道。为了代码段的共享性，重定位时不能将地址写死在代码中，要想办法转移到数据段。因此，不同情况的处理方式如下：

- 模块内部的调用或跳转：使用相对偏移，不需要重定位。
- 模块内部的数据访问：因为模块代码段和数据段的位置和大小都固定，指令在代码段中的位置固定，变量在数据段中的位置也固定，所以在编译阶段就可以将模块内部的数据访问指令转换成相对寻址的一组指令。所以这种情况下也不需要重定位。
- 模块间的数据访问：如果程序访问了共享库中的变量，肯定是需要重定位的，因为各个模块间的相对位置只有重定位后才知道。解决办法就是在本模块的数据段放置一个 GOT 表 (Global Offset Table)，每个外部模块的变量对应一项，在编译时就把 GOT 以及所有项的相对位置确定下来，这样就变成了模块内部的数据访问了，而重定位时只需要把变量最终的地址填充到 GOT 中，因为 GOT 在数据段，所以重定位不影响代码段。
- 模块间的调用或跳转：同上，只不过 GOT 表项中保存的是函数地址。

注意：共享对象的最终装载地址在编译时是无法确定的，重定位后才知道。

```sh
readelf -d foo.so | grep TEXTREL   # 判断 SO 是否 PIC (TEXTREL 是代码段重定位表，PIC 的 SO 中不会包含它)
gcc -fPIC -shared -o Lib.so Lib.c  # 编译 PIC 的共享对象
gcc -o hello hello.c ./Lib.so      # 动态链接
```



#### 延迟绑定

很多函数在程序执行完毕都不会用到，因此全部提前重定位很浪费时间。延迟绑定是在函数第一次被调用时才重定位。

原理就是在 GOT 访问上再增加一层 PLT (Procedure Linkage Table)，如果 GOT 中已经有地址则直接使用，如果没有就先装载到 GOT 再访问。



#### 动态链接器

Linux 下的动态链接器是 `ld-x.y.so`。

```bash
readelf -l a.out | grep interpreter       # 查看动态链接器的地址
```

这些信息存放在 ELF 文件的 `.interp` 段中。

通常是 `/lib/ld-linux.so.2 -> /lib/ld-x.y.so` 。



#### 查看 ELF 文件依赖的共享库

```sh
ldd a.out        # 查看依赖的共享库 (信息记录在 ELF 文件的 .dynamic 段中)
```



#### 运行时链接

由程序自己加载需要的共享对象，不需要时可以卸载。这种共享对象叫动态链接库 (DLL)。

```c
dlopen();     // 打开共享库文件
dlsym();      // 查找符号的地址
dlerror();    // 判断其它几个调用是否执行成功
dlclose();    // 关闭共享库
```



#### 共享库版本命名、SO-NAME 

共享库 `libname.so.x.y.z` 中的 x,y,z 分别表示主版本号、次版本号、发布版本号。

不同的主版本号之间是不兼容的，高的次版本号兼容低的，发布版本号之间完全兼容，不改接口。

SO-NAME 的命名规范是 `libname.so.x` ，它其实是指向共享库的软链接，没有次版本号和发布版本号。

```sh
ldconfig       # 更新所有共享库的软连接和缓存
```



#### 次版本号交汇问题

Minor-reversion Rendezvous Problem，即找到的共享库的次版本号比需要的低了，这时调用可能会出错。

可使用基于符号的版本机制 (Symbol Versioning) 解决，即通过一种方法标明函数的版本号。

```c
asm(".symver func, func@VERS_1.1");
int func() { ... }
```



#### 共享库的路径

按照 FHS (Filesystem Hierarchy Standard) 的规定，有三个地方可以存放共享库：

```sh
# 如果是 64 位的系统就将 lib 换成 lib64
/lib
/usr/lib
/usr/local/lib
```

如果上面的目录中不存在，就会去找 `/etc/ld.so.conf` 中配置的目录。

为了加速查找，有些信息放在缓存文件 `/etc/ld.so.cache` 中，因此无论修改/删除/创建了共享库文件或修改了配置文件都需要执行 `ldconfig` 命令来更新缓存和 SO-NAME。



#### 环境变量

`LD_LIBRARY_PATH`：临时更改某个应用程序的共享库查找路径，不影响其它程序。多个路径用 `:` 隔开。

`LD_PRELOAD`：可以预先载入的共享库或目标文件。优先级高于 LD_LIBRARY_PATH。它会覆盖后面的同名全局符号，可用于调试或测试。`/etc/ld.so.preload` 的作用相同。

`LD_DEBUG`：可以打开调试功能，打印出很多有用的信息。各种值的用途可查看帮助信息 `LD_DEBUG=help ./hello` 。



#### 创建共享库

```sh
# -shared 表示生成共享库
# -fPIC 使用地址无关代码
# -Wl,name,value 将 name=value 参数传给链接器
gcc -shared -fPIC -Wl,-soname,libfoo.so.1 -o libfoo.so.1.0.0 libfoo1.o libfoo2.o -lbar1 -lbar2
```

或者分开执行：

```bash
gcc -c -g -Wall -o libfoo1.o libfoo1.c
gcc -c -g -Wall -o libfoo2.o libfoo2.c
ld -shared -soname libfoo.so.1 -o libfoo.so.1.0.0 libfoo1.o libfoo2.o -lbar1 -lbar2
```

为链接器指定目标程序的共享库的查找路径：

```bash
ld -rpath /your/path -o hello hello.o -lsomelib
```



#### 清除符号信息

```bash
strip libfoo.so
ld -s     # 链接时清除所有符号
ld -S     # 链接时仅清除调试符号
```



#### 安装共享库

将它复制到标准共享库目录，比如 `/lib,/usr/lib` 等，然后运行 `ldconfig` 即可自动创建 SO-NAME 软链接并更新 cache。

编译时应该用 `-L,-l` 选项为 GCC 提供共享库的搜索目录和名称。

```sh
ldconfig
ldconfig -n shared_library_path   # 在指定目录下找
```



#### 共享库的构造和析构函数

如果希望共享库在被装载时做些初始化工作，卸载时做些清理工作，需要为函数加上声明：

```c
void __attribute__((constructor)) init_function(void);
void __attribute__((destructor)) fini_function(void);
```

其原理就是 GCC 让这两个函数分别在 main 函数的之前和之后执行。

还可以指定多个构造和析构函数：

```c
void __attribute__((constructor(1))) init_function1(void);
void __attribute__((constructor(2))) init_function2(void);
```

构造函数的执行顺序是从数小的到数大的，析构函数则正好相反。



#### 共享库脚本

共享库 (.so) 文件可以是一个脚本文件，它列出了一组依赖的共享库文件：

```
GROUP( /lib/libc.so.6 /lib/libm.so.2 )
```



#### 程序的内存布局

```
0xc0000000  内核空间 
            栈
0x40000000  动态库
            堆
            读写区 (.data, .bss)
0x08048000  只读区 (.init, .rodata, .text)
```

程序依赖的动态库会被载入到堆栈区中间的一个地方。



#### 栈帧 (Stack Frame) 结构

保存了函数调用需要维护的信息：

- ebp -> 函数的返回地址、参数
- esp -> 编译器保存的一些数据、局部变量、寄存器上下文、Old EBP



#### 调用惯例 (Calling Convention)

- 函数参数的传递顺序和方式：如果通过栈传递参数，参数入栈方向是从左至右还是相反。也可以通过寄存器传递参数。
- 栈的维护方式：出栈工作由函数还是调用方完成。
- 名字修饰策略

C 语言的默认调用惯例是 `cdecl`，从左至右参数入栈、函数调用方负责出栈、在函数名前加下划线来修饰名字。

常见调用惯例有 cdecl、stdcall、fastcall、pascal。



#### 传递函数返回值

函数返回值放在 eax 寄存器中，如果超过 4 个字节，则放到 [edx + eax] 中。更大的则用 eax 返回结构体的指针，结构体存放在栈中，位置是函数地址和参数的上面。



#### 堆空间分配与维护

堆空间的维护一般由运行库 (Glibc) 完成。Linux 下有两种堆分配方式：`brk()`、`mmap()`。

`brk()` 可以调整数据段（包含 BSS）的结束地址。

`mmap()` 可以向内核分配虚拟地址空间，当它不用于映射文件时就叫匿名空间，可以当堆空间用。

Glibc 对小于 128K 的请求会使用现有的堆空间，对大于 128K 的请求的则会使用 `mmap()` 分配一块匿名空间。



#### 堆分配算法

常用的方法有空闲链表、位图、对象池。

空闲链表：每一块区域可以任意大小，最开始的地方有两个指针，分别指向空闲的 next 和 prev 块。

位图：将空间划分为相同大小，使用位图表示占用状态。

对象池：通常分配的对象都是较为固定的几个值，可以预先分配好空间。对象池既可以用空闲链表也可以用位图实现。

Glibc 对小于 64 字节的请求从对象池分配。 



#### 程序入口

真正的程序入口并不是 main 函数，而是程序的初始化部分，它往往是运行库的一部分，地址由 ELF 可执行文件的 entry 指定。

Glibc 程序的入口为 `_start`。



#### C 运行库

ANSI C 是 C 语言的标准库。

Glibc 是 GNU 的 C 标准库，有动态和静态两个版本。

Glibc 的启动文件：

- `crt1.o`：包含程序入口函数 `_start`，并由它调用 main 函数。
- `crti.o` 和 `crtn.o`：分别包含 `_init()` 和 `_fini()` 函数的开始和结尾部分。即 C++ 中的构造和析构的入口。

链接的格式一般是：

```bash
ld crt1.o crti.o [用户的目标文件] [系统库] crtn.o
```



GCC 平台相关文件：`crtbeginT.o` 和 `crtend.o` 才是真正用于 C++ 构造和析构的目标文件。`libgcc_eh.a` 包含 C++ 异常处理相关函数。`libgcc.a` 为跨平台相关的函数，`libgcc.so` 为它的动态链接版本。



#### 线程局部存储 (TLS)

即每个线程都有属于自己的私有数据。

> 标准库不包含线程相关内容，它跟平台相关。

```c
#define errno (*__errno_location ())        # Glibc 的实现方式

char *strtok(...);
char *strtok_s(..., char **context);        # Glibc 提供的线程安全版本

__thread int number;                        # GCC 把它当做 TLS 全局变量
```



#### Linux 系统调用

在 x86 下，系统调用由 0x80 中断完成，通过通用寄存器传递参数，eax 传递系统调用号，当系统调用返回时，返回值存入 eax。

查看系统调用的手册：`man 2 xxx`

堆栈切换：进入系统调用或者返回时，都需要在堆栈的内核态和用户态之间切换，并复制数据，因为它们运行在不同的特权级。

新型系统调用指令：`sysenter` 和 `sysexit` 是由处理器支持的新型系统调用指令。