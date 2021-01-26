---
layout: mypost
title: PCIe SR-IOV
categories: [PCIe]
---

SR-IOV (Single Root Input/Output Virtualization) 的主要作用是将一个物理设备模拟成多个虚拟设备，其中每一个虚拟设备可以和一个虚拟机绑定，从而便于虚拟机访问同一个物理设备。在引入SR-IOV技术之前，一个PCIe设备在指定的时间段内，只能与一个虚拟机A绑定，当其他虚拟机B要访问该PCIe设备时，需要向虚拟机A发送请求，由虚拟机A获得数据以后再返回给虚拟机 B，这种情况下将会增大虚拟机访问设备的延时，同时也干扰了其它虚拟机的正常运行。而如果在一个系统中增加多个同样的PCIe设备，虽然可以解决这个问题但是增加了成本，复杂度，造成了不必要的浪费。

SR-IOV引入了两种新的PCIe的Function：
1. PFs：完整功能的PCIe Function包含了SR-IOV Extended Capability。这个Capability被用来配置和管理SR-IOV的功能。
2. VFs：一部分轻量级的PCIe function，只包含必要的用于数据移动的最小可配置的资源。

一个 具备SR-IOV能力的设备可以被配置成被枚举出多个Virtual Functions（数量可控制），而且每个Funciton 都有自己的完整有BAR的配置空间。VMM可以把一个或者多个VF 分配给VM。

![SR-IOV](SR-IOV.png)

SR-IOV的初始化过程如下：
1. BIOS 在PCI设备枚举阶段会先去检测设备是否支持SR-IOV Extended Capability。
2. 如果支持BIOS可以通过SR-IOV Controlde 寄存器去enable VF。
3. BIOS program System Page Size （System Page Size = 1，则表示页大小为4KB。表示该PF的所有VF的bar必须以System Page Size对齐）。
4. BIOS program VF BAR (PF的bar寄存器一样：通过写全1，然后读回来确定bar空间的大小。BAR0表示该PF的所有VF的bar0的基地址。BAR0映射的空间大小为: 一个VF的bar0的大小 * NumVFs。Vf的每个bar大小是一样的，另外VF不支持I/O space）

![SR-IOVCAP](SR-IOVCAP.png)

参考：

《PCI-SIG SR-IOV Primer》

《Single Root I/O Virtualization and Sharing Specification Revision 1.1》
