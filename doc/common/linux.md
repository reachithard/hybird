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

sequence before

happen before

