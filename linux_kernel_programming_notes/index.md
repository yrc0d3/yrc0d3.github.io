# 《Linux Kernel Programming》读书笔记


## Kernel Workspace Setup
安装虚拟机：[[Apple Silicon M1的Mac电脑安装ubuntu虚拟机 id=7787d9a5-9a6e-4d71-8283-3becf9bd7525]]

安装cross toolchain and QEMU：`sudo apt install qemu-system-arm`

安装cross compiler：`sudo apt install crossbuild-essential-armhf`

安装tldr：man的替代品， 文档更友好

内核官方文档：[https://www.kernel.org/doc/html/latest/](https://www.kernel.org/doc/html/latest/)

静态分析工具：Sparse、Coccinelle、Smatch、Flawfinder、Cppcheck

商业静态分析工具：SonarQube、Coverity Scan、Klocwork

Linux Trace Toolkit next generation（LTTng）：[https://lttng.org/](https://lttng.org/)

Trace Compass GUI：[https://www.eclipse.org/tracecompass/](https://www.eclipse.org/tracecompass/)

procmap：[https://github.com/kaiwan/procmap](https://github.com/kaiwan/procmap) 作者自己的项目，可以查看VAS

Simple Embedded ARM Linux System（seals）：[https://github.com/kaiwan/seals/](https://github.com/kaiwan/seals/) 可以制作arm使用的小型linux系统，也是作者自己的项目。

eBPF：4.0之后的内核可以使用，推荐搭配BCC或bpftrace使用。

Linux Driver Verification（LDV）：[http://linuxtesting.org/ldv/online?action=rules](http://linuxtesting.org/ldv/online?action=rules) 包含很多测试规则

## Building the 5.x Linux Kernel from Source
通过`uname -r`可以查看内核版本号。其命名规则：`major.minor[.patchlevel][-EXTRAVERSION]` ，也常描述为`w.x[.y][-z]` 。

内核开发流程：[https://www.kernel.org/doc/html/latest/process/2.Process.html#how-the-development-process-works](https://www.kernel.org/doc/html/latest/process/2.Process.html#how-the-development-process-works)

整体流程：

1.  获取源码。两种方式：
   1. 去[https://www.kernel.org/](https://www.kernel.org/)下载源码。研究源码的话推荐使用这种方式，下载特定版本的代码即可。
   2. 使用git clone。想要贡献源码的话可以使用这种方式。
2. 解压缩（git clone方式可跳过）。设置环境变量`export LLKD_KSRC=${path_to_kernel_src}`方便后续操作。
3. 配置内核编译选项
   1. 每个可配置项`FOO`都用一个`CONFIG_FOO` 宏表示。其可选值为y(=yes，直接编译进内核镜像)、m(=module，编译成一个单独的内核模块对象文件)、n(=no，不编译)。
   2. Kconfig文件中保存了各种`CONFIG_FOO`的定义。
   3. 真正的内核配置文件是内核源码根目录下的`.config`文件，一般都是通过Makefile来生成的。
   4. 作者列举了3种配置方式
      1. 使用典型的嵌入式linux系统内核配置，比如树莓派使用的配置是`${LLKD_KSRC}/arch/arm/configs/bcm2835_defconfig`
      2. 使用编译机所使用的发行版的内核配置，直接复制即可。`cp /boot/${uname -r} ${LLKD_KSRC}/.config`
      3. 通过localmodconfig来配置。将localmodconfig作为make的目标，就可以生成仅以正在使用的内核模块为对象的`.config`文件。这种方法只会启用编译机上已加载的模块，因此能显著减少编译时间。
   5. `make menuconfig`有UI界面，最好用它来修改配置（即`.config`文件），而不要直接修改`.config`文件。因为很多配置项会有依赖关系，手动修改容易出错。
   6. 还讲了如何在UI界面中增加配置项，即编辑Kconfig文件。要使其生效还需要修改Makefile。
4. 编译内核镜像和模块
   1. 在`${LLKD_KSRC}`目录下通过`make`命令来编译。默认执行的是`make all`命令，它会编译出来3个重要产出：
      1. `vmlinux`以及`System.map`是没有压缩的内核镜像，用于debug。
      2. modules是一堆`.ko` 文件，即各种内核模块，散落在各个文件夹中。
      3. bzImage是压缩后的可启动内核镜像，位于`arch/<arch>/boot/`目录下。bootloader会把它载入内存，解压后启动。该文件是跟cpu架构相关的，bzImage是x86下的产物，在arm平台(如m1 mac)上的产物则为Image.gz。
   2. 使用`time make -j4`命令可以记录整体时间，以及通过多线程来加速。
5. 安装内核模块。
   1. 在`${LLKD_KSRC}`目录下通过`sudo make modules_install`命令来安装，默认安装到`/lib/modules/<kernel-ver>`下。
   2. 通过环境变量`INSTALL_MOD_PATH`可以改变安装路径。
6. 安装GRUB bootloader，生成initramfs(init ram filesystem)镜像。
   1. 通过执行`sudo make install`命令。
   2. initramfs在2.4及之前的版本中叫做initrd。
   3. 介绍了linux boot过程，可以参考后面的图片。
7. 定制GRUB bootloader
   1. 完成上面的步骤后，重启后默认会使用刚刚编译的内核。想要在GRUB菜单中选择内核的话，就需要定制了。
   2. 通过编辑`/etc/default/grub`来定制。

linux启动过程的好图：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/linux_boot_stages.png)

## Writing Your First Kernel Module
### 内核构成
系统调用system call：是特殊的API，是用户态进程访问内核的唯一合法途径。

内核主要组件：

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_4_2.png)

LKM (Loadable Kernel Module) framework：

* 内核模块是一段可以动态加载进内核的代码。

* 在Linux内核源码树以外来开发并编译一个模块，称为树外开发。

* 之所以提供模块机制，是因为Linux内核本身是一个单内核，单内核由于所有内容都集成在一起， 效率很高，但可扩展性和可维护性较差，模块机制可弥补这一缺陷。

* 2.6之后的内核模块都以`.ko`(kernel object)结尾。

下图显示的是内核模块的编译，以及插入到内核空间的过程。

![image](https://raw.githubusercontent.com/yrc0d3/imagehosting/master/img/lkp_4_3.png)

### 内核模块示例代码
```cpp
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>

MODULE_AUTHOR("<insert your name here>");
MODULE_DESCRIPTION("LKP book:ch4/helloworld_lkm: hello, world, our first LKM");
MODULE_LICENSE("Dual MIT/GPL");
MODULE_VERSION("0.1");

static int __init helloworld_lkm_init(void)
{
        printk(KERN_INFO "Hello, world\n");
        return 0;               /* success */
}

static void __exit helloworld_lkm_exit(void)
{
        printk(KERN_INFO "Goodbye, world\n");
}

module_init(helloworld_lkm_init);
module_exit(helloworld_lkm_exit);
```
构成：

* kernel headers
   * `/lib/modules/$(uname -r)/build`是链接，指向系统中内核header的位置。
   * 对于`#include <linux/init.h>` ，编译器会去`/lib/modules/$(uname -r)/build/include/`搜索。
* module macros：MODULE\_FOO()格式的那些。
* entry and exit points：`module_[init|exit]()`
* return values
   * `static int __init <modulename>_init(void)`
      * 遵循the 0/-E return conversion
      * 成功返回0
      * 失败返回负整数，glibc会将其乘以-1后设置为全局的`errno`。
      * init函数返回值只能是int，不方便。想要返回其他类型怎么办？可以通过ERR\_PTR、PTR\_ERR、IS\_ERR方法来hack。
   * `static void __exit <modulename>_exit(void)`
* `__init`和`__exit` 这两个宏是用来标记代码的，方便编译器处理。

### 常见操作
用户空间中glibc的`printf`，或者c++的`cout`，在内核中都是无法使用的。内核空间中只能使用`printk`这个API。

`printk`会输出到kernel log buffer中，通过`dmesg`这个shell命令来查看内核日志。

以下这些命令都是kmod命令的软链接：

* insmod: insert module
* rmmod: remove module
* lsmod: list modules
* modinfo: module info
* modprobe
* depmod

作者提供了一个叫做lkm的脚本，其整体流程clean/build/load/lsmod/dmesg。

### kernel logging和printk
printk默认输出到内存中的kernel log buffer(volatile)中，也可以额外输出到kernel log file(non-volatile)、console。

内存中的kernel log buffer是ring buffer，默认大小为$2^{18}=256$KB。

为解决内存日志的易失性问题，各种linux发行版也会写入文件日志。Red Hat系的写入`/var/log/messages`，Debian系的写入`/var/log/syslog`。

新版本的linux中，会通过`systemd-journal`这个后台进程来写处理日志。`journalctl` 是用户接口。日志格式为`[timestamp] [hostname] [source]: [... log message ...]` 。

printk支持8种日志级别：

```cpp
// /lib/modules/$(uname -r)/build/include/linux/kern_levels.h
#define KERN_EMERG      KERN_SOH "0"    /* system is unusable */
#define KERN_ALERT      KERN_SOH "1"    /* action must be taken immediately */
#define KERN_CRIT       KERN_SOH "2"    /* critical conditions */
#define KERN_ERR        KERN_SOH "3"    /* error conditions */
#define KERN_WARNING    KERN_SOH "4"    /* warning conditions */
#define KERN_NOTICE     KERN_SOH "5"    /* normal but significant condition */
#define KERN_INFO       KERN_SOH "6"    /* informational */
#define KERN_DEBUG      KERN_SOH "7"    /* debug-level messages */
```
如果使用printk时没有指定log level，那么默认使用KERN\_WARNING。

`pr_<foo>`宏更方便使用。比如

```cpp
// /lib/modules/$(uname -r)/build/include/linux/printk.h
#define pr_info(fmt, ...) \
        printk(KERN_INFO pr_fmt(fmt), ##__VA_ARGS__)
```
默认情况下，只有KERN\_WARNING的更高级别才会打印到console。

```bash
$ cat /proc/sys/kernel/printk
4    4    1    7
```
这四个数字的含义为：

1. 当前console的日志级别，数字小于该值的日志才会输出到console。
2. 没有设置log level时的默认级别
3. The minimum allowed log level
4. The boot-time default log level

当需要限制printk打印频率时，可以用`printk_ratelimited`这个宏（实际上是调用了`lib/ratelimit.c:__ratelimit()`），它默认允许在5s内最多打印10条相同的消息出来。对应的，也有`pr_<foo>_ratelimited`这种宏，如`pr_info_ratelimited`。

通过设置`pr_fmt`宏，可定制通用的日志前缀。

### kernel module的Makefile
```makefile
# ch4/printk_loglvl/Makefile
PWD          := $(shell pwd)
obj-m        += printk_loglvl.o

# Enable the pr_debug() and pr_devel() as well by removing the comment from
# one of the lines below
#EXTRA_CFLAGS += -DDEBUG
#CFLAGS_printk_loglvl.o := -DDEBUG

all:
        make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
install:
        make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules_install
clean:
        make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) clean
```
Kbuild系统使用`obj-y`和`obj-m`两个变量来表示不同模块的编译配置。

`obj-y` 表示把`*.o`编译进内核。

`obj-m` 表示把`*.o`作为模块进行编译，不会编译进内核，而是生成独立的`*.ko`文件。

### A "Better" Makefile template
* usual targets: build, install, clean
* adhering to kernel coding style (indent+checkpatch)
* several static analysis targets (via sparse, gcc, flawfinder, cppcheck)
* two 'dummy' dynamic analysis targets (KASAN, LOCKDEP)
* a packaging (.tar.xz) target
* a help target.

### Cross-compliing a kernel module
没有交叉编译的需求，先跳过。

### Licensing kernel module
内核源码本身是基于GNU GPL v2 (GPL-2.0)的。如果你想向内核贡献源码，那么你的代码也得遵循GPL-2.0。

可通过`MODULE_LICENSE()`宏来指定协议。

### Emulating "library-like" features for kernel modules

使内核模块模拟library特性

> do not reinvent the wheel, software reuse, modularity

可内核里是没有library的，但有两种方式可以让内核模块，模拟库的效果：

1. 在内核模块显式引入其他模块的源码文件
2. module stacking

#### Performing library emulation via multiple source files

除了在c语言源码文件中通过include引入其他c文件之外，还需要修改Makefile：

```make
obj-m               := <label>.o
<label>-objs := proj1.o proj2.o proj3.o
```

可以看到，在`obj-m`指令之后，紧接着`<label>-objs`指令。内核编译系统会将每个被引入的c文件单独编译成一个`.o`文件，然后把它们链接在一起，从而形成最终的二进制内核模块目标文件`<label>.ko`。

第8章中有个一个真实的示例：

```make
# ch8/lowlevel_mem/Makefile
obj-m                 += lowlevel_mem_lkm.o
lowlevel_mem_lkm-objs := lowlevel_mem.o ../../klib_llkd.o
```

#### module stacking

在了解module stacking之前，需要先了解下内核模块中方法和变量的作用域。

自内核2.6版本开始，所有的内核模块的变量(static和global)、方法的作用域默认只在本模块内，外部不可见。

LKM框架提供了`EXPORT_SYMBOL()`和`EXPORT_SYMBOL_GPL()`这两个宏来改变作用域，使得全部模块都可见。

想要使用`EXPORT_SYMBOL()`，需要去掉`static`修饰符(虽然允许同时使用`static`和`EXPORT_SYMBOL`，但最好不要这么用)。

```c
int my_glob = 5;
EXPORT_SYMBOL(my_glob);

long my_foo(int key)
{ [...]
}
EXPORT_SYMBOL(my_foo);
```

当一个内核模块的变量、数据结构、函数使用`EXPORT_SYMBOL()`之后，其他模块就可以通过`extern`引用它们了。（Makefile中的`obj-m`指令也需要添加被引用的模块？？？暂不确定）

```c
extern int my_glob;
extern long my_foo(int);
```

通过`lsmod`命令可以查看一个内核模块被其他模块引用的情况，第三列表示数量，第四列是其used by的具体模块名称。

```bash
$ lsmod | grep vbox
vboxnetadp       28672  0
vboxnetflt       28672  1
vboxdrv         479232  3 vboxnetadp,vboxnetflt
```

内核模块之间的这种依赖关系，就像是stack一样，因此有了module stacking这个概念。

### module parameters

模块参数是在模块插入时（`insmod`）通过`name=value`的格式传入的，例如`sudo insmod modparams1.ko mp_debug_level=2`。

module没有`main()`方法，也就没有`(argc, argv)`参数列表，那么这种传参是如何实现的？其实是链接器做的：在内核模块中将变量声明为`static`，然后调用宏`module_param()`即可。

```c
// ch5/modparams/modparams1/modparams1.c
static int mp_debug_level;
module_param(mp_debug_level, int, 0660);
MODULE_PARM_DESC(mp_debug_level,
"Debug level [0-2]; 0 => no debug messages, 2 => high verbosity");
```

`module_param()`的三个参数：
1. 变量名称，需要`stacic`修饰
2. 数据类型
3. 对应sysfs的文件权限

`MODULE_PARM_DESC`用于添加描述信息，通过`modinfo`命令可查看。

在模块被插入之后，可以在`/sys/module/<module-name>/parameters/*`看到所有的模块参数。`module_param()`的第三个参数就对应这里的特定参数的文件权限。修改文件中的值，也就修改了模块参数。

模块参数支持的数据类型在`include/linux/moduleparam.h`中`module_param`的注释中可以查看。

所有的内核模块参数默认都是可选的。如何做到必填呢？两种方式：一是在`module_init`时检查参数如果是默认值就报错，二是使用`module_param_cb`这个宏。

### auto-loading modules on system boot

`make modules_install`之后，将`<module-name>`写入文件`/etc/modules-load.d/<module-name>.conf`中（或者添加到文件`/etc/modules-load.d/modules.conf`中）。重启系统生效。

如果自动装载的模块需要输入参数怎么办？编辑`/etc/modprobe.d/*.conf`文件，按照格式`options <module-name> <parameter-name>=<value>`进行传参。
