# [CSAPP]第四章 处理器体系结构


一个处理器支持的指令和指令的字节级编码称为它的**指令集体系结构(Instruction-Set Architecture, ISA)**。

本章首先定义一个简单的指令集，作为我们处理器实现的运行示例，我们称之为“Y86-64”指令集。

作为设计处理器的第一步，我们给出一个基于顺序操作、功能正确但是有点不实用的Y86-64处理器。这个处理器每个时钟周期执行一条完整的Y86-64指令。

以这个顺序设计为基础，我们进行一系列的改造，创建一个流水线化的处理器(pipelined-processor)。这个处理器将每条指令的执行分解成五步，每个步骤由一个独立的硬件部分或阶段(stage)来处理。指令步经流水线的各个阶段，且每个时钟周期有一条新指令进入流水线。所以，处理器可以同时执行五条指令的不同阶段。为了使这个处理器保留Y86-64 ISA的顺序行为，就要求处理很多冒险或冲突(hazard)情况。冒险就是一条指令的位置或操作数依赖于其他仍在流水线中的指令。

## 4.1 Y86-64指令集体系结构

定义一个指令集体系结构，包括定义各种状态单元、指令集和它们的编码、一组编程规范和异常事件处理。

### 4.1.1 程序员可见的状态

Y86-64程序中的每条指令都会读取或修改处理器状态的某些部分，这称为**程序员可见(programmer-visible)**状态。包括寄存器、条件码、程序计数器(PC)、内存，还有状态码Stat(表明程序执行的总体状态，它会指示是正常运行，还是出现了某种异常)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_1_1_programmer_visible_state.png)

### 4.1.2 Y86-64指令

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_1_2_instruction_set.png)

### 4.1.3 指令编码

图4-2还给出了指令的字节级编码。每条指令需要1～10个字节不等。每条指令的第一个字节表明指令的类型，这个字节又分成两个部分，每个部分4位：高4位是code部分，低4位是function部分。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_3_function_codes.png)

15个程序寄存器如下：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_4_registers.png)

指令集的一个重要性质就是字节编码必须有唯一的解释。任意一个字节序列要么是一个唯一的指令序列的编码，要么就不是一个合法的字节序列。这个性质保证了处理器可以无二义性地执行目标代码程序。

### 4.1.4 Y86-64异常

对于Y86-64，当遇到异常的时候，我们就简单地让处理器停止执行指令。在更完整的设计中，处理器通常会调用一个**异常处理程序(exception handler)**，这个过程被指定用来处理遇到的某种类型的异常。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_5_status_codes.png)

### 4.1.5 Y86-64程序

```
long sum(long *start, long count)
{
    long sum = 0;
    while (count) {
        sum += *start;
        start++;
        count--;
    }
    return sum;
}
```

图4-6是这段程序的x86-64和Y86-64汇编代码。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_6_assembly_programs.png)

图4-7是Y86-64汇编代码编写的一个完整的程序文件的例子。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_7_sample_program.png)

### 4.1.6 一些Y86-64指令的详情

`pushq`会把栈指针减8，并且将一个寄存器写入内存中。因此，当执行`pushq %rsp`指令时，处理器的行为时不确定的，因为要入栈的寄存器会被同一条指令修改。有两种不同的约定：(1)压入`%rsp`的原始值，(2)压入减去8的`%rsp`的值。x86-64和Y86-64都采用了(1)。

## 4.2 逻辑设计和硬件控制语言HCL

要实现一个数字系统需要三个主要的组成部分：计算对位进行操作的函数的组合逻辑、存储位的存储器单元，以及控制存储器单元更新的时钟信号。

本节简要描述这些不同的组成部分，还将介绍HCL(Hardware Control Language，硬件控制语言)，用这种语言来描述不同处理器设计的逻辑控制。

### 4.2.1 逻辑门

逻辑门是数字电路的基本计算单元。包括与、或、非。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_9_logic_gate.png)

### 4.2.2 组合电路和HCL布尔表达式

将很多的逻辑门组合成一个网，就能构建计算块(computational block)，称为**组合电路(combinational circuits)**。

图4-10为检测位相等的组合电路，HCL函数为`bool eq = (a && b) || (!a && !b)`

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_10_bit_equality.png)

图4-11是多路复用器(multiplexor，通常称为"MUX")，它根据输入控制信号的值，从一组不同的数据信号中选出一个。HCL表达式为`bool out = (s && a) || (!s && b)`

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_11_mux.png)

