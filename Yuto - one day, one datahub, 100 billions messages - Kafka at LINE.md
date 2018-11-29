# One Day, One DataHub, 100 Billion Message: Kafka at LINE

这是LINE的工程师YUTO在KAFKA SUMMIT 2017上的一次次演讲，介绍了他甄别，改进KAFKA几个性能瓶颈的历程。

[Video](https://kafka-summit.org/sessions/single-data-hub-services-feed-100-billion-messages-per-day/)


## KAFKA-4614

Yuto在生产环境中偶尔会观察到这样的现象：系统会产生小数据量的磁盘读，且伴随着读操作，broker处理读请求的延迟会变大。由于消费者总是
读一个partition的last segment。而根据kafka的设计，last segment会被缓存在page cache中，因此这种情形应该不会发生。

为找到造成这个现象的终极原因，Yuto作了以下分析：

1. 使用监控工具观察到现象 
![Performance](/images/yotu_metrics.png)

2. 使用systemtap分析BIO_READ请求的来源
![systemtap](/images/yuto_systemtap.png)

3. 发现奇怪的munmap调用
![munmap](/images/yotu_munmap.png)

4. 分析kafka源代码，发现Kafka会把offset index的内容map到内存
![offset index](/images/yotu_offsetindex.png)

5. 找到munmap的调用者 - 原来时GC线程闯的祸
![caller](/images/yotu_caller.png)

## 分析

FileChannel.map()可以把文件的部分区域通过系统调用mmap()映射到JAVA进程的地址空间。这块被映射的内存只能通过MappedByteBuffer对象来访问。
MappedByteBuffer本身是驻留在JVM的heap中的，其生命周期由JVM的GC来控制；但它所管理的映射内存驻留在JVM的off-heap中。

当你关闭该MBB对应的RandomAccessFile和FileChannel，取消所有指向这个MBB对象的引用时，该MBB管理的这块内存仍然有效。只有当GC回收MBB对象时，
一个cleaner才会被触发，从而调用munmap()释放这块内存。

Kafka有一个log clean thread，它的一个动作是删除过期的segment以及对应的offset index文件。让我们看看删除offset index文件时发生了什么：
* 让程序内部的对象不再引用对应的MappedByteBuffer对象了（但这并不意味着该MBB会被马上GC）
* 删除文件
* 操作系统会将该文件对应的数据结构的引用计数减一。但由于MBB没有被GC，所以其内存映射仍然有效，所以此时的引用计数为一（而非零）

一会儿以后，GC开始工作了：
* 回收上述的MBB对象
* 这导致munmap被调用，从而使得该文件的引用计数变为0
* 引用技术变为0，使得VFS调用实际的文件系统代码来删除磁盘文件
* 这个删除的过程会产生磁盘读（比如读入inode），从而使得GC的STW过程异常的漫长（一般情况下，GC只涉及内存操作，比较快）

## 解决方案

解决方案很简单，但不需要一个MappedByteBuffer时，显示调用（而非被动等待GC触发）清理函数

mbb.asInstanceOf[sun.nio.ch.DirectBuffer].cleaner().clean()