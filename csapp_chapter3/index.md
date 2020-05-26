# [CSAPP]第三章 程序的机器级表示


## 3.1 历史观点

略

## 3.2 程序编码

假设一个C程序，有两个文件p1.c和p2.c，可以使用命令`gcc -Og -o p p1.c p2.c`来编译。编译选项-Og告诉编译器使用会产生符合原始C代码整体结构的机器代码的优化等级。使用较高级别优化产生的代码会严重变形，以至于产生的机器代码和初始源代码之间的关系非常难以理解。因此我们使用-Og优化作为学习工具。

### 3.2.1 机器级代码

对于机器级编程来说，有两种重要的抽象：
- 指令集体系结构/指令集架构(Instruction Set Architecture, ISA)，定义了机器级程序的格式和行为，定义了处理器状态、指令的格式，以及每条指令对状态的影响。
- 机器级程序使用的内存地址是虚拟地址。

一条机器指令只执行一个非常基本的操作。例如，将存放在寄存器中的两个数字相加，在存储器和寄存器之间传送数据，或是条件分支转移到新的指令地址。编译器必须产生这些指令的序列，从而实现程序结构。

### 3.2.2 代码示例

假设我们写了mstore.c：
```
long mult2(long, long);

void multstore(long x, long y, long *dest) {
    long t = mult2(x, y);
    *dest = t;
}
```

运行命令`gcc -Og -S mstore.c`产生的汇编文件代码包含以下几行：
```
multstore:
  pushq   %rbx
  movq    %rdx, %rbx
  call    mult2
  movq    %rax, (%rbx)
  popq    %rbx
  ret
```
上面代码中每个缩进去的行都对应与一条机器指令。

使用`gcc -Og -c mstore.c`会编译并汇编该代码，产生目标文件mstore.o，它是二进制格式的。1368字节的mstore.o中有一段14字节的序列，它的十六进制表示为：
```
53 48 89 d3 e8 00 00 00 00 48 89 03 5b c3
```
这即是上面的汇编指令对应的目标代码。我们可以从中得到一个重要信息，即机器执行的程序只是一个字节序列，它是对一系列指令的编码。

要查看机器代码文件的内容，有一类称为反汇编器(disassambler)的程序非常有用。Linux中可以用`objdump -d mstore.o`得到如下结果：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/caspp_3_2_2_objdump.png)

### 3.2.3 关于格式的注解

假设我们使用`gcc -Og -S mstore.c`生成文件mstore.s，内容如下：
```
        .file   "010-mstore.c"
        .text
        .globl  multstore
        .type   multstore, @function
multstore:
        pushq   %rbx
        movq    %rdx, %rbx
        call    mult2
        movq    %rax, (%rbx)
        popq    %rbx
        ret
        .size   multstore, .-multstore
        .ident  "GCC: (Ubuntu 4.8.1-2ubuntu1~12.04) 4.8.1"
        .section        .note.GNU-stack,"",@progbits
```
所有以'.'开头的行都是指导汇编器和链接器工作的伪指令，通常可忽略。
为了更清楚地说明汇编代码，我们使用下面这种格式：
![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_2_3_comment.png)

## 3.3 数据格式

由于是从16位体系结构扩展成32位的，Intel用术语“字(word)”表示16位数据类型。32位数称为“双字(double words)”，64位数称为“四字(quad words)”。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_3_word.png)

如图所示，大多是GCC生成的汇编代码指令都有一个字符的后缀，表明操作数的大小。后缀'l'用来表示双字，因为32位数倍看成是“长字(long words)”。虽然'l'也用来表示8字节双精度浮点数，但不会产生歧义，因为浮点数使用的是一组完全不同的指令和寄存器。

## 3.4 访问信息

一个x86-64的CPU包含一组16个存储64位值的**通用目的寄存器**，用来存储整数数据和指针。它们的名字都以%r开头，后面的名字则是跟历史相关了。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_4_int_registers.png)

名称解释：
- ax: accumulator register 累加器；也用来存放函数返回值
- bx: base register 基地址寄存器，没啥特殊用途
- cx: counter register 计数寄存器for loop
- dx: data register 数据寄存器
- si: source index 源索引寄存器
- di: destination index 目标索引寄存器
- bp: base pointer 基址指针；指向栈底
- sp: stack pointer 栈指针；指向栈顶

