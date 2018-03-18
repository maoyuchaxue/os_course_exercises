# lec5 SPOC思考题

##**提前准备**
（请在上课前完成）

- 完成lec５的视频学习和提交对应的在线练习
- git pull ucore_os_lab, v9_cpu, os_course_spoc_exercises in github repos。这样可以在本机上完成课堂练习。
- 理解连续内存动态分配算法的实现（主要自学和网上查找）

NOTICE
- 有"hard"标记的题有一定难度，鼓励实现。
- 有"easy"标记的题很容易实现，鼓励实现。
- 有"midd"标记的题是一般水平，鼓励实现。


## 思考题
---

## “连续内存分配”与视频相关的课堂练习

### 5.1 计算机体系结构和内存层次

1.操作系统中存储管理的目标是什么？

目标是：

+ 抽象，把各进程的线性地址转化成物理地址；
+ 保护，各进程只被允许访问自己权限内的数据；
+ 共享，各进程之间可以共享一部分数据。
+ 虚拟化，进程内部使用的是更便于其使用的虚拟地址，例如各进程内相同的线性地址空间。

### 5.2 地址空间和地址生成
1.描述编译、汇编、链接和加载的过程是什么？

+ 编译是将源代码转成汇编代码的过程；
+ 汇编试图将汇编代码转成机器码，但其中有一些文件外引用的符号由于汇编单个文件时无法知晓其具体位置或地址，因此需要将这些符号信息保留下来；
+ 链接是将多个源文件拼合成二进制文件，将之前未确定地址的符号均确定下来，同时还可能和一些已有库进行拼接；
+ 加载是将链接完后的二进制文件加载入内存的某处连续地址准备运行，为了使其地址正确，需要更改其中的全部绝对地址，使其符合起始地址对应的偏移量。

2.动态链接如何使用？尝试在Linux平台上使用LD_DEBUG查看一个C语言Hello world的启动流程。  (optional)

动态链接可以通过`gcc -shared -fPIC`编译生成，然后在使用时用`gcc -L`命令指定相应的动态链接库文件即可使用。其本质是生成地址无关代码，与外界交互的地址通过在其数据段设下全局数组来完成。这样外部就可以随时加载，并以开销不大的方式使用。

在Linux上写一个简单的Hello world程序，命名为`hw`，用`LD_DEBUG=all ./hw`命令观察其启动流程:

