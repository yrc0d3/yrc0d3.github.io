# 《Linux Kernel Programming》读书笔记-第二部分


## Kernel Internal Essentials - Processes and Threads
### Understanding process and interrupt contexts
宏内核(monolithic)意味着：当进程或线程进行系统调用时，会切换到内核模式然后执行内核代码，也可能操作内核数据。

内核代码会在两种context中执行：

* Process context：通过系统调用或者异常进入内核态，是同步的(top down)
* Interrupt context：通过硬件中断进入内核态，是异步的(bottom up)

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_1.png)

图6.1中也展示了纯内核线程，它们只在内核空间执行，并且看不到用户VAS。

### Understanding the basics of the process VAS
![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_2.png)

从[《Anatomy of a Program in Memory》](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)上找来的图（需要在chrome里设置允许展示混合内容，参考[《Chrome 禁止混合内容的解决办法》](https://sspai.com/post/63507)）：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/linux_x32_process_vas_layout.png)

linux进程包含的mappings (or segments)：

* Text segment：static
* Data segment：
   * Initialized data segment：static
   * Uninitialized data segmeng：也叫bss，static
   * Heap segmeng：dynamic。通过`malloc` 分配的空间，小于`MMAP_THRESHOLD` (默认128KB)会直接在堆上分配，大于的会在进程VAS的一块单独的mapping空间分配(通过`mmap`系统调用)。
* Libraries (text, data)：dynamic
* Stack：dynamic

图6.2中，main、thrd2、thrd3这三个线程，除了各自的stack是独立的之外，其他都是共享的。

本书作者提供了一个命令行查看进程VAS的可视化工具[https://github.com/kaiwan/procmap](https://github.com/kaiwan/procmap)

### Organizing processes, threads, and their stacks - user and kernel space
> A thread is merely an execution path within a process.
Threads share all process resources except for the stack.
The thread, not the process, is the kernel schedulable entity (aka KSE).

linux中，每个线程都对应内核中的一个task structure (aka process descriptor)。

> we require one stack per thread per privilege level supported by the CPU.

因此，linux中每个用户空间线程都有两个栈，一个用户空间栈和一个内核空间栈。当然，内核空间的线程只有一个内核空间的栈。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_3.png)

#### user space organization
![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_4.png)

用户态进程包含：

* Text: r-x
* Data Segment: rw-
* LIbrary mappings
* Downward-growing stack(s)

图6.4中的P1是单线程的进程，只有一个main()线程

图6.4中的P2、Pn是多线程的进程，main()线程在调用`fork`时分配，其他线程在调用`pthread_create`时分配。

用户空间的线程虽然时动态分配的，但是大小有限制，即`RLIMIT_STACK` ，一般是8MB。

#### kernel space organization
![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_5.png)

每个用户态线程都有一个对应的内核态的栈，加上用户态的栈，一共有两个栈，如图6.5中的P1、P2和Pn。

每个内核态线程只有一个内核态的栈，如图6.5中的kthrd1、kthrd2和kthrdn。

内核态的栈都是固定大小的，并且很小。一般来说，32位机器上为2页大小，64位机器上为4页大小，一页一般是4KB。

很多架构（x86和ARM64等）都支持一个单独的CPU绑定栈用于处理中断。当外部硬件中断发生时，CPU的控制单元会立刻切换到中断处理器，这个栈就是用来保存中断处理器代码执行路径的。

### Viewing the user and kernel stacks
#### 传统方式
内核态线程：

`/proc/<pid>/stack` 

```bash
$ sudo cat /proc/2075/stack 
[<0>] __switch_to+0x100/0x150
[<0>] do_wait+0x298/0x2b8
[<0>] kernel_wait4+0x9c/0x268
[<0>] __do_sys_wait4+0xa4/0x1c0
[<0>] __arm64_sys_wait4+0x30/0x40
[<0>] el0_svc_common.constprop.0+0xf4/0x200
[<0>] el0_svc_handler+0x38/0xa8
[<0>] el0_svc+0x10/0x180
```
每一行都表示一个栈帧。最上面的是栈顶，最下面的是栈底。

每一行最左面的`[<0>]` 是代码的地址，出于安全原因，都被显示为0。

用户态线程：

要困难一些，一些linux发行版中可以安装gstack或gstack（原理都是用gdb）。

```bash
$ pstack 7013
#0  0x00000039958c5620 in __read_nocancel () from /lib64/libc.so.6
#1  0x000000000047dafe in rl_getc ()
#2  0x000000000047def6 in rl_read_key ()
#3  0x000000000046d0f5 in readline_internal_char ()
#4  0x000000000046d4e5 in readline ()
#5  0x00000000004213cf in ?? ()
#6  0x000000000041d685 in ?? ()
#7  0x000000000041e89e in ?? ()
#8  0x00000000004218dc in yyparse ()
#9  0x000000000041b507 in parse_command ()
#10 0x000000000041b5c6 in read_command ()
#11 0x000000000041b74e in reader_loop ()
#12 0x000000000041b2aa in main ()
```
无法解释的栈会写作??

#### 现代化方式-\[e\]BPF
4.X开始的内核支持\[e\]BPF。通过BCC或bpftrace来使用会方便些。

可以按照这个步骤安装一下BCC：[https://github.com/iovisor/bcc/blob/master/INSTALL.md](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

BCC提供了`stackcount` (ubuntu上叫`stackcount-bpfcc`)工具可以同时查看内核栈和用户栈。

### 进程VAS概述
![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_7.png)

### task structure
task structure代码位于`include/linux/sched.h:struct task_struct` 

linux中，每个进程都包含一个或多个线程，每个线程都对应一个内核数据结构task structure。

图6.8展示了task structure包含的主要内容：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_8.png)

所有的task structrue对象都存放在内核内存中的叫做task list的双向循环链表中。

在内核中可以通过`current`宏(`linux/sched.h`)来获得当前线程的task\_struct指针。

内核代码会在process (or task) context和interrupt (or atomic) context之间切换，因此，编写内核或驱动代码时要判断当前处于哪个context，可以使用`in_task`宏(`linux/preempt.h`)来判断当前是否处于process context。

`current`宏只在process context中才有效。

最好通过内核中的内建的helper methods来获取task\_struct的成员变量，比如通过`task_pid_nr()`来获取PID，而不是直接通过`current->pid` 这种方式来获取。这些方法都在`include/linux/sched.h` 中。

要遍历所有进程，可以使用`for_earch_process`宏（只遍历每个进程的`main()`线程）：

```cpp
// include/linux/sched/signal.h
#define for_each_process(p) \
    for (p = &init_task; (p = next_task(p)) != &init_task; )
```
要遍历所有线程，可以使用`do_each_thread() {...} while_each_thread()`这一对宏。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_10.png)

