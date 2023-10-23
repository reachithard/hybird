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

### Modified(修改)：

### Execlusive(独占)：

### Shared(共享)：

### Invalid(失效)：