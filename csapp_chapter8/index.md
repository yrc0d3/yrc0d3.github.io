# [CSAPP]第八章 异常控制流



从处理器加电开始，直到你断电为止，程序计数器假设一个值的序列

$$ a_0,a_1,...a_{n-1} $$

其中，每个$a_k$是某个相应的指令$I_k$的地址。每次从$a_k$到$a_{k+1}$的过渡称为**控制转移(control transfer)**。这样的控制转移序列叫做处理器的**控制流(flow of control, or control flow)**。

 

最简单的一种控制流式一个“平滑的”序列，其中每个$I_k$和$I_{k+1}$在内存中都是相邻的。这种平滑流的突变（也就是$I_k$和$I_{k+1}$不相邻）通常是由诸如跳转、调用和返回这样一些熟悉的程序指令造成的。

 

但是系统也必须能够对系统状态的变化做出反应，这些系统状态不是被内部程序变量捕获的，而且也不一定要和程序的执行相关。比如，硬件定时器产生的信号、网络事件等。



现代系统通过使控制流发生突变来对这些情况做出反应。一般而言，我们把这些突变称为异常控制流（Exceptional Control Flow，ECF）。异常控制流发生在计算机系统的各个层次，如硬件层、操作系统层、应用层。



## 8.1 异常



异常时异常控制流的一种形式，它一部分由硬件实现，一部分由操作系统实现。



**异常(exception)**就是控制流中的突变，用来响应处理器状态中的某些变化。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_1.png)



状态变化称为**事件(event)**。在任何情况下，当处理器检测到由事件发生时，它就会通过一张叫做**异常表(exception table)**的跳转表，进行一个间接过程调用(异常)，到一个专门设计用来处理这类事件的操作系统子程序(异常处理程序，exception handler)。当异常处理程序完成后，根据引起异常的事件的类型，会发生以下3种情况中的一种：

1. 处理程序将控制返回给当前指令$I_curr$，即当事件发生时正在执行的指令。

2. 处理程序将控制返回给$I_next$，如果没有发生异常将会执行的下一条指令。

3. 处理程序终止被中断的程序。



### 8.1.1 异常处理



系统中可能的每种类型的异常都分配了一个唯一的非负整数的**异常号(exception number)**。其中一些是由处理器的设计者分配(被零除、缺页、内存访问违例灯)，其他由操作系统内核的设计者分配(系统调用、来自外部的I/O信号等)。



在系统启动时，操作系统分配和初始化一张称为**异常表**的跳转表，表目k包含异常k的处理程序的地址。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_2.png)



### 8.1.2 异常的类别



异常可分为四类：中断(interrupt)、陷阱(trap)、故障(fault)、终止(abort)。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_4.png)



#### 1. 中断



中断是异步发生的，是来自处理器外部的I/O设备的信号的结果。硬件中断的异常处理程序常常称为**中断处理程序(interrupt handler)**。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_5.png)



#### 2. 陷阱和系统调用



陷阱是**有意的**异常，是执行一条指令的结果。陷阱最重要的用途是在用户程序和内核之间提供了一个像过程调用一样的接口，叫做**系统调用**。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_6.png)



#### 3. 故障



故障由错误情况引起，它可能能够被故障处理程序修正。当故障发生时，处理器将控制转移给故障处理程序。如果处理程序能够修正这个错误情况，它就将控制返回引起故障的指令，从而重新执行它。否则，处理程序返回到内核中的abort例程，abort例程会终止引起故障的应用程序。



一个经典的故障示例是缺页异常。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_7.png)



#### 4. 终止



终止是不可恢复的致命错误造成的结果，通常是一些硬件错误。终止处理程序直接将控制返回给一个abort例程，该例程会终止这个应用程序。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_8.png)



### 8.1.2 Linux/x86-64系统中的异常



x86-64定义了256种不同的异常类型。0～31的号码对应的是由Intel架构师定义的异常，因此对任何x86-64系统都是一样的。32～255的号码对应的是操作系统定义的中断和陷阱。



#### 1. Linux/x86-64故障和终止



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_9.png)





