# lab0 SPOC思考题

##**提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在linux, ucore, v9-cpu中是如何具体体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？

为了实现进程的切换，硬件上必须有中断机制，并且有时钟中断，否则将无法实现进程的轮流调度。这样为了处理中断，也需要有与中断有关的特权指令，例如获取eflag，中断号，开关中断，设置中断向量表地址等。
为了实现虚存，至少需要有分页或分段机制。硬件需要支持分段/分页的页表查找、地址映射等。需要有指令用于设置段页表地址，另外，由于机器不可能一上电就直接获得段页表，所以还需要能够开启、关闭段页功能，以便于初始化。
为实现文件系统，需要有能够实现存储的外设，以及进行地址映射、完成数据传输的总线。为了和外设进行交互，需要in/out输入输出指令，若使用异步的外设，可能也需要中断的支持。

- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？

实模式可以看作是未开启主要功能的初始状态，地址映射为单纯的段地址+偏移形式，没有特权级检查，偏移地址也仅为16位。
保护模式中使用段描述符，对各段设置了特权级等，可以检查不允许的内存访问/指令执行。可以支持开启分页，可以使用32位地址。

物理地址是与外设交互时使用的地址，即经过段、页地址转换后得到的真实的地址。
逻辑地址是程序中所使用的地址，是初始的、未经转换的地址。
线性地址是逻辑地址经过分段机制的转化（即添加了偏移）之后，但未经过分页地址转换得到的地址。

- 理解list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```

:后表示该字段所占的bit数，是c中用于定义紧凑结构的特殊语法。

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？

这段指令语法不正确，intr为unsigned类型无法直接转为gatedesc类型，将会报错。可以修改为：
```
unsigned intr;
intr=8;
gatedesc d = *((gatedesc*)&intr);
SETGATE(d, 1,2,3,0);
intr = *((unsigned*)&d);
```
用指针进行强制转换即不会出现类型问题了。
所得到的指针指向地址空间中的连续64bit值为：
0000 0000 0000 0000 0100 1111 0000 0000
0000 0000 0000 0002 0000 0000 0000 0011
由于被转为unsigned int,故该数中只包括末32位，其值为0x00020003.

### 课堂实践练习

#### 练习一

请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。

  - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)

  - ##### [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)

  - ##### [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)

  - ##### [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)

  - ##### [[IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)]

__memset函数实现如下:

```
__memset(void *s, char c, size_t n) {
    int d0, d1;
    asm volatile(
        "rep; stosb;"
        : "=&c" (d0), "=&D" (d1)
        : "0" (n), "a" (c), "1" (s)
        : "memory");
    )
    return s;
}
```
这一段使用汇编自带的循环实现高效的memset.其汇编后生成的代码为：
```
mov -0x10(%ebp), %ecx
mov -0x9(%ebp), %eax
mov -0x8(%ebp), %edx
mov %edx, %edi
rep stos %al, %es:(%edi)
mov %edi, %edx
mov %ecx, -0x14(%ebp)
mov %edx, -0x18(%ebp)
```

rep指令中，ecx用于指定重复次数，为计数器；stos指令用于将eax内的数据存入%es:(%edi)地址中。这两者结合即完成了memset操作。这里内联汇编将三个输入参数指定到了对应的寄存器里，并且"memory"指明了操作会更改内存。


#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore中宏定义的用途，并举例描述其含义。

 > 利用宏进行复杂数据结构中的数据访问；
 > 利用宏进行数据类型转换；如 to_struct, 
 > 常用功能的代码片段优化；如  ROUNDDOWN, SetPageDirty

宏定义包括:
1. 指明常用常数，例如把DPL\_USER = 3, DPL\_KERNEL = 0, SEG\_UDATA = 4, T\_SYSCALL = 0x80等，其中部分和硬件要求直接相关，部分和操作系统里设计的约定有关，作为实现它的代码，不应将这些约定和硬件接口一并包含进去，所以用宏定义是妥当的。
2. 用宏实现内联函数，根据个人阅读感受，最常用的实际上是位运算。例如PDX(la),PTX(la)，实际上即取线性地址中的最高10位/中间10位，这样避免每次都写位运算，让代码重复。不过，基本上实现简单、功能清晰的函数均可使用，包括但不限于：
+ 函数的简写或扩展，如alloc\_page() => alloc\_pages(1)，让代码更简洁。
+ 简单的数学函数，如ROUNDDOWN, ROUNDUP用于向上下取整，可以用于内存对齐。又如USER\_ACCESS(start, end)，用于简单地检查访问区域是否在合法范围内。
3. 用宏实现常规c代码无法实现的功能。实际上目前看到的唯一使用例是to\_struct以及offsetof。

其中前两个用法并无特别之处，换成常量和内联函数也一样可以从功能上替代之，但最后一个不同，如offsetof(type, member) => ((size_t)(&((type *)0)->member))，传入的member参数被直接作为字符串替换进代码中，实现了类似于python的obj["field"]的简便功能，这是应用了宏提供的过于强大的功能的例子。（当然这样很容易导致代码失去严谨性，不过少量的使用影响不大。）