### 4.2.3 字级的组合电路和HCL整数表达式

通过将逻辑门组合成大的网，可以构造出能计算更加复杂函数的组合电路。通常，我们设计能够对数据字(word)进行操作的电路。

执行字级计算的组合电路根据输入字的各个位，用逻辑门来计算输出字的各个位。

组合逻辑电路可以设计成在字级数据上执行许多不同类型的操作，具体的设计已经超出了我们讨论的范围。**算术/逻辑单元(ALU)**是一种很重要的组合电路。图4-15是它的一个抽象的图示。这个电路有三个输入：标号A和B的两个数据输入，以及一个控制输入。根据控制输入的设置，电路会对数据输入执行不同的算术或逻辑操作。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_15_alu.png)

### 4.2.4 集合关系

略

### 4.2.5 存储器和时钟

> 注意，csapp中文版将register file翻译成了寄存器文件，但更通用的翻译为寄存器堆，因此后文使用了寄存器堆这个翻译。

组合电路从本质上讲，不存储任何信息。相反，它们只是简单地响应输入信号，产生等于输入的某个函数的输出。为了产生**时序电路(sequential circuit)**，也就是有状态并且在这个状态上进行计算的系统，我们必须引入按位存储信息的设备。存储设备都是由同一个时钟控制的。时钟是一个周期性信号，决定什么时候要把新值加载到设备中。考虑两种存储设备：
- **时钟寄存器(clocked registers)**(简称**registers**)：存储单个位或字。时钟信号控制寄存器加载输入值。
- **随机访问存储器(random access memories)**(简称**内存(memories)**)：存储多个字，用地址来选择该读或该写哪个字。ram的例子包括主存和寄存器堆。

在说到硬件和机器级编程时，“寄存器”这个词是由细微差别的。为了区分，我们会分别称之为“硬件寄存器”和“程序寄存器”。

图4-16说明了一个硬件寄存器是如何工作的。当每个时钟到达上升沿时，值才会从寄存器的输入传送到输出。我们的Y86-64处理器会用时钟寄存器保存程序计数器(PC)、条件码(CC)、程序状态(Stat)。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_16_register_operation.png)

下图展示了一个典型的寄存器堆：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_typical_register_file.png)

寄存器堆有两个读端口(A和B)，还有一个写端口(W)，这样一个多端口ram允许同时进行多个读和写操作。图中的寄存器堆，电路可以读取两个程序寄存器的值，同时更新第三个寄存器的状态。

处理器有一个ram用来存放程序数据，如下图：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_data_memory.png)

这个内存有一个地址输入，一个用于写的数据输入，一个用于读的数据输出。read和write是控制信号。error是输出信号。

我们的处理器还包括另外一个只读存储器，用来读指令。在大多数实际系统中，这两个存储器被合并为一个具有双端口的存储器：一个用来读指令，另一个用来读或者写数据。

## 4.3 Y86-64的顺序实现

现在已经有了实现Y86-64处理器所需要的部件。首先，我们描述一个称为SEQ(sequential)的处理器。每个时钟周期上，SEQ执行处理一条完整指令所需的所有步骤，因此时钟周期频率较低，效率低。我们的最终目标是实现一个高效的、流水线化的处理器。

### 4.3.1 将处理组织成阶段

通常，处理一条指令包括很多操作：
- **取指(fetch)**：从内存读取指令字节，地址为PC的值。
- **译码(decode)**：从寄存器堆中读入最多两个操作数。
- **执行(execute)**：ALU要么执行指令指明的操作，计算内存引用的有效地址，要么修改栈指针。
- **访存(memory)**：将数据写入内存，或者从内存读出数据。
- **写回(write back)**：做多可以写两个结果到寄存器堆。
- **更行PC(PC update)**：将PC设置成下一条指令的地址。

下面我们使用图4-17中的代码来描述不同的Y86-64指令的处理。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_17_seq_sample_code.png)

图4-18给出了OPq(整数和逻辑运算)、rrmovq(寄存器-寄存器传送)和irmovq(立即数-寄存器传送)类型的指令的处理。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_18_computations.png)

图4-19给出了内存读写指令rmmovq和mrmovq的处理。注意，内存操作的有效地址是通过计算偏移量与基址寄存器之和得到的。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_19_rmmovq.png)