#### 2. Linux/x86-64系统调用



每个系统调用都有一个唯一的整数号，对应于一个到内核中跳转表的偏移量（和异常表不一样）。



在x86-64系统上，系统调用是通过一条称为syscall的陷阱指令来提供的。所有到Linux系统调用的参数都是通过通用寄存器而不是栈传递的。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_10.png)



## 8.2 进程



这里我们将关注进程提供给应用程序的关键抽象：

- 一个独立的逻辑控制流，它提供一个假象，好像我们的程序独占地使用处理器。

- 一个私有的地址空间，它提供一个假象，好像我们的程序独占地使用内存系统



### 8.2.1 逻辑控制流



每个程序执行时PC(程序计数器)值的序列叫做**逻辑控制流**，或者简称**逻辑流**。



### 8.2.2 并发流



一个逻辑流的执行在时间上与另一个重叠，称为**并发流(concurrent flow)**，这两个逻辑流被称为**并发地运行**。



多个流并发地执行的一般现象被称为**并发(concurrency)**。一个进程和其他进程轮流运行的概念称为**多任务(multitasking)**。一个进程执行它的控制流的一部分的每一时间段叫做**时间片(time slice)**。因此，多任务也叫做**时间分片(time slicing)**。



并行流是并发流的真子集。如果两个流并发地运行在不同的处理器核或者计算机上，那么我们称它们为**并行流(parallel flow)**，它们**并行地运行(running in parallel)**，且**并行地执行(paralles execution)**。



### 8.2.3 私有地址空间



每个进程都有自己的**私有地址空间**。



尽管和每个私有地址空间相关联的内存的内容一般是不同的，但是每个这样的空间都有相同的通用结构。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_13.png)



### 8.2.4 用户模式和内核模式



为了使操作系统内核提供一个无懈可击的进程抽象，处理器必须提供一种机制，限制一个应用可以执行的指令以及它可以访问的地址空间范围。



处理器通常是用某个控制寄存器的一个**模式位(mode bit)**来提供这种功能，改寄存器描述了进程当前享有的特权。当设置了模式位时，进程就运行在**内核模式(kernel mode)**中(又是也叫超级用户模式supervisor mode)。没有设置模式位时，进程就运行在**用户模式(user mod)**中。



Linux提供了一种聪明的机制，叫做`/proc`文件系统，它允许用户模式进程访问内核数据结构的内容。`/proc`文件系统将许多内核数据结构的内容输出为一个用户程序可以读的文本文件的层次结构。2.6版本的Linux内核引入了`/sys`文件系统，它输出关于系统总线和设备的额外的低层信息。



### 8.2.5 上下文切换



操作系统内核使用一种称为**上下文切换(context switch)**的较高层形式的异常控制流来实现多任务。



内核为每一个进程维持一个**上下文(context)**。上下文就是内核重新启动一个被抢占的进程所需的状态。它由一些对象的值组成，包括通用目的寄存器、浮点寄存器、程序计数器、用户栈、状态寄存器、内核栈，和各种内核数据结构，比如描述地址空间的页表(page table)、包含当前进程信息的进程表(process table)、包含进程已打开文件信息的文件表(file table)。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_14.png)



## 8.3 系统调用错误处理



当unix系统级函数遇到错误时，它们通常会返回-1，并设置全局整数变量errno来表示什么出错了。strerror函数返回一个文本串，描述了和某个errno值相关联的错误。



一般的错误检查：

```

if ((pid = fork()) < 0) {

    fprintf(stderr, "fork error: %s\n", strerror(errno));

    exit(0);

}

```



使用error-handling wrappers之后的Fork(首字母大写)代码：

```

void unix_error(char *msg)

{

    fprintf(stderr, "%s: %s\n", msg, strerror(errno));

    exit(0);

}



pid_t Fork(void)

{

    pid_t pid;

    

    if ((pid = fork()) < 0)

        unix_error("Fork error");

    return pid;

}

```



本书后面都使用了error-handling wrappers。



## 8.4 进程控制



### 8.4.1 获取进程ID



