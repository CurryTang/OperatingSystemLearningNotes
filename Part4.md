## I/O 子系统

I/O子系统的目标：
1.提供统一的接口
2.提供合适的抽象

device的种类: character, block, network

输入a后的执行路径：
vector table --> trap --> kbdintr() --> kbdgetc() --> consputc()

inb()
asm volatile("in %1, %0" : "=a"(data) : "d"(port))

data是输出变量，port是输入变量 %1是输入 %0是输出 

三种i/O的模式
* blocking 同步模式，阻塞住直到好了
* non-blocking i/o 直接返回，不管了
* async 直接返回，好了叫我

DMA
![](https://image.slidesharecdn.com/8237-140305110327-phpapp01/95/8237-8257-dma-5-638.jpg?cb=1394017474)
简而言之就是不用过CPU了

i/o 通知的两种方式 polling/interrupt

如果一个用户在user level发起了i/o 请求，会发生什么
![](https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/images/Chapter13/13_13_IO_LifeCycle.jpg)

注意device driver和interrupt handler一样 分top half/bottom half

### File System 

VFS 提供了一个统一的接口 physical device --> driver --> vfs 

EXT 文件系统
inode的优点
Optimized for file systems with many small files

Each inode can directly point to 48KB of data

Only one layer of indirection needed for 4MB files

Faster file access

Greater meta-data locality  less random seeking

No need to traverse long, chained FAT entries

Easier free space management

Bitmaps can be cached in memory for fast access

inode and data space handled independently

为了解决Locality问题，引入block group,每个block group有自己的data structure
再分配时，尽量把相关的文件放进同一个block group

缺点是对于一个大文件来说，需要跨Block group存储

对于大文件来说，我们使用extent来进行管理，包含两个部分，address和length
![](https://slideplayer.com/slide/3266376/11/images/90/Example+B-Tree+ext4+uses+a+B-Tree+variant+known+as+a+H-Tree.jpg)


Crash Consistency

1. Just the data block (Db) is written to disk

What will happen?

2. Just the updated inode (I[v2]) is written to disk

What will happen?

3. Just the updated bitmap (B[v2]) is written to disk

What will happen?

三种情况，第一种没关系，第二种inconsistency,第三种space leakage

1. The inode (I[v2]) and bitmap (B[v2]) are written to disk, but not data (Db) garbage data

2. The inode (I[v2]) and the data block (Db) are written, but not the bitmap (B[v2]) metadata inconsistency

3. The bitmap (B[v2]) and data block (Db) are written, but not the inode (I[v2]) 数据丢失

文件操作的基本要求
Durable (Persistence): effects of operation are visible
Both a and b are visible

Atomic: all steps of operation visible or none
Either a and b are visible or none is visible

Ordered: exactly a prefix of operations is visible
If b is visible, then a is visible

三种recovery的手段
1. sync data update+fsck
2. soft update
3. logging

由于我们的文件系统没有给予durability的保证，我们如何确保durability以及consistency呢
我们可以通过fsync和rename(shadow copy)
前者是说我们操作完后手动把文件写到磁盘上，只有成功才返回
后者是说rename是一个原子操作，要么成功要么不变

fsck会检查整个文件系统，非常耗时


文件创建和文件删除的正确步骤
File creation: what's the right order of synchronous writes? 
1. mark inode as allocated 
2. create directory entry 

File deletion
1. erase directory entry 
2. erase inode addrs[], mark as free 
3. mark blocks free 

诸如write-back cache这样的机制可以提升性能，但是没有同步性的保证

基本思想 write-ahead log
在所有的操作后加一个flag "done"如果有这个done,那么在recovery的时候把write重新做一遍，没有的话就忽视

transaction的概念！提供了一系列操作的原子性

begin_trans: 
need to indicate which group of writes must be atomic! 
lock -- xv6 allows only one transaction at a time
log_write: 
record sector # 
append buffer content to log 
leave modified block in buffer cache (but do not write) 
commit_trans(): 
record "done" and sector #s in log 
do the writes 
erase "done" from log 
recovery: 
if log says "done": copy blocks from log to real locations on disk

### ext3的journaling


