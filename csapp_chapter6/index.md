# [CSAPP]第六章 存储器层次结构


## 6.1 存储技术
### 6.1.1 随机访问存储器
#### 1. 静态RAM
SRAM，有电就保持不变，贵。
#### 2. 动态RAM
DRAM，需要刷新。
#### 3. 传统的DRAM
电路设计者将DRAM组织成二维阵列，而不是线性数组。一个原因是降低芯片上地址引脚的数量。确定是必须分两步发送地址，增加了访问时间。
#### 4. 内存模块
DRAM芯片封装在内存模块(memory module)中，它插到主板的扩展槽上。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_5.png)
#### 5. 增强的DRAM
略。
#### 6. 非易失性存储器
整体上被称为只读存储器(Read-Only Memory, ROM)。
还介绍了闪存(flash memory)和固态硬盘(Solid State Disk, SSD)。
#### 7. 访问主存
数据流通过称为**总线(bus)**的共享电子电路在处理器和DRAM主存之间来来回回。每次CPU和主存之间的数据传送都是通过一系列步骤来完成的，这些步骤称为**总线事务(bus transactoin)**，分为读事务和写事务。
图6-6展示了一个示例计算机系统的配置。系统总线连接CPU和I/O桥接器，内存总线连接I/O桥接器和主存。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_6.png)
图6-7展示了`movq A,%rax`这种读事务的三个步骤。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_7.png)
图6-8展示了`movq %rax,A`这种写事务的三个步骤。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_8.png)

### 6.1.2 磁盘存储
#### 1. 磁盘构造
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_9.png)
#### 2. 磁盘容量
略。
#### 3. 磁盘操作
磁盘以扇区大小的块来读写数据。对扇区的访问时间(access time)有三个主要的部分：寻道时间(seek time)、旋转时间(retational latency)、传送时间(transfer time)。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_10.png)
#### 4. 逻辑磁盘块
现代磁盘封装中有一个小的硬件/固件设备，称为磁盘控制器，维护着逻辑块号和实际(物理)磁盘扇区之间的映射关系。
(盘面，磁道，扇区)这个三元组唯一地标识了对应的物理扇区。
#### 5.连接I/O设备
图6-11展示了一个典型的I/O总线结构，有三种不同类型的设备连接到总线。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_11.png)
#### 6. 访问磁盘
图6-12总结了当CPU从磁盘读数据时发生的步骤。
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_12.png)
CPU使用一种称为内存映射I/O(memory-mapped I/O)的技术来向I/O设备发送命令(图6-12a)。地址空间中又一块地址是为与I/O设备通信保留的，每个这样的地址称为一个I/O端口(I/O port)。

### 6.1.3 固态硬盘

略。

### 6.1.4 存储技术趋势

从上述讨论中可总结出介个重要的思想：
- 不同的存储技术有不同的价格和性能折中。
- 不同存储技术的价格和性能属性以截然不同的速率变化着。
- DRAM和磁盘的性能之后于CPR的性能。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_16.png)

## 6.2 局部性

一个编写良好的计算机程序常常具有良好的局部性(locality)。也就是，它们倾向于引用邻近于其他最近引用过的数据项的数据项，或者最近引用过的数据项本身。这种倾向性，被称为局部性原理(principle of locality)，是一个持久的概念，对硬件和软件系统的设计和性能都有着极大的影响。

## 6.3 存储器层次结构

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_21.png)

存储器层次结构的中心思想是，对于每个k，位于k层的更快更小的存储设备作为位于k+1层的更大更慢的存储设备的缓存。

数据总是以块大小为传送单元(transfer unit)在第k层和第k+1层之间来回复制的。虽然在层次结构中任何一对相邻的层次之间块大小是固定的，但是其他的层次对之间可以有不同的块大小。

缓存不命中的种类：
- 一个空的缓存有时被称为冷缓存(cold cache)，此类不命中称为冷不命中(cold miss)，也称作强制性不命中(compulsory miss)。
- 冲突不命中(conflict miss)：某些限制性的放置策略(placement policy)，会将某些对象映射到同一个缓存块，虽然缓存足够大，但一直不命中。
- 容量不命中(capacity miss)：工作集的大小超过缓存的大小。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_23.png)

## 6.4 高速缓存存储器

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_24.png)

### 6.4.1 通用的高速缓存存储器组织结构

考虑一个计算机系统，每个存储器地址有$m$位，形成$M=2^m$个不同的地址。它的告诉缓存被组织成一个有$S=2^s$个高速缓存组(cache set)的数组。每个高速缓存组包含$E$个高速缓存行(cache line)。每行是由一个$B=2^b$字节的数据块(block)组成的，一个有效位(valid bit)指明这个行是否包含有意义的信息，还有$t=m-(b+s)$个标记位(tag bit)(是当前块的内存地址的位的一个子集)，标记位唯一地标识存储在这个高速缓存行中的块。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_25.png)

一般而言，高速缓存的结构可以用元组$(S, E, B, m)$来描述，其容量$C=S \times E \times B$，标记位和有效位不含括在内。

高速缓存是如何检测它是否包含一个主存地址A处的数据的副本呢？如图6-25b所示，通过类似于哈希的一种映射来做到。

### 6.4.2 直接映射高速缓存

根据每个组的高速缓存行数$E$，高速缓存被分为不同的类。每个组只有一行的高速缓存称为直接映射高速缓存(direct-mapped cache)。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_27.png)

确定一个请求是否命中，然后抽取出被请求的字的过程，分为三步：组选择、行匹配、字抽取。

#### 1. 直接映射高速缓存中的组选择

通过$s$个组索引位

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_28.png)

