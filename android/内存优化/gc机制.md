# GC
在Android的高级系统版本中，针对Heap空间（堆）既对象内存，有一个Generational Heap Memory的模型，其中将整个内存分为三个区域：

 - Young Generation（年轻代）
 - Old Generation（年老代）
 - Permanent Generation（持久代）（方法区）   
![](../images/gc.jpg)


1. Young Generation
由一个Eden区和两个Survivor区组成，程序中生成的大部分新的对象都在Eden区中，当Eden区满时，还存活的对象将被复制到其中一个Survivor区，当次Survivor区满时，此区存活的对象又被复制到另一个Survivor区，当这个Survivor区也满时，会将其中存活的对象复制到年老代。
2. Old Generation
一般情况下，年老代中的对象生命周期都比较长。
3. Permanent Generation
用于存放静态的类和方法，持久代对垃圾回收没有显著影响。

## GC过程
### 总结：内存对象的处理过程如下：

 - 对象创建后在Eden区。
 - 执行GC后，如果对象仍然存活，则复制到S0区。
 - 当S0区满时，该区域存活对象将复制到S1区，然后S0清空，接下来S0和S1角色互换。
 - 当第3步达到一定次数（系统版本不同会有差异）后，存活对象将被复制到Old Generation。
 - 当这个对象在Old Generation区域停留的时间达到一定程度时，它会被移动到Old Generation，
 - 系统在Young Generation、Old Generation上采用不同的回收机制。每一个Generation的内存区域都有固定的大小。随着新的对象陆续被分配到此区域，当对象总的大小临近这一级别内存区域的阈值时，会触发GC操作，以便腾出空间来存放其他新的对象。
 - 如果老生代的空间也被占满，当来自新生代的对象再次请求进入老生代时就会报OutOfMemory异常。
 - JVM内存模型中的方法区更被人们倾向的称为永久代（Perm Generation），保存在永久代中的对象一般不会被回收。其永久代进行垃圾回收的频率就较低，速度也较慢。永久代的垃圾收集主要回收废弃常量和无用类。
### 执行GC占用的时间与Generation和Generation中的对象数量有关：

 - Young Generation < Old Generation < Permanent Generation
 - Gener中的对象数量与执行时间成正比。

## Young Generation GC
- 由于其对象存活时间短，因此基于Copying算法（扫描出存活的对象，并复制到一块新的完全未使用的控件中）来回收。新生代采用空闲指针的方式来控制GC触发，指针保持最后一个分配的对象在Young Generation区间的位置，当有新的对象要分配内存时，用于检查空间是否足够，不够就触发GC。
## Old Generation GC
- 由于其对象存活时间较长，比较稳定，因此采用Mark（标记）算法（扫描出存活的对象，然后再回收未被标记的对象，回收后对空出的空间要么合并，要么标记出来便于下次分配，以减少内存碎片带来的效率损耗）来回收。

## gc类型
###  在Android系统中，GC有三种类型：
- kGcCauseForAlloc：分配内存不够引起的GC，会Stop World。由于是并发GC，其它线程都会停止，直到GC完成。
- kGcCauseBackground：内存达到一定阈值触发的GC，由于是一个后台GC，所以不会引起Stop World。
- kGcCauseExplicit：显示调用时进行的GC，当ART打开这个选项时，使用System.gc时会进行GC。

 - 尽量少用finalize函数。finalize函数是Java提供给程序员一个释放对象或资源的机会。但是，它会加大GC的工作量，因此尽量少采用finalize方式回收资源。
## GC日志查看


D/dalvikvm(7030)：GC_CONCURRENT freed 1049K, 60% free 2341K/9351K, external 3502K/6261K, paused 3ms 3ms
### GC_CONCURRENT是当前GC时的类型，GC日志中有以下几种类型：

 - GC_CONCURRENT：当应用程序中的Heap内存占用上升时（分配对象大小超过384k），避免Heap内存满了而触发的GC。如果发现有大量的GC_CONCURRENT出现，说明应用中可能一直有大于384k的对象被分配，而这一般都是一些临时对象被反复创建，可能是对象复用不够所导致的。
 - GC_FOR_MALLOC：这是由于Concurrent GC没有及时执行完，而应用又需要分配更多的内存，这时不得不停下来进行Malloc GC。
 - GC_EXTERNAL_ALLOC：这是为external分配的内存执行的GC。
 - GC_HPROF_DUMP_HEAP：创建一个HPROF profile的时候执行。
 - GC_EXPLICIT：显示调用了System.GC()。（尽量避免）  
再回到上面打印的日志:
freed 1049k 表明在这次GC中回收了多少内存。
60% free 2341k/6261K 表明回收后60%的Heap可用，存活的对象大小为2341kb，heap大小是9351kb。
external 3502/6261K 是Native Memory的数据。存放Bitmap Pixel Data（位图数据）或者堆以外内存（NIO Direct Buffer）之类的。第一个值说明在Native Memory中已分配3502kb内存，第二个值是一个浮动的GC阈值，当分配内存达到这个值时，会触发一次GC。
paused 3ms 3ms 表明GC的暂停时间，如果是Concurrent GC，会看到两个时间，一个开始，一个结束，且时间很短，如如果是其他类型的GC，很可能只会看到一个时间，且这个时间是相对比较长的。并且，越大的Heap Size在GC时导致暂停的时间越长。