图6.10中，线程名称用方括号包起来的表示这是内核线程。

在linux内核中，每个线程都有一个独有的PID。这违反了POSIX.1b中的pthreads标准（同一进程的不同线程共享同一个PID）。为了解决这个问题，2.5版本linux内核中的task\_struct加入了**Thread Group IDentifier** (TGID)：对于单线程的进程，其tgid和pid相同。对于多线程的进程，main线程的tgid和pid相同；其他线程的tgid等于main线程的tgid，但pid各不相同。可以结合图6.10中的VBoxService来理解。

通过通用的GNU ps(1)命令也能查看TGID和PID，需要使用`-LA`选项：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_6_ps.png)

如图所示，第一列PID表示`task_struct`中的tgid，第二列的LWP (LightWeight Process or thread)表示`task_struct` 中的pid。

## Memory Management Internals - Essentials
### 从Hello World开始
```cpp
printf("Hello, world.\n");
```
`printf(3)` 代码是写在libc里的。我们的程序在启动时，loader会检测需要的共享库，然后将它们映射到当前进程的VAS中（具体知识涉及动态链接、位置无关代码等）。然后就能调用到了。

通过`ldd(1)` 命令可以查看程序依赖的共享库。

通过`strace(1)` 命令可以查看程序执行中的系统调用命令。

