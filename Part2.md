# Part 2
这一部分记录了OS的整体架构，一些基本概念以及OS的启动(以xv6为例)。
如果是初学者，对这一部分内容第一次看的话没有什么感觉，建议在学完一遍后再回头来翻一翻，很多东西就豁然开朗了。

## 操作系统的内核
我们操作系统的内核主要可以分为几下几类
* Monolithic
* Microkernel
* Exokernel
* Hybrid

Monolithic类的内核是一些经典的系统所采用的设计，包括我们熟悉的MS DOS, Linux, 以及BSD等等。Monolithic内核的主要思想是内核完成大量的工作，整个操作系统都运行在kernel态，所以这种情况下我们对操作系统可以简单理解为就是内核本身。

Microkernel也是现在广泛采用的一种设计，这种内核的特点是只实现最少的功能，比如IPC和scheduling之类的，而我们熟悉的File system, Memory Management System, I/O Driver等等全部在user space实现，早期这种设计会遇到很大的性能瓶颈，但在后来逐步被解决。我们熟悉的Mac OS属于这种设计。
早期的Microkernel设计一般被我们称为L3 kernel,之后又出现了一种L4 Kernel,这类内核的特点是把原本同步的IPC设置成了select(),poll()这样的异步IPC,减少了block的时间，同时实现了一种叫"zero copy"的机制，大大增加了IPC的效率。

Exokernel微内核的核心观点是：只要内核还提供对系统资源的抽象，就不能实现性能的最大优化 -- 内核应该支持一个最小的、高度优化的原语集，而不是提供对系统资源的抽象。从这个观点上来说，IPC也是一个太高级的抽象因而不能达到最高的性能。Exokernel微内核的核心是支持一个高度优化的原语名叫保护控制转移(protected control transfer, PCT)。PCT是一个不带参数的跨地址空间的过程调用，其功能类似于一个硬件中断。在PCT的基础上，可以实现高级的IPC抽象如RPC。在MIPS R3000处理器上，一个基于PCT的RPC实现了仅10µs的开销，而在同一硬件上运行的Mach RPC为95µs。
Exokernel的一个核心思想是保护与管理的分离，具体来说，它会提供给用户管理的权限，不去限制这一部分的功能，转而保护系统上的资源，从而实现保护与管理的分离。
![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/exokernel.JPG)
最后来看一下不同内核的对比
![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/compar.JPG)

## 启动
先大概回忆一下x86下的编程模式
* eip寄存器指向下一条将要执行的语句，call, ret, jmp以及conditional jump这些指令会修改eip的值
* Intel x86下分为几种模式,不同的模式下会有不同的寻址方式
![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/x86mode.JPG)

接下来再介绍一些非常重要的寄存器，之后的lab里基本就在与这些打交道了=^=
### eflags寄存器
eflags寄存器记录了我们的system flags
![](https://www.google.com/url?sa=i&source=images&cd=&ved=2ahUKEwjosLLqj8PiAhWL0FQKHWhUDXcQjRx6BAgBEAU&url=http%3A%2F%2Fwww.c-jump.com%2FCIS77%2FASM%2FInstructions%2FI77_0050_eflags.htm&psig=AOvVaw2zeBviztfPucd2_RnJuIvK&ust=1559301208653741)

eflags里的位往往会记录与前几次计算的结果有关的值，比如是否有进位，是否为正，同时也有一些系统状态位，比如interrupt是否打开，在之后我们处理trapframe的时候eflags是非常重要的一个结构

### CR系寄存器
![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/cr.JPG)
CR系的寄存器在以后的操作系统开发中会时常与我们见面，用的最多的相信就是大家耳熟能详的CR3,记录了PDB的位置；CR2则是记录了上一次page fault具体出错的address,CR0和memory相关的设置相关，CR4用的相对较少，之后你会在内存管理碰到4M大页时与CR4打交道。

剩下的课件里还有GDTR,IDTR,TSS等结构，我觉得放在后面的部分比较好。