名称解释的参考资料：https://www.swansontec.com/sregisters.html

如图，指令可以对这16个寄存器的地位字节中存放的不同大小的数据进行操作。对于生成小于8个字节结果的指令，寄存器剩下的字节会怎样，有两条规则：
- 生成1字节和2字节数字的指令会保持剩下的字节不变
- 生成4字节数字的指令会把高位4字节置为0

### 3.4.1 操作数指示符

大多数指令有一个或多个**操作数(operand)**，指示出执行一个操作中要使用的源数据值，以及存放结果的目的位置。x86-64支持的操作数格式如下：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_4_operand.png)

源数据值可以以常数形式给出，或是从寄存器或内存中读出。
结果可以存放在寄存器或内存中。
因此，操作数可被分为三种类型：
1. 立即数(immediate)：用来表示常数值。书写方式为`$`后面跟一个用标准C表示法表示的整数，如`$101`、`$0x1F`
2. 寄存器(register)：某个寄存器的内容。我们用符号$r_a$表示任意寄存器$a$，用引用$R[r_a]$表示它的值。这是将寄存器集合看成一个数组$R$，用寄存器标识符作为索引。
3. 内存引用：它会根据计算出来的地址访问某个内存位置。用符号$M_b[Addr]$表示对存储在内存中从地址$Addr$开始的$b$个字节的引用。为了简便，通常隐去$b$。

如上图，有多种不同的**寻址模式**，允许不同形式的内存引用。最常用的是图底部的$Imm(r_b, r_i, s)$：一个立即数偏移$Imm$，一个基址寄存器$r_b$，一个变址寄存器$r_i$和一个比例因子$s$(只能为1/2/4/8)。基址和变址寄存器都必须是64位寄存器。当引用数组和struct元素时，复杂的寻址模式是很有用的。

### 3.4.2 数据传送指令

最频繁使用的指令是将数据从一个位置复制到另一个位置的指令。

最简单形式的数据传送指令为MOV类，由四条指令组成：movb、movw、movl、movq，区别在于操作的数据大小不同。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/caspp_3_4_2_mov.png)

源操作数指定的值是一个立即数，存储在寄存器或者内存中。目的操作数指定一个位置，要么是一个寄存器，要么是一个内存地址。x86-64加了一条限制，传送指令的两个操作数不能都指向内存位置。内存->内存可以通过两条指令完成，即内存->寄存器，寄存器->内存。

下图给出了五种可能的组合。记住，第一个参数是源操作数，第二个参数是目的操作数。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_4_2_move_example.png)

MOVZ和MOVS是两类将较小的源值复制到较大的目的时使用的数据移动指令，都是把数据从源复制到寄存器。MOVZ类中的指令把目的中剩余的字节填充为0，MOVS类中的指令通过符号扩展来填充，把源操作数的最高位进行复制。每条指令名字的最后两个字符都是大小指示符，分别指示源和目的的大小。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_4_2_movz.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_4_2_movs.png)

### 3.4.3 数据传送示例

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_4_3_example.png)

从中可以看出，C语言中的“指针”其实就是地址。间接引用指针就是将该指针放在一个寄存器中，然后在内存引用中使用这个寄存器。

### 3.4.4 压入和弹出栈数据

在x86-64中，程序栈存放在内存中某个区域。栈向下增长，栈顶元素的地址是所有栈中元素地址中最低的。图中的栈是倒着画的，栈顶在图的底部。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_4_4_stack_op.png)

将一个四字值压入栈中，首先要将栈指针减8，然后将值写到新的栈顶地址。因此，指令`pushq %rbp`等价于下面两条指令：
```
subq $8,%rsp       // Decrement stack pointer
movq %rbp,(%rsp)   // Store %rbp on stack
```

区别在于机器代码中pushq指令编码为1个字节，而上面两条指令一共需要8个字节。

弹出一个四字的操作包括从栈顶位置读出数据，然后将栈指针加8。因此，指令`popq %rax`等价于下面两条指令：
```
movq (%rsp),%rax   // Read %rax from stack
addq $8,%rsp       // Increment stack pointer
```