```

pid_t getpid(void);  // caller's PID

pid_t getppid(void); // parent PID

```



### 8.4.2 创建和终止进程



从程序员的角度，可以认为进程总是处于下面三种状态中：

- Running：正在被执行或者等待调度。

- Stopped：被挂起(suspended)，不会被调度。

- Terminated：3种原因，1)收到信号且其默认行为是终止，2)从主程序返回，3)调用exit函数。



父进程通过调用fork函数来创建一个新的运行的子进程。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_15.png)



通过画进程图可以帮助理解fork函数。进程图是刻画程序语句的偏序的一种简单的前趋图。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_16.png)



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_17.png)



### 8.4.3 回收(Reaping)子进程



当一个进程由于某种原因终止后，会保持终止状态，知道被它的父进程回收(reaped)。



一个终止了但还未被回收的进程称为僵死进程(zombie)。



如果一个父进程终止了，内核会安排init进程成为它的孤儿进程的养父。



一个进程可以通过调用waitpid函数来等待它的子进程终止或者停止。



waitpid的内容略。



### 8.4.4 让进程休眠



sleep函数将一个进程挂起一段指定的时间。



pause函数让函数休眠，直到该进程收到一个信号。



```

unsigned int sleep(unsigned int secs);

int pause(void);

```



### 8.4.5 加载并运行程序



execve函数在当前进程的上下文中加载并运行一个新的程序。



> 程序和进程

> A **process** is a specific instance of a program in execution; a **program** always runs in the context of some process.

> The **fork** function runs the same program in a new child process that is a duplicate of the parent. 

> The **execve** function loads and runs a new program in the context of the current process. While it overwrites the address space of the current process, **it does not create a new process**. The new program still has the same PID, and it inherits all of the file descriptors that were open at the time of the call to the execve function.



### 8.4.6 利用fork和execve运行程序



略。



## 8.5 信号



一个**信号**就是一条小消息，它通知进程系统中发生了一个某种类型的事件。



每种信号类型都对应于某种系统事件。地层的硬件异常是由内核异常处理程序处理的，正常情况下，对用户进程而言是不可见的。信号提供了一种机制，通知用户进程发生了这些异常。比如，如果一个进程试图除以0，那么内核就发送给它一个SIGFPE信号。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_26.png)



### 8.5.1 信号术语



传送一个信号到目的进程包含两个步骤：发送信号，接收信号。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_27.png)



一个发出儿没有被接收的信号叫做**待处理信号(pending signal)**。在任何时刻，一种类型至多只会有一个待处理信号，再来就会被丢弃。一个进程可以有选择性地阻塞接收某种信号。但一种信号被阻塞时，它仍可以被发送，但是产生的待处理信号不会被接收，直到进程取消对这种信号的阻塞。



一个待处理信号最多只能被接收一次。内核为每个进程在pending位向量中维护着待处理信号的集合，而在blocked位向量中维护着被阻塞的信号集合。



### 8.5.2 发送信号



Unix系统提供了大量向进程发送信号的机制，所有这些机制都是基于**进程组(process group)**这个概念的。



#### 1. 进程组



每个进程都只属于一个进程组。进程组通过一个正整数**进程组ID**来标识。



默认地，一个子进程和它的父进程同属于一个进程组。`getpgrp`函数返回当前进程的进程组ID。`setpgid`函数可以改变进程自己或其他进程的进程组。



#### 2. 用/bin/kill程序发送信号



/bin/kill程序可以向进程发送任意信号。



#### 3. 从键盘发送信号



CTRL+C会导致内核发送一个SIGINT信号到前台进程组的每个进程。



#### 4. 用kill函数发送信号



`int kill(pid_t pid, int sig)`



#### 5. 用alarm函数发送信号



`unsigned int alarm(unsigned int secs)`安排内核在secs秒后发送一个SIGALRM信号给调用进程。



### 8.5.3 接收信号



