---
layout: mypost
title: CPU电源管理-C-state
categories: [ACPI]
---

C-state 是 Processor(CPU) State 的简写。 C-state 是ACPI spec定义的CPU工作在G0时的power states，这些状态包括C0,C1,C2,C3…Cn。C0表示CPU在执行指令的状态，除了C0之外的其他State都是低功耗的状态，CPU不会执行指令。也因而会节省更多的功耗。系统在运行时会根据loading状况在各个C-state之间切换 降低功耗，图1是C-state切换的一个简单的当CPU在进出sleeping state时会有一定的延时，通常延迟越大功耗对应的C-state的功耗就越低。APCI规定C0 C1 C2需要保持cache的一致性（要保证CPU cache中的数据一定要是最新的数据），C3以及后续的state就没有这个要求了，也就是说如果系统还要memory request，OS就不会进入C3以及之后的状态。

![C-state](C-state.png)

ACPI定义的C State是按照数字递增的如C0,C1,C2,C3，他们和CPU vendor定义的并不完全一致，CPU厂商通常会定义C0, C1E, C6, C7。所以在通过ACPI method _CST report 时需要做一个映射。
BIOS 通过ACPI Table report 给OS当前支持的 C-State和实现的方式:
1.  _OSC & _PDC _0SC(Operating System Details) & _PDC(Processor Driver Capabilities)在功能上比较接近， 基本上供OSPM调用和BIOS传递一些关于C-state P-state T-state是否支持，以及支持的程度和实现方式的一些设定，BIOS可以依据OSPM的参数回报相应的ACPI Structures。
2. ACPI定义多种C-state的控制接口，FADT P_BLK P_LVL2,P_LVL3, P_LVLx_LAT,以及_CST 。_CST会override FADT的一些设置。其中Register 表示OSPM调整C-State的方式；Type表示C State的类型（1=C1, 2=C2, 3=C3)；Latency表示进入该C-state的最大的延迟；Power表示在该C-state时的功耗（单位是毫瓦）。

	```c
	Package {
	Count // Integer
	CStates[0] // Package
	….
	CStates[Count-1] // Package
	}
	
	Package {
	Register // Buffer (Resource Descriptor)
	Type // Integer (BYTE)
	Latency // Integer (WORD)
	Power // Integer (DWORD)
	}
	```	
	进入C-state通常有通过1. 读IO的方式 2.通过mwait的方式 3.指令HLT(C1)。如下述的参考代码所示，该CPU支持2个C-state,其中C1使用FFixedHW的方式访问，C2都是通过读取IO地址0x414的方式进入C-state。

	```c
	Name(_CST, Package() 
	{
	2,      // There are four C-states defined here with three semantics
	        // The third and fourth C-states defined have the same C3 entry semantics 
	Package(4){Register(FFixedHW, 0x01, 0x02, 0x0000000000000000,0x01)),     0x01,  0x03, 0x000003e8}}), 
	Package(4){Register(SystemIO, 0x08, 0x00, 0x0000000000000414, 0x01,)), 0x02, 0x0190, 0x00000000}})
	})
	```	
	
	FFixedHW看着挺奇怪的，它的全称是Functional Fixed Hardware (FFH),ASL code中的标识符是0x7F,当OSPM看到FFH的标识符时，它会使用MWAIT指令进入C-state，下图中的Arg0会放到EAX中作为参数传递给是MWAIT指令。MWAIT的实现相对于IO可能会更轻量和高效。

	![FFH](FFH.png)

3. _CSD C-state Dependency 用于向OSPM提供多个 logic processor之间C-state的依赖关系。比如在一个Dual Core的平台上，每颗核可以独立运行C1但是如果其中一个核切换到C2，另一个也必须要切换到C2，这时就需要在_CSD中提供这部分信息。其中NumEntries 表示_CSD这个package里面一共有多少项；Revision 表示版本号，目前都为0；Domain表示和这个CPU的C-State有依赖关系的CPU所属的域，通常是一个physical core上的两个logic core（thread）共属一个域。CoordType 表示是由OSPM负责协调有依赖关系的CPU的C-state的进出还是由硬件负责，0xFC (SW_ALL), 0xFD (SW_ANY) or 0xFE (HW_ALL)，这里通常都是设置为HW_ALL。

	```c	
	Package {
	NumEntries // Integer
	Revision // Integer (BYTE)
	Domain // Integer (DWORD)
	CoordType // Integer (DWORD)
	NumProcessors // Integer (DWORD)
	Index // Integer (DWORD)
	}
	
	Name(_CSD, Package(1)
	{
	Package(6) {0x06, 0x00, 0x00000000, 0x000000FE, 0x00000002, 0x00000000}
	})
	```

参考： 
	
	- [1. ACPI Spec](https://www.uefi.org/sites/default/files/resources/ACPI_6_2.pdf)

	- [2. Intel® Processor Vendor-Specific ACPI](https://www.intel.com/content/dam/www/public/us/en/documents/product-specifications/processor-vendor-specific-acpi-specification.pdf)

	- [3. Cpu0Cst.asl](https://github.com/tianocore/edk2-platforms/blob/master/Silicon/Intel/Vlv2DeviceRefCodePkg/ValleyView2Soc/CPU/PowerManagement/AcpiTables/Ssdt/Cpu0Cst.asl)

	- [4. CPU省电的秘密（二）：CStates](https://zhuanlan.zhihu.com/p/25675639)