`%rsp`指向的地址总是栈顶。

## 3.5 算数和逻辑操作

这些操作被分成了四组：加载有效地址、一元操作、二元操作、位移。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_5_operations.png)

### 3.5.1 加载有效地址

加载有效地址(load effective address)指令`leaq`实际上是`movq`指令的变形，它将有效地址写入到目的操作数，可以为后面的内存引用产生指针。另外，它还可以简洁地描述普通的算数操作，例如假设寄存器`%rdx`的值为`x`，那么指令`leaq 7(%rdx,%rdx,4),%rax`将设置寄存器`%rax`的值为`5x+7`。

### 3.5.2 一元和二元操作

一元操作类似于C语言中的`++`和`--`操作符。

二元操作类似于C语言中的赋值运算符，如`x-=y`。

### 3.5.3 移位操作

先给出移位量，再给出要移位的数。

移位量是一个立即数，或者放在单字节寄存器`%cl`中。

左移指令有两个名字：SAL和SHL。二者效果一样，都是将右边填充0。

右移指令，SAR执行算术移位(填充符号位)，SHR执行逻辑移位(填充0)。

移位操作的目的操作数可以是一个寄存器或事一个内存地址。

### 3.5.4 讨论

上面的大多数指令，既可以用于无符号运算，又可以用于补码运算。只有右移操作要求区分有符号和无符号数。这个特性是使得补码运算成为实现有符号整数运算的一种比较好的方法的原因之一。

### 3.5.5 特殊的算术操作

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_5_5_128op.png)

两个64位有/无符号整数相乘的结果需要128位来表示。x86-64指令集对128位数的操作提供有限的支持。Intel把16字节的数称为八字(oct word)。

了解一下即可。

## 3.6 控制

机器代码提供两种基本的低级机制来实现有条件的行为：测试数据值，然后根据测试结果来改变控制流或者数据流。

### 3.6.1 条件码

除了整数寄存器，CPU还维护着一组单个位的条件码(conditon code)寄存器，它们描述了最近的算术或逻辑操作的属性。可以检测这些寄存器来执行条件分支指令。最常用的条件码有：
- CF: 进位标志。最近的操作使最高位产生了进位。可用来检查无符号操作的溢出。
- ZF: 零标志。最近的操作得出的结果为0。
- SF: 符号标志。最近的操作得到的结果为负数。
- OF: 溢出标志。最近的操作导致一个补码溢出--正溢出或负溢出。

图3-10中，除了`leaq`外，其他指令都会设置条件码。

`CMP`和`TEST`指令只设置条件码而不改变任何其他寄存器。

`CMP`根据两个操作数之差来设置条件码。它除了只设置条件码而不更新目的寄存器之外，与`SUB`指令的行为是一样的。

`TEST`的行为与`AND`一样。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_1_CMP_TEST.png)

### 3.6.2 访问条件码

条件码通常不会直接读取，常用的使用方式有三种：
1. 可以根据条件码的某种组合，将一个字节设置为0或者1
2. 可以条件跳转到程序的某个其他的部分
3. 可以有条件地传送数据

我们将这一整类指令称为`SET`指令。它们之间的区别就在于它们考虑的条件码的组合是什么，这些指令名字的不同后缀指明了它们所考虑的条件码的组合。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_2_set.png)

一条`SET`指令的目的操作数是低位单字节寄存器元素之一，或是一个字节的内存位置，指令会将这个字节设置成0或者1。为了得到一个32位或64位的结果，必须对高位清零。

一个计算C语言表达式`a < b`的典型指令序列如下：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_2_set_cmp.png)

注：这一小节没看太明白，但感觉内容也不是很重要

### 3.6.3 跳转指令

正常执行的情况下，指令会按照它们出现的顺序一条一条地执行。跳转(jump)指令会导致执行切换到程序中一个全新的位置。在汇编代码中，这些跳转的目的地通常用一个标号(label)指明。

在产生目标代码文件时，汇编器会确定所有带标号指令的地址，并将跳转目标(目的指令的地址)编码为跳转指令的一部分。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_3_jump.png)

`jmp`指令是无条件跳转。它可以是直接跳转，即跳转目标是作为指令的一部分编码的；也可以是间接跳转，即跳转目标是从寄存器或内存地址读出的。