图4-20给出了pushq和popq的处理。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_20_pushq.png)

图4-21给出了三类控制转移指令的处理：各种跳转、call和ret。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_21_jump.png)

我们创建了一个统一的框架，能够处理所有不同类型的Y86-64指令。虽然指令的行为不大相同，但是我们可以将指令的处理组织成6个阶段。

### 4.3.2 SEQ硬件结构

图4-22给出了一个能执行6个阶段的硬件结构的抽象表示。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_22_abstract_seq.png)

图4-23更详细地给出了实现SEQ所需的硬件。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_23_seq_hardware.png)

### 4.3.3 SEQ的时序

> 原则：从不回读
> 处理器从来不需要为了完成一条指令的执行而去读由该指令更行了的状态。

### 4.3.4 SEQ阶段的实现

本节会设计实现SEQ所需要的控制逻辑块的HCL描述。

略。

## 4.4 流水线的通用原理

流水线化的一个重要特征就是提高了系统的**吞吐量(throughput)**，也就是单位时间内处理的指令数量，不过它也会轻微地增加**延迟(latency)**，也就是处理一条指令所需要的时间。

### 4.4.1 计算流水线

在现代逻辑设计中，电路延迟以**微微秒**或**皮秒(picosecond, "ps")**，也就是$10^{-12}$秒为单位来计算。

我们以每秒千兆条指令(GIPS)，也就是每秒十亿条指令，为单位来描述吞吐量。从头到尾执行一条指令所需要的时间称为**延迟(latency)**。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_32_unpipelined.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_33_pipelined.png)

### 4.4.2 流水线操作的详细说明

流水线阶段之间的指令转移是由时钟信号来控制的。每隔120ps，信号从0上升到1，开始下一组流水线阶段的计算。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_34_pipelined.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_35_one_clock_cycle.png)

### 4.4.3 流水线的局限性

图4-33给出了一个理想的流水线化的系统，通过将计算划分成三个相互独立的阶段，来使得每个阶段需要的时间是原来逻辑的三分之一。不幸的是，会出现其他一下因素，降低流水线的效率。

#### 1. 不一致的划分

如图4-36，仍然将计算划分成三个阶段，但每个阶段的延迟从50ps到150ps不等，通过所有阶段的延迟和仍然为300ps。不过运行时钟的速率是由最慢的阶段的延迟限制的。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_36_nonuniform_stage.png)

通常，处理器中的某些硬件单元，如ALU和内存，是不能被划分成多个延迟较小的单元的，这就使得创建一组平衡的阶段非常困难。

#### 2. 流水线过深，收益反而下降

图4-37说明了流水线技术的另一个局限性。将组合逻辑拆小后，寄存器更新引起的延迟将限制吞吐量的增长。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_37_overhead.png)

为了提高时钟频率，现代处理器采用了很深的流水线(15或更多的阶段)。电路设计者小心地设计流水线寄存器，使其延迟尽可能的小。

### 4.4.4 带反馈的流水线系统

之前的例子，各个阶段的指令之间是完全独立的。但实际上，相邻指令之间很可能是相关的。

当我们将流水线技术引入Y86-64处理器时，必须正确处理反馈的影响，不能改变系统的行为。我们必须以某种方式来处理指令间的数据和控制依赖，以使得到的行为与ISA定义的模型相符。

## 4.5 Y86-64的流水线设计

首先，对顺序的SEQ处理器做一点小的改动，将PC的计算挪到取指阶段。然后，在各个阶段之间加上流水线寄存器。再做一些修改，就能实现我们的目标--一个高效的、流水线化的实现Y86-64 ISA的处理器。

### 4.5.1 SEQ+：重新安排计算阶段

为了更利于流水化，调整SEQ中五个阶段的顺序，使得更新PC阶段再一个时钟周期开始时执行，而不是结束时才执行。我们称这种修改过的设计为“SEQ+”。

图4-40给出了SEQ+硬件的一个详细的说明。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_40_seq+.png)

SEQ到SEQ+中对状态单元的改变是一种很通用的改进的例子，这种改进称为**电路重定时(circuit retiming)**。重定时改变了一个系统的状态表示，但是并不改变它的逻辑行为。通常用它来平衡一个流水线系统中各个阶段之间的延迟。

### 4.5.2 插入流水线寄存器

我们要在SEQ+的各个阶段之间插入流水线寄存器，并对信号进行重新排列，得到PIPE-处理器，这里的“-”代表这个处理器和最终的处理器设计相比，性能要差一点。

