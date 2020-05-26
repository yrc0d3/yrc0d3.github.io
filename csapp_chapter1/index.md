# [CSAPP]第一章 计算机系统漫游



示例程序hello.c
```
#include <stdio.h>

int main()
{
    printf("hello, world\n");
    return 0;
}
```

## 1.1 信息就是bits+context

hello.c程序是以字节序列的方式存储在文件中的。

系统中的所有信息--包括磁盘文件、内存中的程序、内存中存放的用户数据以及网络上传送的数据，都是由一串比特表示的。区分不同数据对象的唯一方法是我们读到这些数据对象时的上下文。

## 1.2 程序被其他程序翻译成不同的格式

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_2_compilation_sys.png)

预处理器、编译器、汇编器、链接器一起构成了编译系统(compilation system)：
- 预处理器：根据已字符#开头的命令，修改原始的C程序，结果得到另一个C程序，通常是以.i作为文件扩展名。
- 编译器：将文本文件hello.i翻译成文本文件hello.s，它包含一个汇编文件程序。
- 汇编器：将hello.s翻译成机器语言指令，将这些指令到爆成为一种叫做可重定位目标文件的格式，并将结果保存在目标文件hello.o(二进制文件)中。
- 链接器：printf函数存在与一个名为printf.o的单独的预编译好了的目标文件中，这个文件必须以某种方式合并到hello.o中。链接器就负责这种合并。结果就得到hello这个可执行目标文件文件，可以被加载到内存中，由系统执行。

## 1.3 了解编译系统如何工作是大有益处的

- 优化程序性能
- 理解链接时出现的错误
- 避免安全漏洞


## 1.4 处理器读并解释存储在内存中的指令

### 1.4.1 系统的硬件组成

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_4_hardware.png)

#### 总线

贯穿整个系统的是一组电子管道，称作**总线**，它携带信息字节并负责在各个部件间传递。通常总线被设计成传送定长的字节块，也就是**字(word)**。字中的字节数(即字长)是一个基本的系统参数，现代系统大多为4个字节(32位)或8个字节(64位)。

#### I/O设备

每个I/O设备都通过一个**控制器**或**适配器**与I/O总线相连。控制器与适配器之间的区别主要在于它们的封装方式。控制器是I/O设备本身或者系统的主印制电路板(主板)上的芯片组。适配器则是一块插在主板插槽上的卡。

#### 主存

**主存**是一个临时存储设备，在处理器执行程序时，用来存放程序和程序处理的数据。从物理上来说，主存是由一组**动态随机存取存储器(DRAM)**芯片组成的。

#### 处理器

**中央处理单元(CPU)**，是解释(或执行)存储在主存中指令的引擎。处理器的核心是一个大小为一个字的存储设备(或寄存器)，称为**程序计数器(PC)**。在任何时刻，PC都指向主存中的某条机器语言指令(即含有该条指令的地址)。

处理器一直不断地执行程序计数器指向的指令，再更新程序计数器，使其指向下一条指令。处理器看上去是按照一个非常简单的指令执行模型来操作的，这个模型是由**指令集架构**决定的。

寄存器文件是一个小的存储设备，由一些单个字长的寄存器构成，每个寄存器都有唯一的名字。

ALU计算新的数据和地址值。

我们将处理器的指令集架构和处理器的微体系结构区分开来：
- 指令集架构描述的是每条机器代码指令的效果
- 微体系结构描述的是处理器实际上如何实现的

### 1.4.2 运行hello程序

当我们在键盘上输入字符串`./hello`后， shell程序将字符逐一读入寄存器，再把它放到内存中。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_4_2_input.png)

敲回车后，shell执行一系列指令来加载可执行的hello文件，这些指令将hello目标文件中的代码和数据从磁盘复制到内存汇总。利用直接存储器存取(DMA)技术，数据可以不通过处理器而直接从磁盘到达主存。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_4_2_load.png)

然后处理器执行hello程序中的main程序中的机器语言指令。这些指令将"hello, world\n"字符串中的字节从主存复制到寄存器文件，再从寄存器文件中复制到显示设备，最终显示在屏幕上。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_4_2_output.png)

## 1.5 高速缓存至关重要

处理器和主存之间的速度差距在持续扩大。针对这一现象，系统设计者采用了更小更快的存储设备，称为高速缓存存储器(cache memory)，作为暂时的集结区域，存放处理器机器可能会需要的信息。

L1、L2、L3高速缓存是用一种叫做静态随机访问存储器(SRAM)的硬件技术实现的。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_5_cache.png)

## 1.6 存储设备形成层次结构

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_6_mem_hierarchy.png)

## 1.7 操作系统管理硬件

操作系统有两个基本功能：
1. 防止硬件被失控的应用程序滥用
2. 向应用程序提供简单一致的机制来控制复杂而又通常大不相同的低级硬件设备。

文件是对I/O设备的抽象；虚拟内存是对主存和磁盘I/O设备的抽象表示；进程是对处理器、主存和I/O设备的抽象表示。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_7_system.png)

## 1.8 系统间利用网络通信

略

## 1.9 重要主题

### 1.9.1 Amdahl定律

当我们对系统的某个部分加速时，其对系统整体性能的影响取决于该部分的重要性和加速程度。想要显著加速整个系统，必须提升全系统中相当大的部分的速度。

### 1.9.2 并发和并行

多核处理器和超线程的出现，让线程级并发得到极大提升。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_9_2_multi_core.png)

在较低的抽象层次上，现代处理器可以同时执行多条指令的属性称为指令级并行。早期的处理器心需要多个时钟周期来处理一条指令。现代处理器可以保持每个时钟周期2～4条指令的执行速率，这得益于流水线(pipelining)的使用。

在最低层次上，许多现代处理器拥有特殊的硬件，允许一条指令产生多个可以并行执行的操作，这种方式称为单指令、多数据(single-instruction, multiple-data)，即SIMD并行。

### 1.9.3 计算机系统中抽象的重要性

抽象的使用时计算机科学中最为重要的概念之一。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_1_9_3_abstraction.png)