表中所示的其他跳转指令都是有条件的。这些指令的名字和跳转条件与`SET`指令的名字和设置条件是相匹配的。条件跳转只能是直接跳转。

### 3.6.4 跳转指令的编码

在汇编代码中，跳转目标用符号标号书写。汇编器，以及后来的链接器，会产生跳转目标的适当编码。跳转指令有几种不同的编码，但是最常用的都是PC相对的(PC-relative)。也就是，它们会将目标指令的地址与紧跟在跳转指令后面那条指令的地址之间的差作为编码。这些地址偏移量可以编码为1、2或4个字节。第二种编码方式是给出“绝对”地址，用4个字节直接指定目标，汇编器和链接器会选择适当的跳转目的编码。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_4_jmp_labelled.png)

右边反汇编器产生的注释中，第2行中跳转指令的跳转目标指明为0x8，第5行中跳转指令的跳转目标指明为0x5。再观察指令的字节编码，第一个跳转指令的目标编码为0x03(红框标注的)，把它加上0x5(即下一条指令的地址)就得到跳转目标地址0x8，也就是第4行指令的地址。

类似的，第二个跳转指令的目标为0xf8(红框标注的，十进制-8)，再加上0xd(即下一条指令的地址，十进制13)，我们得到0x5，即第3行指令的地址。

这些例子说明，当执行PC相对寻址时，程序计数器的值时跳转指令后面的那条指令的地址，而不是跳转指令本身的地址。

下面是链接后的程序反汇编版本：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_4_disassembled.png)

这些指令被重定位到不同的地址，但是第2行和第5行中跳转目标的编码并没有变。通过使用与PC相对的跳转目标编码，指令编码很简洁(只需要2字节)，而且目标代码可以不做改变就移到内存中不同的位置。

### 3.6.5 用条件控制来实现条件分支

条件控制：conditional control

将条件表达式和语句从C语言翻译成机器代码，最常用的方式是结合有条件和无条件跳转。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/caspp_3_6_5_conditonal_statements.png)

C语言中的`if-else`语句的通用形式模板如下：
```
if (test-expr)
    then-statement
else
    else-statement
```

对于这种通用形式，汇编实现通常会使用下面这种形式（这里用C语法描述）：
```
    t = test-expr;
    if (!t)
        goto false;
    then-statement
    goto done;
false:
    else-statement;
done:
```

也就是，汇编为`then-statement`和`else-statement`产生各自的代码块。它会插入条件和无条件分支，以保证能执行正确的代码块。

### 3.6.6 用条件传送来实现条件分支

条件传送：conditional move

这种方法会计算一个条件操作的两种结果（即计算`then-statement`和`else-statement`），然后再根据条件是否满足，从中选取一个。只有在一些受限制的情况下，这种策略才可行。一旦可行，就可以用一条简单的conditional move指令来实现它，这个指令更符合现代处理器的性能特性。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_5_conditional_assignment.png)

为什么基于conditional move的代码会比基于conditional control的代码性能要好？因为处理器使用pipelining来获得高性能，这要求能够事先确定要执行的指令序列，这样才能保证流水线中充满了待执行的指令。当机器遇到条件跳转（分支）时，只有当分支条件求值完成之后，才能决定分支往哪边走。如果处理器预测错了，那么处理器将丢掉该跳转指令后所有指令已经做了的工作，然后再从正确的位置重新填充流水线，这样性能就下降了。

下图列举了一些条件conditional control指令。它们都有两个操作数，结果取决于条件码的值。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_6_conditional_move.png)

考虑下面的条件表达式和赋值的通用形式：
```
v = test-expr ? then-expr : else-expr;
```

用conditional control transfer的标准方法来编译这个表达式，会得到如下形式：
```
    if (!test-expr)
        goto false;
    v = then-expr;
    goto done;
false:
    v = else-expr;
done:
```

基于conditional control的代码，会对`then-expr`和`else-expr`都求值，最终值的选择基于对`test-expr`的求值。可以用下面的抽象代码描述：
```
v = then-expr
ve = else-expr
t = test-expr
if (!t) v = ve
```