PIPE-的抽象结构如图4-41。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_41_pipe-.png)

流水线寄存器：
- F：保存程序计数器的**预测**值
- D：位于取指和译码之间。
- E：位于译码和执行之间。
- M：位于执行和访存之间。保存罪行执行的指令的结果。还保存关于用于处理条件转移的分支条件和分支目标的信息。
- W：位于访存和反馈路径之间，反馈路径将计算出来的值提供给寄存器堆写，而当完成ret指令时，它还要向PC选择逻辑提供返回地址。

图4-42是指令流通过流水线的示例。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_42_pipeline.png)

### 4.5.3 对信号进行重新排列和标号

为了唯一标识信号值，我们采用这样的命名机制：通过在信号名前面加上大写字母的流水线寄存器名字作为前缀，来唯一标识存储在流水线寄存器中的信号；通过在信号名前面加上小写字母的阶段名的第一个字母作为前缀，来唯一标识某个阶段内刚刚计算出来的信号。

> 信号M_stat和m_stat的区别：
> 在命名系统中，大写的前缀"D"、"E"、"M"、"W"指的是流水线寄存器，所以M_stat指的是流水线寄存器M的状态码字段。小写的前缀"f"、"d"、"e"、"m"、"w"指的是流水线阶段，所以m_stat指的是在访存阶段中由控制逻辑产生出的信号状态。

### 4.5.4 预测下一个PC

除了条件转移指令和ret以外，根据取指阶段中计算出的信息，我们能够确定下一条指令的地址。对于call和jump来说，下一条指令的地址是指令中的常数字valC，而对于其他指令来说就valP。因此，通过预测PC的下一个值，在大多数情况下，我们能够达到每个时钟周期发射一条新指令的目的。
 
猜测分支方向并根据猜测开始取指的技术称为**分支预测(branch prediction)**。我们使用简单的策略，即总是预测选择了条件分支，因而预测PC的新值为valC。

ret指令的新PC值几乎无法预测，因为返回地址是位于栈顶的字，其内容可以是任意的。我们只是简单地暂停处理新指令，知道ret指令通过回写阶段。

图4-41中的"Select PC"块类似于图4-40中"PC"块，它从三个值中选择一个作为指令内存的地址：预测的PC，对于到达流水线寄存器M的不选择分支的指令来说是valP的值（存储在寄存器M_valA中），或是当ret指令到达流水线寄存器W（存储在W_valM）时的返回地址的值。

### 4.5.5 流水线冒险

将流水线技术引入一个带反馈的系统，当相邻指令间存在相关时会导致出现问题。这些相关有两种形式：1)**数据相关(data dependencies)**，下一条指令会用到这一条指令计算出的结果；2)**控制相关(control dependencies)**，一条指令要确定下一条指令的位置，如jmp、call、ret。这些相关可能会导致流水线产生计算错误，称为**冒险(hazard)**。冒险也分为两类：**数据冒险(data hazard)**和**控制冒险(control hazard)**。

当一条指令更新后面指令会读到的那些程序状态时，就有可能出现冒险。对于Y86-64来说，程序状态包括程序寄存器、程序计数器、内存、条件码寄存器、状态寄存器。通过分析，我们只需要处理寄存器数据冒险、控制冒险，以及确保能够正确处理异常，就能避免冒险。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_43.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_44.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_45.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_46.png)


#### 1. 用暂停来避免数据冒险

暂停(stalling)是避免冒险的一种常用技术。暂停时，处理器会停止流水线中一条或多条指令，知道冒险条件不再满足。让一条指令停顿在译码阶段，知道产生它的源操作数的指令通过了回写阶段，这样我们的处理器就能避免数据冒险。

虽然这一机制容易实现，但性能并不好，严重降低了系统的吞吐量。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_47.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_48.png)

#### 2. 用转发来避免数据冒险

将结果值直接从一个流水线阶段传到较早阶段的技术称为**数据转发(data forwading)**，有时称为**旁路(bypassing)**。它需要在基本的硬件结构中增加一些额外的数据连接和控制逻辑。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_49.png)

图4-52给出的是PIPE的结构，它是PIPE-的扩展，能通过转发处理数据冒险。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_52_pipe.png)

#### 3. 加载/使用数据冒险

