# 虚拟化

> “Any problem in computer science can be solved with another level of indirection.”

Virtualization software constructs an isomorphism from guest to host.

三大特性
* isolation
* encapsulation
* interposition

那么如何实现virtualization呢？
三层抽象
* API 
* ABI
* ISA

两种虚拟机的类型 System Virtual Machine Monitor
1. type 1 直接运行在hardware上 例如Xen
2. type 2 运行在Host OS上 例如Vmware Workstation

三种架构 Xen, KVM, QEMU

VMM不处理interrupt,直接转交给Host OS处理

![](https://www.researchgate.net/profile/Ioannis_Papapanagiotou3/publication/320223370/figure/fig2/AS:546002391900160@1507188522425/Hypervisor-based-vs-Container-based-Virtualization.png)
Hypervisor的架构

## CPU虚拟化
虚拟化的一个问题是权限问题，对于我们的虚拟机，由于运行在用户态，没有权限去执行诸如关中断这样的需要高权限的操作
那么我们需要使用trap & emulate

x86下有很多指令无法虚拟化，有几种解决办法
* instruction interpretation 通过软件来模拟fetch/decode/execute
* binary translation 把命令翻译成function call 问题 interrupt
* 参数虚拟化
* 硬件支持的CPU虚拟化 root / non-root operation

常见的虚拟化技术
1. VT-x
root / non-root mode
* VM-X root operation fully-privileged, intended for VM monitor
* VM-X non-root operation not fully-privileged 
通过VMCS 来控制VT-x 虚拟化的CPU

VM Entry:
* Transition from VMM to Guest 
* Enters VMX non-root operation 
* Loads Guest state from VMCS
* VMLAUNCH used on initial entry
* VMRESUME used on subsequent entries

VM Exit
* VMEXIT instruction used on transition from Guest to VMM
* Enters VMX root operation
* Saves Guest state in VMCS
* Loads VMM state from VMCS

内存虚拟化
GVA->GPA->HPA

solution 
1. shadow paging
![](https://ai2-s2-public.s3.amazonaws.com/figures/2017-08-08/f41478e7898e808294a8bc35ee26aef56ee6c0b3/5-Figure3-1.png)
Shadow page table完成HPA到GPA的转换
* VMM intercepts guest OS setting the virtual CR3
* VMM iterates over the guest page table, constructs a corresponding shadow page table
* In shadow PT, every guest physical address is translated into host physical address
* Finally, VMM loads the host physical address of the shadow page table
那么GuestOS想要修改page table怎么办呢？
解决办法是把GuestOS的page table设为read only,一旦想要修改，就会引发page fault,然后trap到内核态继而更新shadow page table

2. direct paging
去除GPA,保留GVA,HPA,使得guestOS有能力直接管理guest page table.

3. 硬件支持
Intel EPT

Memory Ballooning 
Baloon tells VMM to recycle
its “private” pinned pages

I/O 虚拟化

KVM
虚拟机这时只是一个QEMU的进程
分配内存时，会通过QEMU来分配，这些内存在guestOS的严重就是physical memory
ioctl->kernel->native guest execution->kernel exit->userspace exit
ioctl()之后，
The kernel find the VMCS data structure
Load VMCS to processor by VMENTRY
The processor switches from root-mode to non-root
The IP is changed to VMCS->IP, and start to run guest VM’s code

virtIO
![](https://developer.ibm.com/developer/articles/l-virtio/images/figure2.gif)

virtio disk 读磁盘过程
1. Guest fills in request handlers
2. Guest writes to virtio-blk virtqueue notify register
3. QEMU issues I/O request on behalf of guest
4. QEMU fills in request footer and injects completion interrupt
5. guest receives interrupt and executes handler
6. guest reads data from buffer

Intel VT-d directed I/O
完成了interrupt,address,device的重映射
Main Memory <-- IOMMU <-- Device

DMA-remapping 完成了DMA request到实际物理地址的translation
DMA-remapping translates the address of the incoming DMA request to the correct physical memory address and perform checks for permissions to access that physical address, based on the information provided by the system software. 

Interrupt remapping
The VT-d interrupt-remapping architecture addresses this problem by redefining the interrupt-message format.
Interrupt requests specify a requester-ID and interrupt-ID, and remap hardware transforming these requests to a physical interrupt.

SR-IOV 

Serverless Computing
trigger event-> platform gateway -> instantiate according to function codes -> compute -> result return to user/other functions (chained functions)
serveless application latency的来源 : initialization
解决 func-image compilation && func load