这个序列中的最后一条语句是用conditional move实现的--只有当测试条件`t`满足时，`vt`的值才会被复制到`v`中。

不是所有的条件表达式都可以用conditional move来编译。因为要对`then-expr`和`else-expr`都求值，所以如果其中任意一个表达式可能产生错误条件或者副作用，就会导致非法的行为。3.6.5中的例子就是这种情况。

使用conditional move也不总是会提高代码的效率。比如`then-expr`或`else-expr`的求值计算量很大，那么当对应的条件不满足时，就白计算了。

总的来说，条件数据传送提供了一种用条件控制转移来实现条件操作的替代策略。它们只能用于非常受限制的情况，但这些情况还是相当常见的，并且与现代处理器的运行方式更契合。

### 3.6.7 循环

C语言提供了多种循环结构，即`do-while`、`while`和`for`。

#### do-while

通用形式：
```
do
    body-statement
    while (test-expr);
```

这种通用形式可以被翻译成：
```
loop:
    body-statement
    t = test-expr;
    if (t)
        goto loop;
```

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_7_do_while.png)

#### while

通用形式：
```
while (test-expr)
    body-statement
```

第一种翻译方法，我们称之为跳转到中间(jump into middle)。它执行一个无条件跳转跳到循环结尾处的测试，以此来执行初始的测试。

```
    goto test;
loop:
    body-statement
test:
    t = test-expr
    if (t)
        goto loop;
```

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/caspp_3_6_7_while_jm.png)

第一种翻译方法，我们称之为guarded-do。首先用条件分支，如果初始条件不成立就跳过循环，把代码变换为`do-while`循环。

```
t = test-expr
if (!t)
    goto done;
do
    body-statement
    while (test-expr);
done:
```

还可以把它翻译成goto代码：

```
t = test-expr
if (!t)
    goto done;
loop:
    body-statement
    t = test-expr
    if (t)
        goto loop;
done:
```

利用这种实现策略，编译器常常可以优化初始的测试，例如认为测试条件总是满足。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/caspp_3_6_7_while_gd.png)

#### for

通用形式：
```
for (init-expr; test-expr; update-expr)
    body-statement
```

大部分情况下，都可以等价为使用while循环：
```
init-expr
while (test-expr) {
    body-statement
    update-expr
}
```

GCC为`for`循环产生的代码是`while`循环的两种翻译之一，这取决于优化的等级。

### 3.6.8 switch语句

switch语句可以根据一个整数索引值进行多重分支(multiway branching)。在处理具有多种可能结果的测试时，这种语句特别有用。它们不仅提高了C代码的可读性，而且通过使用**跳转表(jump table)**这种数据结构使得实现更加高效。跳转表是一个数组，表项`i`是一个代码段的地址，这个代码段实现当开关索引值等于`i`时程序应该采取的动作。当开关情况数量较多，并且值的范围跨度较小时，GCC会使用跳转表。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_8_switch_trans.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_6_8_switch_assembly.png)

## 3.7 过程

过程是软件中一种很重要的抽象。它提供了一种封装代码的方式，用一组指定的参数和一个可选的返回值实现了某种功能。然后可以在程序中不同的地方调用这个函数。

假设过程P调用过程Q，Q执行后返回到P。这些动作包括以下一个或多个机制：
- **传递控制**：在进入过程Q的时候，程序计数器必须被设置为Q的代码的起始位置，然后在返回时，要把程序计数器设置为P中调用Q后面那条指令的地址。
- **传递数据**：P必须能够向Q提供一个或多个参数，Q必须能够向P返回一个值。
- **分配和释放内存**：在开始时，Q可能需要为局部变量分配空间，而在返回前，又必须释放这些存储空间。

### 3.7.1 运行时栈

程序可以用栈来管理它的过程所需要的存储空间，栈和程序寄存器存放着传递控制和数据、分配内存所需要的信息。

当x86-64过程需要的存储空间超出寄存器能够存放的大小时，就会在栈上分配空间。这个部分称为过程的**栈帧(stack frame)**

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_7_1_stack_frame.png)

如图所示，我们把Q返回时的返回地址当做P的栈帧的一部分，因为它存放的是与P相关的状态。