```
     15300:	
     15300:	file=libc.so.6 [0];  needed by ./hw [0]
     15300:	find library=libc.so.6 [0]; searching
     15300:	 search cache=/etc/ld.so.cache
     15300:	  trying file=/lib/x86_64-linux-gnu/libc.so.6
     15300:	
     15300:	file=libc.so.6 [0];  generating link map
     15300:	  dynamic: 0x00007f12c9715ba0  base: 0x00007f12c9357000   size: 0x00000000003c52c0
     15300:	    entry: 0x00007f12c9378fd0  phdr: 0x00007f12c9357040  phnum:                 10
     15300:	
     15300:	checking for version `GLIBC_2.2.5' in file /lib/x86_64-linux-gnu/libc.so.6 [0] required by file ./hw [0]
     15300:	checking for version `GLIBC_2.3' in file /lib64/ld-linux-x86-64.so.2 [0] required by file /lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	checking for version `GLIBC_PRIVATE' in file /lib64/ld-linux-x86-64.so.2 [0] required by file /lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	
     15300:	Initial object scopes
     15300:	object=./hw [0]
     15300:	 scope 0: ./hw /lib/x86_64-linux-gnu/libc.so.6 /lib64/ld-linux-x86-64.so.2
     15300:	
     15300:	object=linux-vdso.so.1 [0]
     15300:	 scope 0: ./hw /lib/x86_64-linux-gnu/libc.so.6 /lib64/ld-linux-x86-64.so.2
     15300:	 scope 1: linux-vdso.so.1
     15300:	
     15300:	object=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	 scope 0: ./hw /lib/x86_64-linux-gnu/libc.so.6 /lib64/ld-linux-x86-64.so.2
     15300:	
     15300:	object=/lib64/ld-linux-x86-64.so.2 [0]
     15300:	 no scope
     15300:	
     15300:	
     15300:	relocation processing: /lib/x86_64-linux-gnu/libc.so.6 (lazy)
     15300:	symbol=_res;  lookup in file=./hw [0]
     15300:	symbol=_res;  lookup in file=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	binding file /lib/x86_64-linux-gnu/libc.so.6 [0] to /lib/x86_64-linux-gnu/libc.so.6 [0]: normal symbol `_res' [GLIBC_2.2.5]
     15300:	symbol=stderr;  lookup in file=./hw [0]
     15300:	symbol=stderr;  lookup in file=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	binding file /lib/x86_64-linux-gnu/libc.so.6 [0] to /lib/x86_64-linux-gnu/libc.so.6 [0]: normal symbol `stderr' [GLIBC_2.2.5]
     
     ....................

     15300:	symbol=realloc;  lookup in file=./hw [0]
     15300:	symbol=realloc;  lookup in file=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	binding file /lib64/ld-linux-x86-64.so.2 [0] to /lib/x86_64-linux-gnu/libc.so.6 [0]: normal symbol `realloc' [GLIBC_2.2.5]
     15300:	symbol=free;  lookup in file=./hw [0]
     15300:	symbol=free;  lookup in file=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	binding file /lib64/ld-linux-x86-64.so.2 [0] to /lib/x86_64-linux-gnu/libc.so.6 [0]: normal symbol `free' [GLIBC_2.2.5]
     15300:	
     15300:	calling init: /lib/x86_64-linux-gnu/libc.so.6
     15300:	
     15300:	symbol=__vdso_clock_gettime;  lookup in file=linux-vdso.so.1 [0]
     15300:	binding file linux-vdso.so.1 [0] to linux-vdso.so.1 [0]: normal symbol `__vdso_clock_gettime' [LINUX_2.6]
     15300:	symbol=__vdso_getcpu;  lookup in file=linux-vdso.so.1 [0]
     15300:	binding file linux-vdso.so.1 [0] to linux-vdso.so.1 [0]: normal symbol `__vdso_getcpu' [LINUX_2.6]
     15300:	symbol=__libc_start_main;  lookup in file=./hw [0]
     15300:	symbol=__libc_start_main;  lookup in file=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	binding file ./hw [0] to /lib/x86_64-linux-gnu/libc.so.6 [0]: normal symbol `__libc_start_main' [GLIBC_2.2.5]
     15300:	
     15300:	initialize program: ./hw
     15300:	
     15300:	
     15300:	transferring control: ./hw
     15300:	
     15300:	symbol=printf;  lookup in file=./hw [0]
     15300:	symbol=printf;  lookup in file=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	binding file ./hw [0] to /lib/x86_64-linux-gnu/libc.so.6 [0]: normal symbol `printf' [GLIBC_2.2.5]
     15300:	symbol=__tls_get_addr;  lookup in file=./hw [0]
     15300:	symbol=__tls_get_addr;  lookup in file=/lib/x86_64-linux-gnu/libc.so.6 [0]
     15300:	symbol=__tls_get_addr;  lookup in file=/lib64/ld-linux-x86-64.so.2 [0]
     15300:	binding file /lib/x86_64-linux-gnu/libc.so.6 [0] to /lib64/ld-linux-x86-64.so.2 [0]: normal symbol `__tls_get_addr' [GLIBC_2.3]
     15300:	
     15300:	calling fini: ./hw [0]
     15300:		
hello world
```

中间我省略了大量的符号绑定信息。可见当执行程序时，首先是加载了libc.so等库，随后进行了动态链接库间的符号查找和链接，最后才是将执行权交到我们的程序手上，输出了Hello world.


### 5.3 连续内存分配
1.什么是内碎片、外碎片？

+ 内碎片是由于我们一般将内存按照块状分配，对申请的内存需要按照块的大小进行上取整，这样就产生了内存块中的碎片。

+ 外碎片是由于多次申请、释放内存，导致完整的空闲内存区间中几小块被占用，导致剩下的空间虽然总量足够但各自分散，不能组成其他进程所需要的连续的内存空间，这些分散的小空间即为外碎片。

2.最先匹配会越用越慢吗？请说明理由（可来源于猜想或具体的实验）？

从理论上来说，最初内存中没有外碎片，但随着进程不断申请、释放内存，自然会产生外碎片；此时空闲空间链表当然会增长，每次查询的时间也会相应增加，故会越用越慢。不过，碎片数量不可能无限制增长下去，随着释放时相邻空闲空间被合并，最终碎片量会保持稳定，速度也会保持稳定。

