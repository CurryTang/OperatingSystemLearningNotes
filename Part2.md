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

### XV6具体的boot分析

首先是两种内存寻址的模型，一种是Intel的8086实模式，一种是Intel的8086保护模式。
在刚开机的时候，我们尚处于实模式下，这种模式的特点是我们直接使用Physical Address进行寻址操作，同时这时候只有20位的内存空间(1MB),使用段式寻址。

注意这20位是怎么来的，其实这个时候我们的段寄存器CS DS SS ES IP什么的都是16位的，那我们的物理地址其实是PA = 16 * segment + offset得到的，也就是高16位的段以及低位的段偏移量。

然后我们看一看现在的address space(主要是低1MB的这块区域)
![](https://yangbolong.github.io/images/lab2.mm.png)

最后在看具体的code之前我们整理一遍具体的boot流程
* 首先把内存上一块reserved区域载入BIOS代码(0xf0000-0x100000)
* 进行各种device初始化的设置
* 执行刚刚载入的BIOS代码，把硬盘的第一个sector载入到0x7c00与0x7DFF之间的区域
* BIOS代码跳转到0x7C00,将控制权交给bootloader
* bootloader的前一部分是汇编写的，主要任务是完成实模式到保护模式的跳转
* bootloader后一部分跳转到了C语言bootmain()函数，这个函数会找到disk上的kernel entry point然后正式进入内核，完成boot流程

```
9100 #include "asm.h"
9101 #include "memlayout.h"
9102 #include "mmu.h"
9103
9104 # Start the first CPU: switch to 32−bit protected mode, jump into C.
9105 # The BIOS loads this code from the first sector of the hard disk into
9106 # memory at physical address 0x7c00 and starts executing in real mode
9107 # with %cs=0 %ip=7c00.
9108
9109 .code16 # Assemble for 16−bit mode
9110 .globl start
9111 start:
9112 cli # BIOS enabled interrupts; disable
9113
9114 # Zero data segment registers DS, ES, and SS.
9115 xorw %ax,%ax # Set %ax to zero
9116 movw %ax,%ds # −> Data Segment
9117 movw %ax,%es # −> Extra Segment
9118 movw %ax,%ss # −> Stack Segment
9119
9120 # Physical address line A20 is tied to zero so that the first PCs
9121 # with 2 MB would run software that assumed 1 MB. Undo that.
9122 seta20.1:
9123 inb $0x64,%al # Wait for not busy
9124 testb $0x2,%al
9125 jnz seta20.1
9126
9127 movb $0xd1,%al # 0xd1 −> port 0x64
9128 outb %al,$0x64
9129
9130 seta20.2:
9131 inb $0x64,%al # Wait for not busy
9132 testb $0x2,%al
9133 jnz seta20.2
9134
9135 movb $0xdf,%al # 0xdf −> port 0x60
9136 outb %al,$0x60
9137
9138 # Switch from real to protected mode. Use a bootstrap GDT that makes
9139 # virtual addresses map directly to physical addresses so that the
9140 # effective memory map doesn’t change during the transition.
9141 lgdt gdtdesc
9142 movl %cr0, %eax
9143 orl $CR0_PE, %eax
9144 movl %eax, %cr0
9150 # Complete the transition to 32−bit protected mode by using a long jmp
9151 # to reload %cs and %eip. The segment descriptors are set up with no
9152 # translation, so that the mapping is still the identity mapping.
9153 ljmp $(SEG_KCODE<<3), $start32
9154
9155 .code32 # Tell assembler to generate 32−bit code now.
9156 start32:
9157 # Set up the protected−mode data segment registers
9158 movw $(SEG_KDATA<<3), %ax # Our data segment selector
9159 movw %ax, %ds # −> DS: Data Segment
9160 movw %ax, %es # −> ES: Extra Segment
9161 movw %ax, %ss # −> SS: Stack Segment
9162 movw $0, %ax # Zero segments not ready for use
9163 movw %ax, %fs # −> FS
9164 movw %ax, %gs # −> GS
9165
9166 # Set up the stack pointer and call into C.
9167 movl $start, %esp
9168 call bootmain

9200 // Boot loader.
9201 //
9202 // Part of the boot block, along with bootasm.S, which calls bootmain().
9203 // bootasm.S has put the processor into protected 32−bit mode.
9204 // bootmain() loads an ELF kernel image from the disk starting at
9205 // sector 1 and then jumps to the kernel entry routine.
9206
9207 #include "types.h"
9208 #include "elf.h"
9209 #include "x86.h"
9210 #include "memlayout.h"
9211
9212 #define SECTSIZE 512
9213
9214 void readseg(uchar*, uint, uint);
9215
9216 void
9217 bootmain(void)
9218 {
9219 struct elfhdr *elf;
9220 struct proghdr *ph, *eph;
9221 void (*entry)(void);
9222 uchar* pa;
9223
9224 elf = (struct elfhdr*)0x10000; // scratch space
9225
9226 // Read 1st page off disk
9227 readseg((uchar*)elf, 4096, 0);
9228
9229 // Is this an ELF executable?
9230 if(elf−>magic != ELF_MAGIC)
9231 return; // let bootasm.S handle error
9232
9233 // Load each program segment (ignores ph flags).
9234 ph = (struct proghdr*)((uchar*)elf + elf−>phoff);
9235 eph = ph + elf−>phnum;
9236 for(; ph < eph; ph++){
9237 pa = (uchar*)ph−>paddr;
9238 readseg(pa, ph−>filesz, ph−>off);
9239 if(ph−>memsz > ph−>filesz)
9240 stosb(pa + ph−>filesz, 0, ph−>memsz − ph−>filesz);
9241 }
9242
9243 // Call the entry point from the ELF header.
9244 // Does not return!
9245 entry = (void(*)(void))(elf−>entry);
9246 entry();
9247 }
9248
9249 
```
注意extended inline asm的语法，我们会经常用到
```
asm ( assembler template 
           : output operands                  /* optional */
           : input operands                   /* optional */
           : list of clobbered registers      /* optional */
           );
```

```
asm ("movl %0,%%eax;
              movl %1,%%ecx;
              call _foo"
             : /* no outputs */
             : "g" (from), "g" (to)
             : "eax", "ecx"
             );

```
%0代表output operand, %1代表input operand,g是constraint,clobber list表示告诉gcc自己可能要用这几个register.

在bootmain()里我们处理了一个ELF格式的binary文件，要理解代码我们需要了解ELF格式
```
 struct elfhdr {
0956 uint magic; // must equal ELF_MAGIC
0957 uchar elf[12];
0958 ushort type;
0959 ushort machine;
0960 uint version;
0961 uint entry;
0962 uint phoff;
0963 uint shoff;
0964 uint flags;
0965 ushort ehsize;
0966 ushort phentsize;
0967 ushort phnum;
0968 ushort shentsize;
0969 ushort shnum;
0970 ushort shstrndx;
0971 };
```
![](https://wiki.osdev.org/File:Elfdiagram.png)

最后是关于段式寻址的使用
如果不了解段式寻址可以看OSTEP的有关章节[Segment](http://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf)
简单来说就是通过base和bounds两个指针划定了一块区域
系统主要有三个descriptor表,GDT,LDT,IDT.这里我们用的是GDT
对于CPU来说，这些表的形式为(size, linear address)

一个GDT的例子
```
GDT[0] = {.base=0, .limit=0, .type=0};                     // Selector 0x00 cannot be used
GDT[1] = {.base=0, .limit=0xffffffff, .type=0x9A};         // Selector 0x08 will be our code
GDT[2] = {.base=0, .limit=0xffffffff, .type=0x92};         // Selector 0x10 will be our data
GDT[3] = {.base=&myTss, .limit=sizeof(myTss), .type=0x89}; // You can use LTR(0x18)
```

## 内存模型
计算机的内存管理机制是Page + Segmentation. 
首先看一下page的机制
![](https://images2018.cnblogs.com/blog/1286804/201807/1286804-20180713113554366-611065232.png)
![](http://www.cs.uni.edu/~fienup/cs142f05/lectures/4360ceb7.jpg)
怎么理解logical address与linear address呢？我的理解是把它们看做存储结构的index会比较好理解。

### PDE,PTE,PF
我们以xv6的三级页表为例来讲解内存模型。
PDE全称page directory entry,是page directory的一项，里面存的是一个page table,CR3存放了指向page directory table的指针
PTE全称page table entry,里面存放了page frame
PF就是page frame,具体的物理页
pde与pte的结构是类似的，可以理解成一个线性表，地址就是这个线性表的index(当然为了存储空间着想，很多位置都是没有初始化的，同时大多数页表也采用了COW技术)。除了具体内容以外，还存放了大量的权限位，包括读写权限，也包括诸如PSE这类的大页设定参数。

![](http://pekopeko11.sakura.ne.jp/unix_v6/xv6-book/en/_images/F2-2.png)
```
1804 static struct kmap {
1805 void *virt;
1806 uint phys_start;
1807 uint phys_end;
1808 int perm;
1809 } kmap[] = {
1810 { (void*)KERNBASE, 0, EXTMEM, PTE_W}, // I/O space
1811 { (void*)KERNLINK, V2P(KERNLINK), V2P(data), 0}, // kern text+rodata
1812 { (void*)data, V2P(data), PHYSTOP, PTE_W}, // kern data+memory
1813 { (void*)DEVSPACE, DEVSPACE, 0, PTE_W}, // more devices
1814 };
```