有一类数据冒险不能单纯用转发来解决，因为内存读在流水线中发生的比较晚。图4-53举例说明了**加载/使用冒险(load/use hazard)**，一条指令load，下一条指令use（0x028和0x032）。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_53.png)

如图4-54所示，可以将暂停和转发结合起来，避免加载/使用冒险。这个需要修改控制逻辑，但是可以使用现有的旁路路径。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_54.png)

这种用暂停来处理加载/使用冒险的方法称为**加载互锁(load interlock)**。加载互锁和转发技术结合起来足以处理所有可能类型的数据冒险。因为只有加载互锁会降低流水线的吞吐量，我们几乎可以实现每个时钟周期发射一条新指令的吞吐量目标。

#### 4. 避免控制冒险

当处理器无法根据处于取指阶段的当前指令来确定下一条指令的地址时，就会出现控制冒险。如之前的讨论，控制冒险只会出现在ret指令和跳转指令，可以通过暂停和往流水线中插入气泡的技术来动态调整流水线的流程。

### 4.5.6 异常处理

我们的指令集体系结构包括三种不同的内部产生的异常：1)halt指令；2)由非法指令和功能码组成的指令；3)取指或数据读写试图访问一个非法地址。

我们把导致异常的指令称为**异常指令(exception instruction)**。在一个更完整的设计中，处理器会继续调用**异常处理程序(exception handler)**，这是操作系统的一部分，超出了本书的讲述范围。

### 4.5.7 PIPE各阶段的实现

HCL代码，略。

### 4.5.8 流水线控制逻辑

现在准备创建流水线控制逻辑，完成我们的PIPE设计。这个逻辑必须处理下面4种控制情况，这些情况是其他机制不能处理的：
- 加载/使用冒险：在一条从内存中读出一个值的指令和一条使用该值的指令之间，流水线必须暂停一个周期。
- 处理ret：流水线必须暂停直到ret指令到达写回阶段。
- 预测错误的分支：在分支逻辑发现不应该选择分支之前，分支目标处的几条指令已经进入流水线了。必须取消这些指令，并从跳转指令后面的那条地址开始取指。
- 异常：当一条指令导致异常，我们想要禁止后面的指令更新程序员可见的状态，并且在异常指令到达写回阶段时，停止执行。

我们先浏览每种情况所期望的行为，然后再设计处理这些情况的控制逻辑。后面的部分略。

### 4.5.9 性能分析

我们可以看到，所有需要流水线控制逻辑进行特殊处理的条件，都会导致流水线不能够实现每个时钟周期发射一条新指令的目标。我们可以通过确定往流水线中插入气泡的频率，来衡量这种效率的损失，因为气泡插入会导致未使用的流水线周期。一条ret指令会产生三个气泡，一个加载/使用冒险会产生一个，而一个预测错误的分支会产生两个。我们可以通过计算PIPE指令一条指令所需要的平均时钟周期数的估计值，来量化这些处罚对整体性能的影响，这种衡量方法称为CPI(Cycles Per Instruction，每指令周期数)。这种衡量值是流水线平均吞吐量的倒数，不过时间单位是时钟周期，而不是微微秒。这是一个设计体系结构效率的很有用的衡量标准。

如果忽略异常带来的性能损失，可以使用下面的方法来大致计算CPI：

```
CPI = 1.0 + lp + mp + rp
```

其中，lp(load penalty)，mp(mispredicted branch penalty)，rp(return penalty)分别为那三种情况插入气泡的平均数。

对于CPI的计算，我们使用下面这组频率：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_4_5_9_freq.png)

由此得到的CPI是1.27。

我们的目标是设计一个每个时钟周期发射一条指令的流水线，也就是CPI为1.0。为了进一步降低CPI，应该重点关注预测错误的分支。

### 4.5.10 未完成的工作

1. 多周期指令：如整数乘法和除法，浮点数运算。采用独立于主流水线的特殊硬件功能单元来处理这些复杂操作，能够得到更好的性能。
2. 与存储系统的接口：涉及存储系统层次结构、缺页中断等。

现代处理器支持超标量(superscalar)操作，意味着它们通过并行地取指、译码和执行多条指令，可以实现小于1.0的CPI。最先进的设计使用了一种称为乱序(out-of-order)执行的技术来并行地执行多条指令，执行的顺序也可能完全不同于它们在程序中出现的顺序，但是保留了顺序ISA模型蕴含的整体行为。

## 4.6 小结

略。

