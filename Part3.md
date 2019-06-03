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
可以在user-level实现，也可以在kernel-level实现
![](http://cse.csusb.edu/tongyu/courses/cs460/images/process/user-thread.png)

multiplexing
![](http://cse.csusb.edu/tongyu/courses/cs460/images/process/hybrid.png)

