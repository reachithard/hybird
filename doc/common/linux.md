# 操作系统

## 计算机结构

regs:寄存器，在x86_64中一般有rip，rbp，rsp，rax，rbx，rcx，rdx，rsi，rdi，r8-r15

L1，L2，L3cache：高速缓存，在i7架构中，L1分为数据cache和指令cache，L2在核心内，L3被每个核共享

Main Memory: 主存，是cpu和硬盘的中转站

磁盘：即硬盘，存储在本地

远端存储：存储在其他计算机

cpu总线，pci，isa总线，南桥，北桥

## CPU访存

cpu(虚拟地址)->[MMU(TLB)]->Cache->[MMU(TLB)]->内存[页表]

物理cache

逻辑cache

### TLB歧义问题

asid解决TLB歧义问题

## Cache写回

数据回写

写通过

预取

## MESI协议

MESI协议是一种缓存一致性协议，用于多处理系统中保持缓存一致性。MESI为：

**Modified(修改)：**

缓存行被修改，且该缓存行是唯一的，即其他缓存中没有该缓存行的副本。

**Execlusive(独占)：**

缓存行被缓存在一个处理器的缓存中，且该缓存行是唯一的，即其他缓存没有该缓存行的副本。

**Shared(共享)：**

缓存行被缓存在多个处理器缓存中。且该缓存行是只读的。

**Invalid(失效)：**

缓存行无效，即该缓存行中不包含有效数据。

MESI协议通过这四个状态来保证缓存的一致性，当一个处理器修改了一个缓存行时，它会将该缓存行的状态从shared或者Execlusive状态变为Modified。并通知其他缓存行将该缓存状态变为Invalid，这样当其他处理器要访问该缓存时，则会从主存中重新读取数据，如下：

|                 步骤                 | core 1 | core 2 |
| :----------------------------------: | :----: | :----: |
|               core1读a               |   E    |   无   |
|               core2读a               |   S    |   S    |
|          core1写a（cache）           |   M    |   I    |
|          core1写回a（主存）          |   E    |   I    |
| 总线嗅探机制通知core2，变量a已经修改 |   S    |   S    |

## 内存序问题

内存顺序描述了计算机cpu获取内存的顺序，内存的排序可能发生在编译期间，也可能发生在cpu指令执行期间，如编译器优化，多发射等。

为了高效率的使用计算机资源的利用率和性能，编译器会对代码进行重新排序，CPU也会对代码进行重新排序，延缓执行等，以达到更好的执行效率。

通常，这些在单线程情况下都不会有问题，但在多线程的情况下，如无锁数据结构，指令乱序将会造成数据竞争等无法预测的行为。一般使用内存栅栏来解决，内存栅栏一般情况有锁内存和锁cache line的两种方式来解决。

**内存栅栏**

内存栅栏是一个令 CPU 或编译器在内存操作上限制内存操作顺序的指令，通常意味着在 barrier 之前的指令一定在 barrier 之后的指令之前执行。
在 C11/C++11 中，引入了六种不同的 memory order，可以让程序员在并发编程中根据自己需求尽可能降低同步的粒度，以获得更好的程序性能。

**sequence before**

在同一个线程，按照程序的代码序执行。

**happen before**

不同线程间，对于同一个原子操作，需要同步关系，store操作一定要先于 load，也就是说 对于一个原子变量x，先写x，然后读x是一个同步的操作，读x并不会读取之前的值，而是写x后的值。

### memory order

**memory_order_relaxed**

不对执行顺序做保证，没有happens-before的约束，编译器和处理器可以对memory access做任何的reorder，这种模式下能做的唯一保证，就是一旦线程读到了变量var的最新值，那么这个线程将再也见不到var修改之前的值了。

**memory_order_acquire 和memory_order_release**

memory_order_acquire保证本线程中,所有后续的读操作必须在本条原子操作完成后执行。memory_order_release保证本线程中,所有之前的写操作完成后才能执行本条原子操作。
acquire/release与顺序一致性内存序相比是更宽松的内存序模型，其不具有全局序，性能更高。核心是：同一个原子变量的release操作同步于一个acquire操作.。
通常的做法是：将资源通过store+memory_order_release的方式”Release”给别的线程；别的线程则通过load+memory_order_acquire判断或者等待某个资源，一旦满足某个条件后就可以安全的“Acquire”消费这些资源了。即释放获得顺序。

**memory_order_consume**

这个内存屏障与memory_order_acquire的功能相似，而且大多数编译器并没有实现这个屏障，而且正在修订中，暂时不鼓励使用 memory_order_consume 。
std::memory_order_consume具有弱的同步和内存序限制，即不会像std::memory_order_release产生同步与关系。
目前一般编译器的行为是类似memory_order_acquire 的。

**memory_order_acq_rel**

双向读写内存屏障，相当于结合了memory_order_release、memory_order_acquire。可以看见其他线程施加 release 语义的所有写入，同时自己的 release 结束后所有写入对其他施加 acquire 语义的线程可见
表示线程中此屏障之前的的读写指令不能重排到屏障之后，屏障之后的读写指令也不能重排到屏障之前。此时需要不同线程都是用同一个原子变量，且都是用memory_order_acq_rel

**memory_order_seq_cst**

通常情况下，默认使用 memory_order_seq_cst
如果是读取就是 acquire 语义，如果是写入就是 release 语义，如果是读取+写入就是 acquire-release 语义
同时会对所有使用此 memory order 的原子操作进行同步，所有线程看到的内存操作的顺序都是一样的，就像单个线程在执行所有线程的指令一样。

## c++内存模型

**Sequential Consistency**

**Relax**

**Acquire-Release**

## C++程序

代码段

数据段

Bss段

堆

栈

虚拟地址

虚拟内存

虚拟内存实现

虚拟存储器

页面置换算法

Lunix的buddy的slab算法

malloc实现：brk和mmap

## 进程

进程状态

僵尸进程

孤儿进程

进程间通信

进程同步方式

死锁及其形成

进程调度算法

fork vfork clone

## 线程



## 协程

协程调度