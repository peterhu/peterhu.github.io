---
layout: mypost
title: IOMMU DMA Remapping
categories: [PCIe]
---

在PCIe IOMMU 概述中我们提到DMA重映射其实就是地址的翻译，将设备的GPA/GVA转化成HPA的过程。通常来自于PCIe设备上的内存地址请求有2类：

1. Requests without address-space-identifier: 这种是正常的endpoint devices的内存请求，请求内容仅包含read/write/atomics类型，DMA目的地的地址与大小、源设备标识。简称为Requests-without-PASID。

2. Requests with address-space-identifier: 这类请求需要包含额外信息以提供目标进程的地址空间标志符(PASID)，以及Execute-Requested (ER) flag和 Privileged-mode-Requested 等细节信息。简称为Requests-with-PASID。

每个平台可能会有一个或者多个IOMMU，如下图所示在x86服务器上通常会有多个DMAR Unit（IOMMU） 每个DMAR会对其下挂载设备的DMA请求进行地址翻译。如PCIe Root Port (dev:fun) (14:0)下面下游设备的DMA请求由DMAR #1负责处理，(14:1)下面设备的DMA请求由DMAR #2负责处理， 而DMAR #3下面是一个RC集成设备(29:0)，这个设备的DMA请求被DMAR #3承包， DMAR #4下面包含了所有平台上没有挂在其他DMAR下面的PCI兼容设备。 这些和IOMMU相关的硬件拓扑信息需要BIOS通过ACPI table汇报给OS，这样OS通过解析ACPI table，它就会知道那些PCI 设备被哪个DMAR管理。然后OS会为某一个特定的DMAR和下属的PCI设备建立翻译表。

![HPC](HPC.png)

VT-d spec定义了 5 张DMA Remapping Reporting (DMAR) ACPI table,对于DMA重映射，主要使用的是DRHD和RMRR这两张表格，这两张表中记录的设备的信息也是以B/D/F表示。

1. DRHD: DMA Remapping Hardware Unit Definition 描述DMAR Unit(IOMMU)的基本信息,主要提供VT-d重定向硬件寄存器基地址,以及该IOMMU所管辖的硬件。

	![DRHD](DRHD.png)

2. RMRR: Reserved Memory Region Reporting 描述保留的物理地址，该地址空间不被重映射。主要包含的是保留的内存的范围以及使用了这些保留内存的设备。

	![RMRR](RMRR.png)

3. ATSR: Root Port ATS Capability 仅限于有Device-TLB的情形，Root Port需要向OS报告支持ATS的能力

4. RHSA: Remapping Hardware Static Affinity Remapping亲和性，在有NUMA的系统下可以提升DMA Remapping的性能

5. ANDD: ACPI Name-space Device Declaration ANDD表用于表示一个以ACPI name-space规则命名，并且可发出DMA请求的设备。ANDD可以和前面提到的
Device Scope Entry结合在一起。

DMA 重映射需要使用一个ID来标识设备（Source-id），使用的是PCIe的Request ID来标识设备。根据PCIe的SPEC，每个PCIe设备的请求都包含了PCI Bus/Device/Function信息，通过BDF我们可以唯一确定一个PCIe设备。

![SRCID](SRCID.png)

为了能够记录设备和Domain的对应关系方便查找和映射地址，VT-d引入了root-entry和context-entry的概念。但是因为IOMMU支持两种形式的Memory Request，Requests-without-PASID和Requests-with-PASID所以这里也定义了两种方式用来映射地址，其实本质上就是两张表格。

**Requests-without-PASID地址映射实现方式：**

Requests-without-PASID地址映射会使用Root-table和Context-table两级表格来实现，Root-table的起始地址由IOMMU的Root Table Address Register 决定，VMM在enable IOMMU的时候需要将Root-table的地址写入到该寄存器中。该表格共计4KB，256个entry构成，每个entry对应一个bus，它 又指向一个4KB的256个entry构成的context-table，每个entry对应了一个Device /Function设备的地址转换页表。

![RTAR](RTAR.png)

![DDMS](DDMS.png)

对于Request-without-PASID，IOMMU硬件将会使用source-id也就是请求包中 B/D/F 作为索引值找到设备和Domain的对应关系的地址转换表，
然后将GPA转成HPA访问物理内存。

![BDFMAP](BDFMAP.png)

**Requests-with-PASID地址映射实现方式：**

这种类型的Memory Request和Request-without-PASID的区别是，请求中也包含B/D/F但是在TLP prefix中会有目标进程的地址空间标志符以及其他的flag信息。

![PASIDPREFIX](PASIDPREFIX.png)

另外请求中包含的地址类型是GVA而不是GPA，所以对于这种类型的请求，就需要GVA=>GPA=>HPA两级的转换，为了实现两级转换就需要用到Extended-root-table，这个table的每个entry中包含2个部分Upper-context-table和Lower-context-table。Upper-context-table里面的指针指向的是PASID的转换表，Lower-context-table指向的是Second-Level-Translation和Request-without-PASID一样的信息。转换的过程就是先用PASID去索引Upper-context-table中的PASID转换表得到GPA，然后拿得到的GPA作为Second-Level-Translation的输入去得到HPA。

![ERT](ERT.png)

Extended-root-table和Root-table的起始地址同样是由Root Table Address Register 决定，如果Bit11 Root Table Type设置成1就表示是Extended-root-table。

对于GPA到HPA的转换过程就和分页机制的转换过程完全一样，可以按照4KB/2MB/1G等方式实现分页，具体可以参考[X86-CPU的工作模式](https://peterhu.github.io/posts/2020/11/17/X86-CPU%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F.html)

![ATP](ATP.png)

参考:
1. 《vt-directed-io-spec》
2. 《A_Tour_Beyond_BIOS_Using_Intel_VT-d_for_DMA_Protection》
3. Intel VT-d（2）- DMA重定向： https://zhuanlan.zhihu.com/p/50750665
4. VT-d DMA Remapping： https://kernelgo.org/dma-remapping.html
5. 手撕intel-iommu之DMA remapping（硬件篇: https://nimisolo.github.io/post/vtd-dma-remapping/
