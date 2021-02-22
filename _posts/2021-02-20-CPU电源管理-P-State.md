---
layout: mypost
title: CPU电源管理-P-State
categories: [ACPI]
---

P-State指的是Performance State，它通过在C0的情况下通过调整电压和频率达到省电的目的。它是ACPI定义的电源管理的状态，其中P0是最高的电源和性能的状态,P7是最低的状态。操作系统可以单个core控制P-State。OSPM通过写PSTATE_CNT MSR 调整CPU的电压和工作频率。

BIOS 通过ACPI table将P-State汇报给OS。
_PCT (Performance Control), BIOS汇报register的形式为Function fixed Hardware PSTATE_CNT MSR。

_PSS (Performance Supported States)提供一个package list告知OS支持的P-State的数量，每个pstate的频率，电源，延迟，Bus Masters延迟，写到PSTATE_CNT的cmd，从PSTATE_STS读回的值。


```c
Package {
PState [0] // Package – Performance state 0
….
PState [n] // Package – Performance state n
}

Package {
CoreFrequency // Integer (DWORD)
Power // Integer (DWORD)
Latency // Integer (DWORD)
BusMasterLatency // Integer (DWORD)
Control // Integer (DWORD)
Status // Integer (DWORD)
}

```

![PST](PST.png)

_PSD (P-state Dependency)每个core都声明这个object，提供给OS相关的依赖信息。它的定义格式如下所示：

```c
Package {
NumEntries // Integer
Revision // Integer (BYTE)
Domain // Integer (DWORD)
CoordType // Integer (DWORD)
NumProcessors // Integer (DWORD)
}

```
NumEntries: 表示这个package里有多少个成员，当前的值为5。
Revision: 表示修订版，这里为0。
Domain: 通常跟是否开启了SMT有关，未开启SMT则为APICID[6:0]，开启了以后通常会是用APICID[6:1]
CoordType:表示有谁负责协调多个core的依赖关系，通常为FEh（HW_ALL）意思是硬件负责协调。
NumProcessors :表示有多少个处理器，开启SMT为2，否则为1。

![PSD](PSD.png)

参考： 
- [1. ACPI Spec](https://www.uefi.org/sites/default/files/resources/ACPI_6_2.pdf)

- [2. Cpu0Ist.asl](https://github.com/tianocore/edk2-platforms/blob/master/Silicon/Intel/Vlv2DeviceRefCodePkg/ValleyView2Soc/CPU/PowerManagement/AcpiTables/Ssdt/Cpu0Ist.asl)
