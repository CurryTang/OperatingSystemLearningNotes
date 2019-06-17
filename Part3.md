# Part 3
这一部分我们会关注这门课程中可以说是最核心的部分，系统调度、进程与中断等等机制，它们之间是相互紧密联系在一起的。

## 进程
进程的定义有很多，这里列举两个个
1. program in execution + stack, registers + temporary data ( such as return addresses, subroutine parameters ) + program counter + other activities
2. one address space + one or more threads of computation
主要就是程序就是程序的运行时信息

那么什么是线程呢？
我们知道线程肯定不等于进程，线程的定义如下
A thread is an abstraction that contains 
the minimal state that is necessary to stop an active and resume it at some point later
说的就是线程是进行调度所需要的最小单位 那么说到了调度的单位，我们可以想到这个东西应该是CPU dependent,不同的instruction set应该有不同的thread

xv6里的线程是一个struct,记录了五个register, edi, esi, ebx, ebp, eip

一个Process有自己的process state
* new:  The process is being created
* running:  Instructions are being executed
* waiting:  The process is waiting for some event to occur
* ready:  The process is waiting to be assigned to a processor
* terminated:  The process has finished execution
调度器会根据这些信息来进行调度
下面的是一个process control block,可以理解成这个进程的户口本
* process state
* program counter ( PC )
* CPU registers
* memory-management information
* accounting information ( e.g. time limits, process #, ... )
* I/O status information ( e.g. list of open files by the process )
 一种可能的Process table
 ![](http://cse.csusb.edu/tongyu/courses/cs460/images/process/pt.png)
 
 ### 进程的创建
 以Unix为例，通过fork()来新建一个进程，具体地说，是对parent的address space做一模一样的拷贝，如果要运行别的程序，可以使用exec()来替换address space
 父进程和子进程通过Pid来进行区分
 ```
 int
fork(void)
{
  int i, pid;
  struct proc *np;

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy process state from p.
  if((np->pgdir = copyuvm(proc->pgdir, proc->sz)) == 0){
    kfree(np->kstack);
    np->kstack = 0;
    np->state = UNUSED;
    return -1;
  }
  np->sz = proc->sz;
  np->parent = proc;
  *np->tf = *proc->tf;

  // Clear %eax so that fork returns 0 in the child.
  np->tf->eax = 0;

  for(i = 0; i < NOFILE; i++)
    if(proc->ofile[i])
      np->ofile[i] = filedup(proc->ofile[i]);
  np->cwd = idup(proc->cwd);

  safestrcpy(np->name, proc->name, sizeof(proc->name));

  pid = np->pid;

  acquire(&ptable.lock);

  np->state = RUNNABLE;

  release(&ptable.lock);

  return pid;
}
 ```
 ### 进程的调度
 调度这一块主要有两部分内容，一部分是context switch实现的思路，一部分是具体地scheduling 算法，后面这一部分主要是一些概念性的内容。
 关于调度算法，比较基本的有round robin, shortest job first这类，比较符合直觉，听名字就可以猜出个大概。
 比较复杂的是multi-level feedback queue,基本思想是引入priority概念，运行满了时间片就降priority,没满就不变，也就是达到fair的效果。
 context switch这一块就比较复杂了，
 先回忆一下calling convention
 ![](https://img-blog.csdn.net/20131124171848671?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2FuZ3llemkxOTkzMDkyOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
 ```
 .globl swtch
swtch:
  movl 4(%esp), %eax
  movl 8(%esp), %edx

  # Save old callee-save registers
  pushl %ebp
  pushl %ebx
  pushl %esi
  pushl %edi

  # Switch stacks
  movl %esp, (%eax)
  movl %edx, %esp

  # Load new callee-save registers
  popl %edi
  popl %esi
  popl %ebx
  popl %ebp
  ret
 ```
 这是主要的context switch函数，做了这样的事，先保存了调用者保存寄存器，然后换栈，`movl %esp, (%eax)`把老的esp保存到了第一个参数里，如果你看C的话是`**context`.
 sched => switch to Scheduler => switchkvm (to kernel) => switchuvm (to user) => switch to P => sched
 ![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/sched.JPG)
 ![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/three.JPG)
 
 ### 线程
 PC + registers + stack space
 
 xv6里：上面写过，一个struct,包含五个寄存器 
 
> a thread shares with peer threads its coded section, data section, and OS resources ( such as open files and signals ) ~ task
> traditionally heavy weight process = thread + task

![](http://cse.csusb.edu/tongyu/courses/cs460/images/process/process-thread.png)

很多library实现了user-level thread,就是我们后端开发里常用的那些
![](http://cse.csusb.edu/tongyu/courses/cs460/images/process/thread-stack.png)

对比process和thread
* both has states new, ready, running, blocked, terminated
* both can create child
* each process operates independently of the others, has its own PC, stack pointer and address space
* threads are not independent of each other; threads can read or write over any other's stack
线程也是我们调度的最小单位

> 进程，直观点说，保存在硬盘上的程序运行以后，会在内存空间里形成一个独立的内存体，这个内存体有自己的地址空间，有自己的堆，上级挂靠单位是操作系统。
> 操作系统会以进程为单位，分配系统资源，所以我们也说，进程是资源分配的最小单位。线程存在与进程当中，是操作系统调度执行的最小单位。说通俗点，线程就是干活的。
引用自 https://blog.csdn.net/tennysonsky/article/details/45030645 


可以在user-level实现，也可以在kernel-level实现
![](http://cse.csusb.edu/tongyu/courses/cs460/images/process/user-thread.png)

multiplexing
![](http://cse.csusb.edu/tongyu/courses/cs460/images/process/hybrid.png)

线程通过yield进行调度执行的流程
![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/yield.JPG)

最后介绍一下Linux里的线程和进程
准确来说，Linux内核里并没有标准意义上的线程的概念，每一个执行实体都有自己对应的task_struct结构；于此同时，我们可以通过clone()创建子进程并且选择性地使用CLONE_FLAGS来指定共享的资源，从而获得light weight process

Linux地用户态地pthread就是基于light weight process来实现，之后在用户地角度，每一个task struct对应一个线程，一组进程和背后的资源被认为是进程
关于pthread,POSIX制定了一些标准

1. 查看进程列表的时候，相关的一组 task_struct 应当被展现为列表中的一个节点；
2. 发送给这个“进程”的信号（对应 kill 系统调用）， 将被对应的这一组 task_struct 所共享，并且被其中的任意一个“线程”处理；
3. 发送给某个“线程”的信号（对应 pthread_kill）， 将只被对应的一个task_struct接收，并且由它自己来处理；
4. 当“进程”被停止或继续时（对应 SIGSTOP/SIGCONT 信号）， 对应的这一组 task_struct 状态将改变；
5. 当“进程”收到一个致命信号（比如由于段错误收到 SIGSEGV 信号）， 对应的这一组 task_struct 将全部退出；

早期地LinuxThreads实现只实现了第五项要求，后期的NPTL实现了全部的五项要求，主要是有了系统内核上的支持

最后是Linux里地进程结构thread_info,非常经典的两个page的设计
![](https://img-blog.csdn.net/20160730163254645)

## IPC
接下来是一个OS里非常重要的概念 IPC.

IPC的定义 Mechanism for processes to communicate and to synchronize their actions

主要实现两个接口 send() receive()

如果让你自己设计一个IPC的model，你会想到哪些？
![](https://github.com/CurryTang/OperatingSystemLearningNotes/blob/master/ipc.JPG)
这两种模型是很符合直觉的，一种通过syscall trap到内核态再返回，另一种是使用shared memory,当然了肯定又是各有各的tradeoff

在介绍了这种模型之后，我们实现IPC的问题其实就回归到了实现Message passing机制的老问题上，如何维护一个支持多线程的passing结构?

一种比较经典的结构就是线性表，当然是带锁的线性表,这种结构有很多名字，比如bounded buffer,pipe之类的
我们看看xv6如何实现ipc
```
struct pipe {
  struct spinlock lock;
  char data[PIPESIZE];
  uint nread;     // number of bytes read
  uint nwrite;    // number of bytes written
  int readopen;   // read fd is still open
  int writeopen;  // write fd is still open
};

int
pipewrite(struct pipe *p, char *addr, int n)
{
  int i;

  acquire(&p->lock);
  for(i = 0; i < n; i++){
    while(p->nwrite == p->nread + PIPESIZE){  //DOC: pipewrite-full
      if(p->readopen == 0 || proc->killed){
        release(&p->lock);
        return -1;
      }
      wakeup(&p->nread);
      sleep(&p->nwrite, &p->lock);  //DOC: pipewrite-sleep
    }
    p->data[p->nwrite++ % PIPESIZE] = addr[i];
  }
  wakeup(&p->nread);  //DOC: pipewrite-wakeup1
  release(&p->lock);
  return n;
}

int
piperead(struct pipe *p, char *addr, int n)
{
  int i;

  acquire(&p->lock);
  while(p->nread == p->nwrite && p->writeopen){  //DOC: pipe-empty
    if(proc->killed){
      release(&p->lock);
      return -1;
    }
    sleep(&p->nread, &p->lock); //DOC: piperead-sleep
  }
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(p->nread == p->nwrite)
      break;
    addr[i] = p->data[p->nread++ % PIPESIZE];
  }
  wakeup(&p->nwrite);  //DOC: piperead-wakeup
  release(&p->lock);
  return i;
}
```
其实基本就是一个非常基本的reader()/writer()

IPC有两种实现模式，既然是通信，那么就有direct/indirect之分

对于direct来说，P直接发送message到Q

对于indirect来说，P先把message发送到一个中间人信箱，Q再从信箱里把message取出来

POSIX里实现了shared memory模式的一套API
```
Process first creates shared memory segment

segment id = shmget(IPC PRIVATE, size, S IRUSR | S IWUSR);

Process wanting access to that shared memory must attach to it

shared memory = (char *) shmat(id, NULL, 0);

Now the process could write to the shared memory

sprintf(shared memory, "Writing to shared memory");

When done a process can detach the shared memory from its address space

shmdt(shared memory);
```

## 中断与异常
中断 a signal generated by a hardware device
异常 an illegal program action
system call a user program can ask for an operating system service

### 异常
Exception的处理
![](https://i.stack.imgur.com/PdHc5.jpg)
最后的address of entry #k = exception number * 4 + exception table base register
![](https://images.slideplayer.com/28/9297804/slides/slide_18.jpg)
在系统中真正实现的exception table interrupt descriptor table(IDT)
每一个IDT的单元都是一个struct
```
struct IDTDescr {
   uint16_t offset_1; // offset bits 0..15
   uint16_t selector; // a code segment selector in GDT or LDT
   uint8_t zero;      // unused, set to 0
   uint8_t type_attr; // type and attributes, see below
   uint16_t offset_2; // offset bits 16..31
};
```
![](http://www.scs.stanford.edu/05au-cs240c/lab/i386/fig9-1.gif)
处理一个exception的时候，下面的结构会被push到kernel stack上，意味着具有完全的运行权限
![](http://sop.upv.es/gii-dso/en/t5-llamadas-al-sistema/int_stack.png)
