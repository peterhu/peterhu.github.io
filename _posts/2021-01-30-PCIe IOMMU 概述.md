---
layout: mypost
title: PCIe IOMMU 概述
categories: [PCIe]
---

IOMMU全称是Input/Output Memory Management Unit，翻译过来就是输入输出的内存管理单元。我们知道CPU里面也有一个MMU , 它是为为了支持多进程的虚拟地址共享同一个物理内存，以及对物理地址的访问进行权限检查的一个硬件单元。IOMMU作为一个内存管理单元，它提供的是设备端的地址翻译能力，它让具有DMA (Direct Memory Access)能力的设备可以使用虚拟地址，然后经过IOMMU翻译成可以直接访问内存的物理地址。设备端的地址翻译功能（DMA重定向）只是IOMMU的功能之一，IOMMU另一个重要的功能是中断重映射。IOMMU 通常是实现在北桥之中，现在北桥通常被集成进SOC中了，所以IOMMU通常都放在SOC内部了。不同芯片厂商的实现大同小异，可是名字却不太一样。Intel的芯片上叫做 VT-d （Virtualization Technology for Directed I/O ），AMD还是叫做IOMMU。我对Intel平台更熟悉一些，所以接下来的介绍都以VT-d 为例。

![IOMMUP](IOMMUP.png)

**DMA重映射(DMA Remapping)**

DMA重映射中最重要的概念就是地址的翻译。如下图所示，图的左侧是处理器的虚拟化，右侧则是IO的虚拟化。右边的设备1和设备2都想去访问0x4000的内存地址。DMA重映射单元能够将客户端物理地址（guest physical address)映射到主机端物理地址（host physical address）。最终其实设备1访问的是0x6000的主机端物理地址，设备2访问的是0x3000的主机端物理地址。

![DMAADRTR](DMAADRTR.png)

地址转换的过程是通过给每个PCIe设备建立一个翻译表（需要BIOS的支持），并且将表的地址填入到VT-d的Root Table Address 寄存器，这些翻译表和CPU的page table很类似而且提供了访问控制机制。DMA重映射单元收到设备发过来的GPA就会通过查表找到对应的HPA然后去访问真正的物理地址。DMA重映射具体的实现过程，会在另一篇文章中详述。

![DDMSRT](DDMSRT.png)

**中断重映射(Interrupt Remapping)**

IOMMU中断重映射单元让系统软件能够控制和集中处理外部的中断请求包括包括从I/O APIC发送出来的中断，以及设备,RP, RC产生的以MSI、MSI-X产生的中断（不包含中断重映射单元硬件本身产生的中断）。

Extended Capability Register寄存器bit3表示Interrupt Remapping Support为1，则表示支持中断重映射。


![ECR](ECR.png)

软件可以通过Program Global Command Register  Bit25 IRE 开启Interrupt Remapping。

![GCR](GCR.png)

没有Enable Interrupt Remapping时，设备中断请求格式称之为兼容格式，它的结构由32bit的Address和一个32bit的Data字段组成，
Address字段包含了中断要投递的目标CPU的APIC ID信息，Data字段主要包含了要投递的vecotr号和投递方式。 Address bit 4为Interrupt Format，用来表示这个Request是兼容格式（bit4=0）还是重映射格式 (bit 4=1)。之后设备的中断请求格式称之为重映射格式，其结构同样由一个32bit的Address和一个32bit的Data字段构成。但与兼容格式不同的是此时Adress字段不再包含目标CPU的APIC ID信息而是提供了一个16bit的HANDLE索引，并且Address的bit 4为"1"表示Request为重映射格式。硬件查询系统软件在内存中预设的中断重映射表 (Interrupt Remapping Table)来投递中断。中断重映射表由中断
重映射表项构成，每个IRTE占用16字节，中断重映射表的基地址存放在Interrupt Remapping Table Address Register中。

![IRT](IRT.png)

其实中断重映射就是指IOMMU会拦截I/O设备产生的中断，然后根据接收到的中断请求索引中断重映射表，根据找到的中断重映射表的表项产生新的中断请求，上传到CPU的LAPIC。通过中断重映射我们能够对同一物理系统中不同Domain的I/O设备中断请求进行隔离。另外为了让I/O设备能够产生可重映射的中断，
并对中断重映射表进行正确的索引，系统软件还需要对I/O设备的中断请求生成进行配置。关于中断重映射的具体实现过程，会在后续的文章中详述。


参考:
1. 《vt-directed-io-spec》
2. 《A_Tour_Beyond_BIOS_Using_Intel_VT-d_for_DMA_Protection》
3. VT-d Interrupt Remapping: https://kernelgo.org/interrupt-remapping.html
4. Intel VT-d（3）- 中断重映射: https://zhuanlan.zhihu.com/p/50785797