3.最差匹配的外碎片会比最优适配算法少吗？请说明理由（可来源于猜想或具体的实验）？

根据JE. Shore的[这篇文章](https://dl.acm.org/citation.cfm?id=360949)中的实验，最差匹配的效率是最低的；它每次选取最大的一块，会使得完整的空间被迅速拆分成小块，导致更多的外碎片。

4.理解0:最优匹配，1:最差匹配，2:最先匹配，3:buddy systemm算法中分区释放后的合并处理过程？ (optional)

+ 最优匹配、最差匹配是相似的，首先需要遍历所有当前已有的空闲区间，若已有空闲区间恰好可以和新被释放的空间拼接，则和之合并成一个空闲区间。由于理论上来说，由于每次释放都会进行同样的操作，因此链表内部不会存在本身就可以合并的空闲空间。因此，只需要检查新区间前后是否可以直接拼接即可。最后，形成的新区间需要根据其大小被放入链表中适当的地方。
+ 最先匹配需要找到新空间地址所处的位置，并观察它和前一个、后一个元素对应的空间能否合并。合并完之后就置于原地即可。
+ buddy system释放空间后，需要去检查与其对应的伙伴（即由一个共同的块分裂而成的两块空间中的另一块）是否也处于空闲状态，若是，则将两者合并，并继续向上层检查。最后，将形成的大块放入正确的链表中。

### 5.4 碎片整理
1.对换和紧凑都是碎片整理技术，它们的主要区别是什么？为什么在早期的操作系统中采用对换技术？ 

+ 对换是将一个进程，或其中的部分程序数据换出到外存当中，以空出内存空间供其他程序使用。
+ 紧凑是将已有的内存空间进行移动，使其占用连续空间，使得外碎片拼合成较大的连续内存。

早期硬件上内存比较紧张，多个进程共同使用未必足够，紧凑只能消除碎片但总剩余内存没有变化，不行；并且早期对速度的要求没有那么苛刻，可以忍受从磁盘高频移入移出的操作，所以使用对换。

2.一个处于等待状态的进程被对换到外存（对换等待状态）后，等待事件出现了。操作系统需要如何响应？

操作系统需要把这个进程标记为挂起就绪状态，接下来调度时，若没有活跃就绪态的进程，或是该进程优先级比活跃就绪态进程都高时，把它换入并执行。

### 5.5 伙伴系统
1.伙伴系统的空闲块如何组织？

组织成二维数组，第一维是空闲块大小，第二维按空闲块地址排序。

2.伙伴系统的内存分配流程？伙伴系统的内存回收流程？

分配时，选择尽量小的、大小达到所需内存的空闲块；若其大小超过需求一倍，则将其拆分为相等的两块；继续拆分，直至无法拆分为止，将这一块空闲区间分给进程。其中，每段拆出的空闲区间分别加入相应的空闲区间序列中。

回收流程在题5.3.4的答案中已讲述过。

## 课堂实践

观察最先匹配、最佳匹配和最差匹配这几种动态分区分配算法的工作过程，并选择一个例子进行分析分析整个工作过程中的分配和释放操作对维护数据结构的影响和原因。

  * [算法演示脚本的使用说明](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep3-malloc.md)
  * [算法演示脚本](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep3-malloc.py)

例如：
```
python ./ostep3-malloc.py -S 100 -b 1000 -H 4 -a 4 -l ADDRSORT -p BEST -n 5 -c
python ./ostep3-malloc.py -S 100 -b 1000 -H 4 -a 4 -l ADDRSORT -p FIRST -n 5 -c
python ./ostep3-malloc.py -S 100 -b 1000 -H 4 -a 4 -l ADDRSORT -p WORST -n 5 -c
```

### 扩展思考题 (optional)

1. 请参考xv6（umalloc.c），ucore lab2代码，选择四种（0:最优匹配，1:最差匹配，2:最先匹配，3:buddy systemm）分配算法中的一种或多种，在Linux应用程序/库层面，用C、C++或python来实现malloc/free，给出你的设计思路，并给出可以在Linux上运行的malloc/free实现和测试用例。


2. 阅读[slab分配算法](http://en.wikipedia.org/wiki/Slab_allocation)，尝试在应用程序中实现slab分配算法，给出设计方案和测试用例。
