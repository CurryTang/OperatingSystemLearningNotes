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
JFS与log structured file system是不同的概念
前者不指定数据存储的方式 后者只包含一个log
JFS把每一次的更新操作封装成一个transaction(atomic) 直到commit block被写到磁盘上，才算一次完整的更新操作
我们的数据不会直接写到disk上去，而是先写到journal里，然后再resync到磁盘上
要完成一次commit 我们需要
```
Close the transaction
All subsequent filesystem operations will go into another transaction
Flush transaction to disk (journal), pin the buffers
After everything is flushed to the journal, update journal header blocks
Unpin the buffers in the journal only after they have been synced to the disk
Release space in the journal
```
ext3的结构
在内存里，我们需要存储
* 需要log的block number
* "handle"的集合

在硬盘上存储 
* FS
* circular log

ext3 log文件的结构
* log superblock
* descriptor block
* data block
* commit block

通过对commit进行batching,ext3提升了文件系统的性能

```
sys_open() {

    h = start()

    get(h, block #)

    modify the block in the cache

    stop(h)

}
```
做syscall时，通过start()和stop()把一个transaction包围起来，保证原子性
```
1. block new syscalls
2. wait for in-progress syscalls to stop()
3. open a new transaction, unblock new syscalls
4. write descriptor to log on disk w/ list of block #s
5. write each block from cache to log on disk
6. wait for all log writes to finish
7. write the commit record
8. wait for the commit write to finish
9. now cached blocks allowed to go to homes on disk (but not forced)
```

A: echo hi > x 
B: ls > y 
上一步的transaction无法读取到下一步transaction内发生的修改

当没有空间来分配commit log的时候，会阻塞掉行为，知道有空间

recovery的步骤
1. find the start of the log -- the first non-freed descriptor
     log "superblock" contains offset and seq# of first transaction
     (advanced when log space is freed)
2. find the end of the log
 scan until bad magic or not the expected seq number
 go back to last commit record
 crash during commit -> no commit record, recovery ignores
3. replay all blocks through last complete transaction
      in log order
      
> For each high-level file
> operation (e.g., write(),
> unlink()) that modifies the
> file system . . .
> • Write the blocks that
> would be updated into
> the journal
> • Once all blocks are in the
> journal, transaction is
> committed; now ext3 can
> issue the “in-place” writes
> to the actual data blocks
> and metadata blocks
> The journal is a circular buffer;
> asynchronously deallocate
> journal entries whose in-place
> updates are done

![](https://ai2-s2-public.s3.amazonaws.com/figures/2017-08-08/4468cbc8a9ad13ebeaa210424e842f158415ab07/5-Figure2-1.png)

### FAT32系统
cluster:存储文件的最小单位
文件系统结构：Super block + File allocation table + File and directory data

文件存储：非顺序地存储，链表式的存储，以0xFFFF代表终点

FAT32的文件命名 有一个长文件名 一个短文件名
长文件名使用0x0F作为flag
想要通过长文件名来搜索一个文件的数据，只能使用顺序搜索

FAT32的缺点是需要大量的seek
![](http://www.ntfs.com/images/recover-ROOT-structure.gif)
### NTFS
![](https://slideplayer.com/slide/7914625/25/images/2/NTFS+Partition+MBR+VBR+Directories+and+Files+%24Mft+Measured+in+Sectors.jpg)
NTFS的master file table是一个关系型的表，row是file record, column是file attribute
MFT的前16位用来存储metadata
对于系统上的每一个文件与文件夹，NTFS都会在MFT中存储一条，每条1KB
NTFS中的文件模型：每个文件都被视为一系列file attribute的集合
![](https://upload.wikimedia.org/wikipedia/commons/c/c9/Ntfs_mft.svg)


### Flash FS