`printf(3)` 会被转换成`write(2)`系统调用，把字符串写到stdout中。那么程序是如何访问内核代码的呢？原理就是用户和内核VAS存在于同一个VAS中，通过一种叫做VM Split的技术，将可用空间按照`u:k`的比例分配给user和kernel。下图展示的是按照`2:2` 比例划分的32位linux系统（Arm-32系统一般都是`2:2`，x86-32一般是`3:1`）：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_1.png)

### VM Split on 64-bit Linux systems

首先，64位并不会全部用于寻址。一般来讲，x86\_64的linux系统的页大小为4KB，用48位(Least Significant Bit, LSB)来寻址。原因就是64位用于寻址的话太多了，用不到那么多。

在用户空间用`printf("address of i is 0x%x\n", &i);` 这种方式打印地址的话， 展示的是UVA(User Virtual Address)。

在内核空间用`printk()` 打印地址的话，展示的是KVA(Kernal Virtual Address)。

virtual address并不是一个从0开始的绝对地址，而是一个bitmask（MMU负责转换）。图7.2展示的是x86\_64的Linux中的虚拟地址：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_2.png)

可以看到，KVA地址的16位Most Significant Bit(MSB)都是1，UVA的MSB都是0。因此，KVA格式都是`0xffff .... .... ....` ，UVA格式都是`0x0000 .... .... ....`。

48位LSB的划分是和CPU的MMU(Memory Management Unit)相关的:

* PGD: Page Global Directory
* PUD: Page Upper Directory
* PMD: Page Middle Directory
* PTE: Page Table Entry
* offset

48位的地址空间中，4KB每页，KVA和UVA划分方式如下：

* Canonical lower half, 128TB: User VAS, from `0x0` to `0x0000 7fff ffff ffff` 
* Canonical upper half, 128TB: Kernel VAS, from `0xffff 8000 0000 0000` to `0xffff ffff ffff ffff`

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_3.png)

如图7.3所示，大部分的VAS都是空没有用到的。

图7.4展示了常见的`user:kernel` VM Split比例：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_4.png)

### The process VAS

所有存活的进程都有它们自己的user-mode VAS，但是会共享kernel segment，如图7.5所示。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_5.png)

通过proc文件系统，可以方便地查看进程的VAS信息。

通过 `/proc/[pid]/maps` 可以查看进程的内存映射。如图7.6所示。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_6.png)

其中每一行都代表一个用户态VAS的segment/mapping

```bash
address                   perms offset  dev   inode       pathname
555d83b65000-555d83b6d000 r-xp 00000000 08:01 524313      /bin/cat
```
每个字段含义为：

* address：start\_uva - end\_uva
* perms：前三个表示rwx权限，最后一个为s(shared)或p(private, copy on write)。
* offset：对于文件映射，表示相对文件起始位置的offset；对于匿名映射，永远是0。
* dev：格式为`major:minor` ，只对文件映射有效，表示文件所在设备。
* inode：只对文件映射有效。
* pathname：表示被映射的文件的路径。

图7.6最后一行的`vsyscall` 是一种古老的进行系统调用的优化手段，可以不用深究。

除了 `/proc/[pid]/maps` 之外，还有其他前端工具可以帮助查看进程内存映射。如  `/proc/[pid]/smaps` 和 `pmap` ，这两个命令展示了更详细的信息。`smem` 工具可以展示物理内存占用情况。

作者还提供了一个他自己写的可视化工具[procmap](https://github.com/kaiwan/procmap)。图7.8和7.9就是该工具的部分结果截图。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_8.png)

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_9.png)

### Virtual Memory Area

 `/proc/[pid]/maps` 中的每一行都是由内核中的数据结构VMA推断而来。用户空间VAS中的每个segment/mapping都对应一个VMA结构。
 
对于每个进程，内核会通过task structure维护一个VMA“链”(实际上是红黑树)，`current->mm->mmap`。通过VMA里的信息，内核可以执行page faults等内存管理操作。