#### 2. 直接映射高速缓存中的行匹配

通过有效位和标记位

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_29.png)

#### 3. 直接映射高速缓存中的字选择

通过偏移量。

#### 4. 直接映射高速缓存中不命中时的行替换

用从下一层中新取出的行替换当前的行。

#### 5. 运行中的直接映射高速缓存

略。

#### 6. 直接映射高速缓存中的冲突不命中

术语抖动(thrash)描述的是这样一种情况，即高速缓存反复地加载和驱逐相同的高速缓存块的组。

可以通过填充来改变映射，从而修正都懂问题。

> 为什么用中间的位来做索引？
> 如果高位用做索引，那么一些连续的内存块就会映射到相同的高速缓存块。这样会导致使用率低。
> 用中间位作为作引，相邻的块总是映射到不同的高速缓存行。这样使用率高。

### 6.4.3 组相联高速缓存

组相联高速缓存(set associative cache)中每个组都保存多于一个的高速缓存行。一个$1<E<C/B$的高速缓存通常称为$E$路组相联高速缓存。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_32.png)

组选择，通过组索引。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_33.png)

行匹配和字选择。这里一个重要思想就是组中的任何一行都可以包含任何映射到这个组的内存块。所以高速缓存必须搜索组中的每一行，寻找一个有效的行，其标记与地址的标记相匹配。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_34.png)

不命中时的行替换，需要用到LFU、LRU等替换策略。

### 6.4.4 全相联高速缓存

全相联高速缓存(full associative cache)是由一个包含所有高速缓存行的组(即$E=C/B$)组成的。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_35.png)

组选择。因为只有一个组，所以地址中没有组索引位，只有标记位和块偏移。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_36.png)

行匹配和字选择。与组相联一致。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_37.png)

全相联高速缓存只适合做小的高速缓存，例如虚拟内存系统中的翻译备用缓冲器(TLB)，它缓存页表项。

### 6.4.5 有关写的问题

读比较简单，命中就返回，不命中就从下一层找到后存储到本层再返回。

写要复杂一些。

写命中(write hit)，即cache中存在要写的字$w$。在cache更新了它的$w$的副本之后，怎么更新到下一层的副本中呢？
- 直写(write-through)：立即将$w$的高速缓存块写回到紧接着的低一层中。简单，但每次写都会引起总线流量。
- 写回(write-back)：尽可能推迟更新，只有当替换算法要驱逐这个更新过的块时，才把它写到紧接着的低一层中。由于局部性，能显著减少总线流量，但缺点是复杂，要额外维护修改位(dirty bit)。

另一个问题是如何处理写不命中。
- 写分配(write-allocate)：加载相应的低一层中的块到高速缓存中，然后更新这个高速缓存块。
- 非写分配(not-write-allocate)：直接把这个字写到低一层中。

write-through搭配not-write-allocate
write-back搭配write-allocate

心里默认采用write-back搭配write-allocate来写程序就好啦。

### 6.4.6 一个真实的高速缓存层次结构中的解剖

只保存指令的高速缓存称为i-cache。
只保存程序数据的高速缓存称为d-cache。
既保存指令又保存数据的高速缓存称为统一的高速缓存(unified cache)。
现代处理器包括独立的i-cache和d-cache。

图6-38给出了Intel Core i7处理器的高速缓存层次结构。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_38.png)

图6-39总结了Core i7高速缓存的基本特性。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_39.png)

### 6.4.7 高速缓存参数的性能影响

衡量性能的指标：
- 不命中率(miss rate)
- 命中率(hit rate)
- 命中时间(hit time)：从cache传送一个字到CPU所需时间。
- 不命中处罚(miss penalty)：不命中所需要的额外的时间。

影响cache性能的因素包括：
- cache大小：较大的cache可能会增加命中时间。
- 块大小
- 相联度
- 写策略

## 6.5 编写高速缓存友好的代码

1. 让最常见的情况运行地快
2. 尽量减小每个循环内部的缓存不命中率

- 对局部变量的反复引用是好的，因为编译器能够将它们缓存在寄存器文件中(时间局部性)
- 步长为1的引用模式是好的，因为存储器层次结构中所有层次上的缓存都是将数据存储为连续的块(空间局部性)

## 6.6 综合：高速缓存对程序性能的影响

### 6.6.1 存储器山

一个程序从存储系统中读数据的速率称为读吞吐量(read throughput)。

图6-40给出了一个测量某个读序列读吞吐量的函数。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_40.png)

run函数的参数size和stride允许我们控制产生出的读序列的时间和空间局部性程度。
size越小，得到的工作集越小，因此时间局部性越好。
stride越小，得到的空间局部性越好。
反复调整size和stride调用run函数，我们就能得到一个读带宽的时间和空间局部性的二维函数，称为存储器山(memory mountain)。

图6-41展示了Intel Core i7系统的存储山。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_41.png)

我们从这座山中取出一个片段，保持stride不变，我们就能很清楚地看到cache的大小和时间局部性对性能的影响。如图6-42所示。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_42.png)

以相反的方向横切这座山，保持size不变，能看到空间局部性对吞吐量的影响。如图6-43所示。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_6_43.png)

### 6.6.2 重新排列循环以提升空间局部性

略。

### 6.6.3 在程序中利用局部性

我们推荐下列技术：
- 将你的注意力集中在内循环上，大部分计算和内存访问都发生在这里。
- 通过按照数据对象存储在内存中的顺序、以步长为1来读数据，从而使得你的程序中的空间局部性最大。
- 一旦从存储器中读入了一个数据对象，就尽可能多地使用它，从而使得程序中的时间局部性最大。
