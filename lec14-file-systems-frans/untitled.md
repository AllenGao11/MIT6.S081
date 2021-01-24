# 14.6 XV6创建inode代码展示

接下来我们通过查看XV6中的代码，更进一步的了解文件系统。因为我们前面已经分配了inode，我们先来看一下这是如何发生的。sysfile.c中有所有与文件系统相关的函数，分配inode发生在sys\_open函数中，因为这个函数会负责创建文件。

![](../.gitbook/assets/image%20%28612%29.png)

在sys\_open函数中，会调用create函数。

![](../.gitbook/assets/image%20%28610%29.png)

create函数中首先会解析路径名并找到最后一个目录，之后会查看文件是否存在，如果存在的话会返回错误。之后就会调用ialloc（inode allocate），这个函数会为文件x分配inode。ialloc函数位于fs.c文件中。

![](../.gitbook/assets/image%20%28615%29.png)

以上就是ialloc函数，与XV6中的大部分函数一样，它很简单，但是又不是很高效。它会遍历所有可能的inode编号，找到inode所在的block，再看位于block中的inode数据的type字段。如果这是一个空闲的inode将其type字段设置为文件，这会将inode标记为已被分配。函数中的log\_write就是我们之前看到在console中有关写block的输出。这里的log\_write是我们看到的整个输出的第一个。

以上就是第一次写磁盘设计到的函数调用。这里有个有趣的问题，如果有多个进程同时调用create函数会发生什么？对于一个多核的计算机进程可能并行运行，两个进程可能同时会调用到ialloc函数，然后调用bread（block read）函数。所以必须要有一些机制确保这两个进程不会互相影响。

让我们看一下位于bio.c的buffer cache代码。首先看一下bread函数

![](../.gitbook/assets/image%20%28589%29.png)

bread函数首先会调用bget函数，bget会为我们从buffer cache中找到block。让我们看一下bget函数

![](../.gitbook/assets/image%20%28616%29.png)

这里的代码还有点复杂。我猜你们之前已经看过这里的代码，那么这里的代码在干嘛？

> 学生回答：这里遍历了linked-list，来看看现有的cache是否符合要找的block。

是的，我们这里看一下block 33的cache是否存在，如果存在的话，将block对象的引用计数（refcnt）加1，之后再释放bcache锁，因为现在我们已经完成了对于cache的检查并找到了block cache。之后，代码会尝试获取block对应的buffer cache的锁。

所以，如果有多个进程同时调用bget的话，其中一个可以获取bcache的锁并扫描buffer cache。此时，其他进程是没有办法修改buffer cache的（注，因为bacche的锁被占住了）。之后，进程会查找block number是否在cache中，如果在的话将block cache的引用计数加1，表明当前进程对block cache有引用，之后再释放bcache的锁。如果有第二个进程也想扫描buffer cache，那么这时它就可以获取bcache的锁。假设第二个进程也要获取block 33的cache，那么它也会对相应的block cache加1。最后这两个进程都会尝试对block 33的buffer cache调用acquiresleep函数。

acquiresleep是另一种锁，我们称之为sleep lock，本质上来说它获取block 33 cache的锁。其中一个进程获取锁之后函数返回。在ialloc函数中会扫描block 33中是否有一个空闲的inode。而另一个进程会在acquiresleep中等待第一个进程完成所有的操作。

> 学生提问：当一个block cache的refcnt不为0时，可以更新block cache，因为在释放bcache锁之前，可能会发生一些事情。
>
> Frans教授：这里我想说几点；首先XV6中对bcache做任何修改的话，都必须持有bcache的锁；其次对block 33的cache做任何修改你需要持有block 33的sleep lock。所以在任何时候，release\(&bcache.lock\)之后，b-&gt;refcnt都大于0。block的cache只会在refcnt为0的时候才会被驱逐，任何时候refcnt大于0都不会驱逐block cache。所以当b-&gt;refcnt大于0的时候，block cache本身不会被buffer cache修改。这里的第二个锁，也就是block cache的sleep lock，是用来保护block cache的内容的。它确保了任何时候只有一个进程可以读写block cache。

如果buffer cache中有两份block 33的cache将会出现问题。