```c
// include/linux/mm_types.h
struct vm_area_struct {
    /* The first cache line has the info for VMA tree walking. */
    unsigned long vm_start;       /* Our start address within vm_mm. */
    unsigned long vm_end;         /* The first byte after our end address within vm_mm. */
    
    struct vm_area_struct *vm_next, *vm_prev;  /* Linked list of VM areas per task, sorted by address */
    
    struct rb_node vm_rb;
    [...]
    struct mm_struct *vm_mm; /* The address space we belong to */
    pgprot_t vm_page_prot;   /* Access permissions of this VMA */
    unsigned long vm_flags;  /* Flags, see mm.h */
    [...]
    /* Function pointers to deal with this VMA */
    const struct vm_operations_struct *vm_opts;
    /* Information about our backing store */
    unsigned long vm_pgoff; /* Offset (within vm_file) in PAGE_SIZE units */
    struct file *vm_file; /* File we map to (can be NULL) */
    [...]
} __randomize_layout
```

### Kernel Segment

虽然kernel segment的内存布局会根据CPU架构的不同而不同，但还是有些共性的。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_10.png)

依次看一下：
* The user mode VAS
* The lowmem region：这部分会和物理内存做直接的映射（32位系统很可能存在放不下的情况，系统会通过`ZONE_HIGHRAM`做间接映射来解决。64位系统不存在该问题。）。后面会在*Direct-mapped RAM and address translation*那部分详细说明。
* The kernel vmalloc region：内核核心或驱动可以通过`vmalloc()`API族在该位置分配空间。也叫做ioremap空间。
* The kernel modules space：保存Loadable Kernel Modules(LKMs)的文本段和数据段。

作者提供的查看内核内存布局的工具`ch7/show_kernel_seg/kernel_seg.c`，是通过查询描述各个部分位置的宏或者变量来实现的。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_12.png)

通过作者提供的procmap工具也可以查看内核的内存布局：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_15.png)

作者提供的`ch7/show_kernel_seg/kernel_seg.c`也可以一起查看内核VAS和用户VAS：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_16.png)

在内核源码中，针对不同架构，都有文档说明其内存布局。如`Documentation/x86/x86_64/mm.rst`和`Documentation/arm64/memory.rst`等。

### Randomizing the memory layout - [K]ASLR

为了安全，避免攻击者利用“已知的”虚拟地址，用户空间和内核空间都支持ASLR(Address Space Layout Randomization)技术。

用户态的ASLR自linux 2.6.12开始支持。其状态可通过`/proc/sys/kernel/randomize_va_space`查看。可随机化的部分包括共享库的起始地址，基于mmap分配的空间，栈开始地址，堆，vDSO页。

3.14开始，内核空间也支持ASLR，称为KASLR。如`Documentation/x86/x86_64/mm.rst`里的说明：

> Note that if CONFIG_RANDOMIZE_MEMORY is enabled, the direct mapping of all physical memory, vmalloc/ioremap space and virtual memory map are randomized. Their order is preserved but their base will be offset early at boot time.

作者提供了一个脚本 `ch7/ASLR_check.sh` 用来检测用户态和内核态的ASLR设置：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_17.png)

这个脚本也可以设置用户态ASLR状态：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_18.png)

### Physical memory

在启动时，内核会把物理内存按树型层次结构来管理，node -> zone -> page frame。

Nodes are data structures used to denote a physical RAM module on the system motherboard and its associated controller chipset. 即Node是软件层面上的概念。每个node都关联一个physical socket（或者一组处理器核心）。其数据结构为`include/linux/mmzone.h`中的`pg_data_t`。

存在两种架构：
* NUMA (Non-Uniform Memory Access) system：存在多个node。
* UMA (Uniform Memory Access) system：只有一个node。linux会将UMA伪装成NUMA来处理。

下面的示例基于AMD的32核处理器（包含两个physical socket，P#0和P#1），32GB内存（分为4个physical banks）：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_19.png)

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_Hwlo.png)

为了性能，在每个core上跑的线程分配内存时，会优先从其对应的NUMA node中获取。如果其中没有空闲的page frame，则内核会通过特定机制从其他node获取。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_7_20.png)

zone由page frame构成（更准确的说，zone包含的是Page Frame Numbers, PFNs）。通过`/proc/buddyinfo`可以查看node和zone的层级结构。其数据结构为`include/linux/mmzone.h`中的`zone`。

## Kernel Memory Allocation for Module Authors
## The CPU Scheduler