大多数过程的栈帧都是定长的，在过程的开始就分配好了。变长的栈帧在3.10.5节中讨论。通过寄存器，过程P可以传递最多6个整数值，但是如果Q需要更多的参数，P可以在调用Q之前在自己的栈帧里存储好这些参数。

实际上，许多函数甚至根本不需要栈帧。当所有的局部变量都可以保存在寄存器中，而且该函数不会调用任何其他函数（有时称之为叶子过程，此时把过程调用看做树结构）时，就可以这样处理。

### 3.7.2 转移控制

将控制从函数P转移到函数Q，只需要简单地把程序计数器(PC)设置为Q的代码的起始位置。不过，当稍后从Q返回的时候，处理器必须记录好它需要继续执行的P的代码位置。在x86-64机器中，这个信息是用指令`call Q`调用过程Q来记录的。该指令会把地址A压入P的栈中，并将PC设置为Q的起始位置。压入的地址A被称为返回地址(return address)，是紧跟在call指令后面的那条指令的地址。对应的指令ret会从栈中弹出地址A，并把PC设置为A。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/caspp_3_7_2_calls_returns.png)

可以看到，这种把返回地址压入栈的简单机制能够让函数在稍后返回到过程中正确的点。C语言标准的调用/返回机制刚好与栈提供的后进先出的内存管理方法吻合。

### 3.7.3 数据传送

在x86-64中，大部分过程的间的数据传送是通过寄存器实现的，可以通过寄存器最多传递6个整形(例如整数和指针)参数。寄存器的使用是有特殊顺序的，寄存器使用的名字取决于要传递的数据类型的大小。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_7_3_reg_args.png)

如果一个函数有大于6个整形参数，超出6个的部分就要通过栈来传递。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_7_3_multi_args.png)

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_7_3_multi_args_frame.png)

### 3.7.4 栈上的局部存储

以下情况，局部数据必须放在内存中：
- 寄存器不足够存放所有的本地数据
- 对一个局部变量使用地址运算符`&`，因此必须能够为它产生一个地址
- 某些局部变量是array或struct，所以必须能够通过array or structure references被访问到

一般来说，过程通过减小栈指针在栈上分配空间。分配的结果作为栈帧的一部分，标号为local variables，如上面的图3-25所示。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_7_4_arg_alloc.png)

### 3.7.5 寄存器中的局部存储空间

寄存器组是唯一被所有过程共享的资源。虽然在给定时刻只有一个过程是活动的，我们仍然必须确保当一个过程调用另一个过程时，被调用者不会覆盖调用者稍后会使用的寄存器组。因此，x86-64采用了一组统一的寄存器使用惯例，所有的过程都必须遵守。

根据惯例，寄存器%rbx、%rbp和%r12~%r15被划分为callee-saved registers。当过程P调用过程Q时，Q必须保存这些寄存器的值，保证它们的值在Q返回到P时与Q被调用时是一样的。过程Q如果要使用这些寄存器，那么必须把原始值压入栈中，返回前再从栈中弹出旧值。这些旧值会在栈帧中创建标号为"saved registers"的一部分，如图3-25所示。

其他所有的寄存器（除了栈指针%rsp）都被分类为caller-saved registers。过程P在某个此类寄存器中有局部数据，然后调用过程Q。因为Q可以随意修改这个寄存器，所以在调用之前首先保存好这个数据是P的责任。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_7_5_callee_saved_reg.png)

### 3.7.6 递归过程

递归调用一个函数本身与调用其他函数是一样的。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_7_6_recursive.png)

## 3.8 数组分配和访问

### 3.8.1 基本原则

对于数据类型`T`和整形常数`N`，声明如下：$T A[N];$

起始位置表示为$x_A$。这个声明有两个效果：首先，它在内存中分配一个$L * N$字节的连续区域，这里$L$是数据类型$T$的大小(单位为字节)。其次，它引入了标识符$A$，可以用$A$来作为指向数组开头的指针，这个指针的值就是$x_A$

### 3.8.2 指针运算

C语言允许对指针进行运算，而计算出来的值会根据该指针引用的数据类型的大小进行伸缩。

单操作数操作符'&'和'*'可以产生指针和间接引用指针。

### 3.8.3 嵌套的数组

