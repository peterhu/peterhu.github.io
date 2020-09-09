---
layout: mypost
title: Multiprocessor Initialization
categories: [CPU]
---

## 
最近的几个项目都不止一次的碰到了MP 初始化的问题，每次都花了不少的时间，于是打算总结一下多处理器初始化的流程，以备将来再次碰到的问题时方便查找。
MP初始化指的是在一个有至少2个或者多个处理器的系统里，怎么去初始化所有的处理器，让系统工作起来。IA-32 ARCH 定义了MP 初始化的协议，该协议使得IA-32,X64的MP系统都可以boot，并且并不需要多余的特定的信号，或者指定特定的BSP.

# **1.BSP AP处理器:**
MP的系统里定义了2种类型的处理器: BSP 和 AP。BSP是boot strap processor， AP是指application processor。其实BSP和AP并没有本质的区别而且也不是固定的，BSP是由硬件动态选择的。其中一种算法是，上电以后每个处理器执行BIST(built in self test)，通过自检以后，大家都去monitor BNR(Bus Not Ready)的信号。如果BNR#一直在翻转，说明还没有ready。一旦BNR#停止翻转，每个处理器都尝试发一个NOP special cycle，第一个成功发出的就是BSP。BSP选出来以后，它会把IA32_APIC_BASE MSR里面的BSP flag设置起来。然后就会开始从reset vector开始执行。其它的AP就会进入"wait-for-SIPI state"。
	![MP](mp.png)
MP初始化协议算法：
在BSP，AP选出以后，通常BSP的初始化顺序为：
-  初始化内存。
-  加载microcode
-  初始化MTRRs
-  初始化Cache
-  加载AP start-up code到1Mbyte以下的4K内存中。
-  Enable APIC (SVR bit8）
-  Program ICR寄存器，把AP start-up code地址写到该寄存器
-  在AP start-up code里,每个AP将会增加一个COUNT变量表示AP已经起来了
-  广播INIT-SIPI-SIPI IPI sequence to the Aps,这时所有的AP才会真正被唤醒起来执行
	![Ini-sipi-sipi](init-sipi-sipi.png)

通常AP的初始化顺序为:
-  获取Lock信号量
-  加载microcode
-  初始化MTRR
-  Enable cache
-  增加COUNT表示AP起来了
-  释放信号量
-  CLI+HLT

在MP 初始化协议算法里有几个关键的机制,1. APIC (IPI,APIC ID,SVR) 2. Atomic operation and Spin Lock，需要专门的介绍一下：

# **2. APIC**
APIC 是高级可编程中断控制器,每个CPU都有一个。有两种模式APIC和X2APIC（支持更多的中断）mode。在MP初始化的过程中，我们需要enable APIC，SVR(0xFEE000F0)寄存器的bit8是用来en/disable它的
	![lapic](lapic.png)
APIC ID是每个处理器的编号，是有microcode生成的然后写APICID寄存器中，通常是8bits。 BSP一般都是0
	![apicid](apicid.png)
Inter-Processor Interrupt（IPI）是处理器间传递消息的机制，BSP会给AP发INIT IPI和SIPI。实现的机制是通过BSP去program APIC中的ICR寄存器去发IPI. Destination field 通常填的就是APIC ID
	![icr](icr.png)

# **3.  Atomic operation and Spin Lock**
原子操作原子操作在MP过程中在在计数器等地方，这类情况下数据有并发的危险，但是用锁去保护又显得有些浪费，所以使用原子类型操作。X86处理器提供了lock prefix可以帮住解决这样的场景。
```asm
 lock inc   dword [edi]
 lock dec   dword [rax]       ; (*CountTofinish)--
```
自旋锁是专为防止多处理器并发而引入的一种锁，它在应用于中断处理等部分(对于单处理器来说，防止中断处理中的并发可简单采用关闭中断的方式，不需要自旋锁)。
自旋锁最多只能被一个任务持有，如果一个processor任务试图请求一个已被争用(已经被持有)的自旋锁，那么这个任务就会一直进行忙循环——旋转——等待锁重新可用。
要是锁未被争用，请求它的processor任务便能立刻得到它并且继续进行。自旋锁可以在任何时刻防止多于一个的内核任务同时进入临界区，因此这种锁可有效地避免多处理器上并发运行的内核任务竞争共享资源。
```c
struct spinlock {
	int locked;
};
void spin_lock(struct spinlock *lock)
{
while (lock->locked || test_and_set(&lock->locked)); //atomic operation
}
void spin_unlock(struct spinlock *lock)
{
	lock->locked = 0;
}
```
Refer: 
- [1. sdm-vol-1-2abcd-3abcd.pdf](https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf) 
- [2. Multiprocessor Initialization](https://www.cs.usfca.edu/~cruse/cs630f08/lesson22.ppt)
- [3. spinlock前世今生](https://zhuanlan.zhihu.com/p/133445693)