当内核把进程p从内核模式切换到用户模式时，它会检查进程p的未被阻塞的待处理信号的集合（pending&~blocked）。如果这个集合为空，那么内核将控制传递到p的逻辑控制流中的下一条指令。然而，如果集合非空，那么内核选择集合中的某个信号k（通常是最小的k），并且强制p接收信号k。收到这个信号会触发进程采取某种行为。一旦进程完成这个行为，那么控制就传递回p的逻辑控制流中的下一条指令。



每个信号类型都有一个预定义的默认行为，见图8-26。进程可以通过`sighandler_t signal(int signum, sighandler_t handler)`函数修改和信号相关联的默认行为。SIGSTOP和SIGKILL的默认行为无法修改。



信号处理程序可以被其他信号处理程序中断，如图8-31所示。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_31.png)



### 8.5.4 阻塞和解除阻塞信号



Linux提供阻塞信号的隐式和显式的机制：

- 隐式阻塞机制：内核默认阻塞任何当前处理程序正在处理信号类型的待处理的信号。

- 显式阻塞机制：应用程序可以使用sigprocmask函数和它的辅助函数，明确地阻塞和解除选定的信号。



### 8.5.5 编写信号处理程序



信号处理程序有几个属性使得它们很难推理分析：

1. 处理程序和主程序并发运行，共享同样的全局变量，因此可能与主程序和其他处理程序互相干扰。

2. 如何以及何时接收信号的规则常常有违人的直觉。

3. 不同的系统有不同的信号处理语义。



#### 1. 安全地信号处理



这里给出一些保守的编写处理程序的原则，使得这些处理程序能安全地并发运行：

- 处理程序要尽可能简单。

- 在处理程序中只调用异步信号安全的函数（要么是可重入的，要么它不能被信号处理程序中断）。

- 保存和恢复errno。

- 阻塞所有的信号，保护对共享全局数据结构的访问。

- 用volatile声明全局变量。

- 用sig_atomic_t声明标志。



#### 2. 正确的信号处理



关键思想是如果存在一个未处理的信号就表明**至少**有一个信号达到了。



不可以用信号来对其他进程中发生的事件计数。



#### 3. 可移植的信号处理



Posix标准定义了sigaction函数，它允许用户在设置信号处理时，明确指定它们想要的信号处理语义。一个更简单的方式是，定义一个包装函数，成为Signal，它调用sigaction。



![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_8_38.png)



### 8.5.6 同步流以避免讨厌的并发错误



基本的问题是以某种方式**同步**并发流，从而得到最大的可行的交错的集合，每个可行的交错都能得到正确的结果。



### 8.5.7 显式地等待信号



介绍了sigsuspend函数。



## 8.6 非本地跳转



C语言提供了一种用户级异常控制流形式，称为**非本地跳转(nonlocal jump)**，它将控制直接从一个函数转移到另一个当前正在执行的函数，而不需要经过正常的调用-返回序列。非本地跳转是通过setjmp和longjmp函数来提供的。



```

#include <setjmp.h>

int setjmp(jmp_buf env);

int sigsetjmp(sigjmp_buf env, int savesigs);



void longjmp(jmp_buf env, int retval);

void siglongjmp(sigjmp_buf env, int retval);

```



非本地跳转的一个重要应用就是允许从一个深层嵌套的函数调用中立即返回，通常是由检测到某个错误情况引起的。另一个重要应用是使一个信号处理程序分支到一个特殊的代码位置，而不是返回到被信号到达中断了的指令的位置。



PS：具体代码示例较复杂，还是看书吧。



> C++和Java提供的异常机制是较高层次的，是C语言的setjmp和longjmp函数的更加结构化的版本。你可以把try语句中的catch子句看做类似于setjmp函数。相似的，throw语句就类似于longjmp函数。



## 8.7 操作进程的工具



STRACE：打印一个正在运行的程序和它的子进程调用的每个系统调用的轨迹。

PS：列出当前系统中的进程（包括僵死进程）。

TOP：打印出关于当前进程资源使用的信息。

PMAP：显式进程的内存映射。

/proc：一个虚拟文件系统，以ASCII文本格式输出大量内核数据结构的内容，用户可以读取这些内容。



## 8.8 小结