要访问多维数组的元素，编译器会以数组起始为基地址，(可能需要经过伸缩的)偏移量为索引，产生计算期望的元素的偏移量，然后使用某种MOV命令。

### 3.8.4 定长数组

C语言编译器能够优化定长多维数组上的操作代码。

### 3.8.5 变长数组

ISO C99引入了变长数组，允许数组的维度是表达式，在数组被分配的时候才计算出来。

## 3.9 异质的数据结构

struct和union

### 3.9.1 结构

struct的所有组成部分都存放在内存中一段连续的区域内，而指向struct的指针就是struct第一个字节的地址。

### 3.9.2 联合

union提供了一种方式，能够规避C语言的类型系统，允许以多种类型来引用一个对象。

### 3.9.3 数据对齐

许多计算机系统对基本数据类型的合法地址做出了一些限制，要求某种类型对象的地址必须是某个值(通常是2、4或8)的倍数。这种**对齐限制**简化了形成处理器和内存系统之间接口的硬件设计。

## 3.10 在机器级程序中将控制与数据结合起来

### 3.10.1 理解指针

我们重点介绍一些指针和它们映射到机器代码的关键原则：
- **每个指针都对应一个类型**
- **每个指针都有一个值**
- **指针用'&'运算符创建**
- **'*'操作符用于间接引用指针**
- **数组与指针紧密联系**
- **将指针从一种类型强制转换成另一种类型，只改变它的类型，而不改变它的值**
- **指针也可以指向函数**

### 3.10.2 应用：使用GDB调试器

略

### 3.10.3 内存越界引用和缓冲区溢出

我们已经看到，C对于数组引用不进行任何边界检查，而且局部变量和状态信息都存放在栈中。这两种情况结合到一起就能导致严重的程序错误，对越界的数组元素的写操作会破快存储在栈中的状态信息。当程序使用这个被破坏的状态，试图重新加载寄存器或执行ret指令时，就会出现很严重的错误。

一种特别常见的状态破坏称为**缓冲区溢出(buffer overflow)**。通常，在栈中分配某个字符数组来保存一个字符串，但是字符串的长度超出了为数组分配的空间。

缓冲区溢出的一个更加致命的使用就是让程序执行它本来不愿意执行的函数。这是一种最常见的通过计算机网络攻击系统安全的方法。通常，输入给程序一个字符串，这个字符串包含一些可执行代码的字节编码，称为**攻击代码(exploit code)**，另外，还有一些字节会用一个指向攻击代码的指针覆盖返回地址。那么，执行ret指令的效果就是跳转到攻击代码。

在一种攻击形式中，攻击代码会使用系统调用启动一个shell程序，给攻击者提供一组操作系统函数。在另一种攻击形式中，攻击代码会执行一些未授权的任务，修复对栈的破坏，然后第二次执行ret指令，(表面上)正常返回到调用者。蠕虫(worm)病毒就使用者这种攻击方式。

### 3.10.4 对抗缓冲区溢出攻击

#### 1. 栈随机化

为了在系统中插入攻击代码，攻击者既要插入代码，也要插入指向这段攻击代码的指针，这个指针也是攻击字符串中的一部分。产生这个指针需要知道这个字符串放置的栈位置。老系统中的程序的栈地址非常容易预测，栈的位置相当固定。

**栈随机化(Stack Randomization)**的思想使得栈的位置在程序每次运行时都有变化。

在Linux中，栈随机化已经变成了标准行为。它是更大的一类技术中的一种，这类技术称为**地址空间布局随机化(Address-Space Layout Randomization)**，或者简称ASLR。采用ASLR，每次运行程序的不同部分，包括程序代码、库代码、栈、全局变量和堆数据，都会被加载到内存的不同区域。

#### 2. 栈破坏检测

最近的GCC版本在产生的代码中加入了一种**栈保护者(stack protector)**机制，来检测缓冲区越界。其思想是在栈帧中任何局部缓冲区与栈状态之间存储一个特殊的**金丝雀(canary)**值，它也称为**哨兵值(guard value)**，是在程序每次运行时随机产生的。在回复寄存器状态和从函数返回之前，程序检查这个金丝雀值是否被该函数的某个操作改变了。如果是的，那么程序异常中止。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_10_4_stack_protector.png)

栈保护很好地防止了缓冲区溢出攻击破坏存储在程序栈上的状态。它只会带来很小的性能损失。

#### 3. 限制可执行代码区域

最后一招是消除攻击者向系统中插入可执行代码的能力。一种方法是限制哪些内存区域能够存放可执行代码。

### 3.10.5 支持变长栈帧

有些函数，需要的局部存储是变长的，比如调用alloca时。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_10_5_code.png)

为了管理变长栈帧，x86-64代码使用寄存器%rbp作为**帧指针(frame pointer，又称为base pointer)**。当使用%rbp时，栈帧的组织结构如下图。可以看到，代码必须把%rbp之前的值保存到栈中，因为它是一个callee-saved register。然后在整个函数的执行过程中，都是用%rbp指向那个时刻栈的位置，然后用固定长度的局部变量(比如i)相对于%rbp的偏移量来引用它们。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_10_5_vframe.png)

在函数的结尾，`leave`指令帧指针恢复到它之前的值(第20行)。这条指令不需要参数，等价于执行下面两条指令：
```
 movq %rbp, %rsp # set stack pointer to beginning of frame
 popq %rbp       # restore saved %rbp and set stack ptr to end of the caller's frame
```

## 3.11 浮点代码

简要回顾历史会对理解x86-64浮点体系结构有所帮助。1997年出现了Pentium/MMX，Intel和AMD都引入了持续数代的用于支持图形和图像处理的媒体(media)指令。这些指令本意是允许多个操作以并行模式执行，称为**single instruction, multiple data**，或**SIMD(读作sim-dee)**。在这种模式中，可以对多个不同的数据并行执行同一个操作。
名字的演化过程：MMX -> SSE(Streaming SIMD Extention) -> AVX(Advanced Vector Extention)
我们的讲述基于AVX2。

AVX浮点体系结构允许数据存储在16个YMM寄存器中，它们的名字为%ymm0~%ymm15。每个YMM寄存器都是256位。当对标量数据操作时，这些寄存器只保存浮点数，而且只使用低32位(float)或64位(double)。汇编代码用寄存器的SSE XMM寄存器名字%xmm0~%xmm15来引用它们，每个XMM寄存器都是对应的YMM寄存器的低128位。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_11_media_reg.png)

### 3.11.1 浮点传送和转换操作

下图给出了一组在内存和XMM寄存器之间，以及从一个XMM寄存器到另一个XMM寄存器，不做任何转换地传送浮点数的指令。引用内存的指令是标量(scalar)指令，意味着它们只对单个而不是一组封装好的数据值进行操作。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_11_1_fp_move_instructions.png)

其他指令，包括浮点数和整数类型之间的转化，不同浮点格式之间的转换，这里就不详细说了。

### 3.11.2 过程中的浮点代码

在x86-64中，XMM寄存器用来向函数传递浮点参数，以及从函数返回浮点值。

### 3.11.3 浮点运算操作

每个指令有一个或两个源操作数，一个目的操作数：加、减、乘、除、最大、最小、平方根。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_11_3_fp_arithmetic_op.png)

### 3.11.4 定义和使用浮点常数

和整数运算操作不同，AVX浮点操作不能以立即数作为操作数。编译器必须为所有的浮点常量值分配和初始化存储空间，然后代码再把这些值从内存读取寄存器来使用。

### 3.11.5 在浮点代码中使用位级操作

位级异或(EXCLUSIVE-OR)，位级与(AND)：

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_11_5_bitwiseop.png)

### 3.11.6 浮点比较操作

它们类似于CMP指令，比较操作数S1和S2，并且设置条件码来指示它们的相对值。

![](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/csapp_3_11_6_cmp.png)

浮点比较指令会设置三个条件码：零标志位ZF、进位标志位CF、奇偶标志位PF(操作数中出现NaN时使用)。

### 3.11.7 对浮点代码的观察结论

用AVX2为浮点数上的操作产生的机器代码风格类似于为整数上的操作产生的代码风格。它们都使用一组寄存器来保存和操作数据值，也都使用这些寄存器来传递函数参数。

AVX2还有能力在封装好的数据上执行并行操作，使计算执行地更快。
