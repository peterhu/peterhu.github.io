---
layout: mypost
title: DDR Memory
categories: [DDR]
---

# **DDR 的定义：**
全称为Double Data Rate SDRAM，中文名为“双倍数据流SDRAM”。DDR SDRAM在原有的SDRAM的基础上改进而来。CLK与CLK#的交叉点都有数据传输因此称之为DDR。
	![DDR](DDR.png)
# **DRAM的存储原理：**
当行地址和列地址选通以后，存储电容就和外部的传输电路导通，从而可以进行放电（读取）与充电（写入）。
	![Theroy](Theroy.png)
**DDR与SDRAM最大的区别：**
Prefetch： 在SDRAM中，并没有这一技术，所以其每一个cell的存储容量等于DQ的宽度（芯片数据IO位宽）。进入DDR时代之后，就有了prefetch技术，DDR1是两位预取（2-bit Prefetch）有的公司则贴切的称之为2-n Prefetch（n代表芯片位宽）。DDR2是四位预取（4-bit Prefetch），DDR3和DDR4都是八位预取（8-bit Prefetch）。芯片位宽的另一种说法是配置模式（Configuration），在DDR3时代，一般有x4，x8，x16。下图是DDR3 X8的读取的一个例子，从这个图上我们可以看出芯片的数据位宽为8(DQ0-7), 又是8-bit Prefetch所以每个cell的容量是8*8=64bits。换一句话说，在指定bank、row地址和col地址之后，可以往该地址内写入（或读取）8 Bytes。
	![Prefetch](Prefetch.png)
DDR核心频率就是内存的工作频率；DDR1内存的核心频率是和时钟频率相同的，到了DDR2和DDR3时才有了时钟频率的概念，就是将核心频率通过倍频技术得到的一个频率。数据传输频率就是传输数据的频率。DDR2内存的时钟频率是核心频率的2倍。DDR3内存的时钟频率是核心频率的4倍 数据传输频率就是核心频率的8倍了 (通常数据传输频率是时钟/总线频率的2倍）DDR 后续还有 DDR2、DDR3、DDR4 的更新，基本上每一代都通过更多的 Prefetch 和更高的时钟频率，达到 2 倍于上一代的数据传输速率。 
	![Trate](Trate.png)
Transfer Rate (MT/s)： 为每秒发生的 Transfer 的数量，一般为 Bus Clock 的 2 倍 （一个 Clock 周期内，上升沿和下降沿各有一个 Transfer）
Internal rate (MHz)： 则是内部 Memory Array 读写的频率。由于 SDRAM 采用电容作为存储介质，由于工艺和物理特性的限制，电容充放电的时间难以进一步的缩短，所以内部 Memory Array 的读写频率也受到了限制，目前最高能到 266.67 MHz，这也是 SDR 到 DDR 采用 Prefetch 架构的主要原因。Memory Array 读写频率受到限制，那就只能在读写宽度上做优化，通过增加单次读写周期内操作的数据宽度，结合总线和 IO 频率的增加来提高整体传输速率。 
# **DDR Memory的架构：**
内存子系统从CPU 到memory芯片的逐层关系为CPU->channel＞DIMM＞rank＞chip＞bank＞row/column.从Memory controller出来先到Channel，每个Channel都要有一组控制寄存器用于配置和操作memory芯片。每个channel上能够拥有多组DIMM（Dual In-line Memory Module）,DIMM就是我们通常所说的内存条。
	![Arch1](Arch1.png)
	![Arch2](Arch2.png)	
Rank和Chip，Rank是指连接到同一个CS的chip，memory controller能够对同一个rank的chip进行读写操作，通常一组channel能够同时读写64bit的数据（ECC功能的是72bit）,所以对于8bit位宽的内存颗粒，8颗可以组成1个RANK。同样如果是16bit位宽则只需要4颗就可以。
	![Rank](Rank.png)
Chip再往下分就是Bank，Bank再往下就是一个个具体的存储数据的电路。在bank中行称之为row，列为colum。每组bank下方还有row buffer (sense amplifier)，负责将读出的行数据缓存，待列地址送达，输出正确的bit。以及判断储存的资料是0还是1。
	![Bank](Bank.png)
	![RowBuffer](RowBuffer.png)
Bank的读取和写入操作。首先memory controller会将地址和控制信号送到总线上。如果是多rank的dimm，CS也会送出相应的信号用于选通rank。Rank继续将信号送给Chip，每个Chip收到后通过解码器解析出对应bank的row/cloum地址。然后先enable row，同一行的数据就会被送到row buffer（sense amplifier）。Row buffer通过column decoder判断出对应row/col数据为0，1后就会送出。
	![Read](Read.png)
	![Write](Write.png)
	![Signal1](Signal1.png)
	![Signal2](Signal2.png)
**预充电(Precharge)：** SRAM的寻址具有独占性，所以每次切换到同一个bank的不同的行时就要将原来的工作行关闭，重新发送行和列地址。L-BANK关闭现有行打开新的行的过程被称作为了预充电。实际上，预充电是一种对工作行中所有存储体进行数据重写，并对行地址进行复位，同时释放S-AMP，以准备新行的工作。地址线A10控制着是否进行在读写之后对当前的L-BANK自动预充电。通常在发出预充电之后还需要一段时间才允许发送RAS#打开新的工作行，这个时间被称为tRP(precharge command period,预充电有效周期)。单位是时钟周期数。
**内存刷新电路(Refresh)：** 之所以叫做DRAM,就是因为它需要不断的刷新才能保持住数据。刷新其实就是对数据进行重写，存储体中电容保持数据的时间上线是64ms，所以每一行的刷新周期就是64ms。刷新分两种，自动刷新(Auto refresh)和自刷新(Self refresh)。自刷新用在S3的状态下，根据内部时钟进行刷新。刷新操作与预充电中重写的操作一样，都是用S-AMP先读再写。但为什么有预充电操作还要进行刷新呢？因为预充电是对一个或所有L-Bank中的工作行操作，并且是不定期的，而刷新则是有固定的周期，依次对所有行进行操作，以保留那些久久没经历重写的存储体中的数据。但与所有L-Bank预充电不同的是，这里的行是指所有L-Bank中地址相同的行，而预充电中各L-Bank中的工作行地址并不是一定是相同的。
信号放大器：用于将外部电路的变化转化成0/1存入电容，或者导出。
**延迟锁定回路（DLL）:** DDR SDRAM对时钟的精确性有着很高的要求，而DDR SDRAM有两个时钟，一个是外部的总线时钟，一个是内部的工作时钟，在理论上DDR SDRAM这两个时钟应该是同步的，但由于种种原因，如温度、电压波动而产生延迟使两者很难同步，所以需要根据外部时钟动态修正内部时钟的延迟来实现与外部时
钟的同步，这就是DLL的任务。
**CK/CK#：** 差分时钟DDR的一个必要设计。CK#的作用通常被理解为第二个触发时钟，但其实它起到了触发时钟校准的作用。由于各种因素的音响CK上下沿间距可能发生变化，CK#就可以起到纠偏的作用。CK上升快下降慢，CK# 则是上升慢下降快
**DQS：**数据选取脉冲，双向信号；读取内存时，由内存触发，DQS的沿和数据的沿对齐。写入时由CPU/MEM controller触发，DQS的中间对应数据的沿。
**DQ0-DQn**：数据输入输出信号。
**RAS#,CAS#,WE#：**行选通，列选通，写使能信号。
**CS#：**片选信号，使能命令解码器。
**A0-An：**行列共用地址线，其中A10在读写命令期间用作自动预充电。
**BA0-BA1：**Bank 选通信号。
**CKE：**时钟使能信号。
**DM0-DMi:** 数据掩码
# **DDR的Command：**
Host 与 SDRAM 之间的交互都是由 Host 以 Command 的形式发起的。一个 Command 由多个信号组合而成，下面表格中描述了主要的 Command。
	![CmdTable](CmdTable.png)
- Active 
Active Command 会通过 BA[1:0] 和 A[12:0] 信号，选中指定 Bank 中的一个 Row，并打开该 Row 的 wordline。在进行 Read 或者 Write 前，都需要先执行 Active Command。 
- Read 
Read Command 将通过 A[12:0] 信号，发送需要读取的 Column 的地址给 SDRAM。然后 SDRAM 再将 Active Command 所选中的 Row 中，将对应 Column 的数据通过 DQ[15:0] 发送给 Host。 
Host 端发送 Read Command，到 SDRAM 将数据发送到总线上的需要的时钟周期个数定义为 CL。 
- Write 
Write Command 将通过 A[12:0] 信号，发送需要写入的 Column 的地址给 SDRAM，同时通过 DQ[15:0] 将待写入的数据发送给 SDRAM。然后 SDRAM 将数据写入到 Actived Row 的指定 Column 中。SDRAM 接收到最后一个数据到完成数据写入到 Memory 的时间定义为 tWR （Write Recovery）。 
- Precharge 
在进行下一次的Read或者Write操作前必须要先执行 Precharge操作。Precharge 操作是以 Bank 为单位进行的，可以单独对某一个 Bank 进行，也可以一次对所有 Bank 进行。如果 A10 为高，那么 SDRAM 进行 All Bank Precharge 操作，如果 A10 为低，那么 SDRAM 根据 BA[1:0] 的值，对指定的 Bank 进行 Precharge 操作。 SDRAM 完成 Precharge 操作需要的时间定义为 tPR。 
- Auto-Refresh 
DRAM 的 Storage Cell 中的电荷会随着时间慢慢减少，为了保证其存储的信息不丢失，需要周期性的对其进行刷新操作。 SDRAM 的刷新是按 Row 进行，标准中定义了在一个刷新周期内（常温下 64ms，高温下 32ms）需要完成一次所有 Row 的刷新操作。 为了简化 SDRAM Controller 的设计，SDRAM 标准定义了 Auto-Refresh 机制，该机制要求 SDRAM Controller 在一个刷新周期内，发送 8192 个 Auto-Refresh Command，即 AR， 给 SDRAM。 SDRAM 每收到一个 AR，就进行 n 个 Row 的刷新操作，其中，n = 总的 Row 数量 / 8192 。此外，SDRAM 内部维护一个刷新计数器，每完成一次刷新操作，就将计数器更新为下一次需要进行刷新操作的 Row。 一般情况下，SDRAM Controller 会周期性的发送 AR，每两个 AR 直接的时间间隔定义为 tREFI = 64ms / 8192 = 7.8 us。 SDRAM 完成一次刷新操作所需要的时间定义为 tRFC, 这个时间会随着 SDRAM Row 的数量的增加而变大。 由于 AR 会占用总线，阻塞正常的数据请求，同时 SDRAM 在执行 refresh 操作是很费电，所以在 SDRAM 的标准中，还提供了一些优化的措施，例如 DRAM Controller 可以最多延时 8 个 tREFI 后，再一起把 8 个 AR 同时发出。 
- Self-Refresh 
Host 还可以让 SDRAM 进入 Self-Refresh 模式，降低功耗。在该模式下，Host 不能对 SDRAM 进行读写操作，SDRAM 内部自行进行刷新操作保证数据的完整。通常在设备进入待机状态时，Host 会让 SDRAM 进入 Self-Refresh 模式，以节省功耗。 
# **DDR的读写时序：# **
在读写 memor之前要先通过CS#选定相应的PBANK(RANK), 接下来通过BA0,BA1选通LBANK，接下来就是通过行有效(RAS#)和列有效(CAS#)选通具体的行和列，然后就可以进行读写了。但是因为行地址和列地址都是共用地址线，所以在RAS#切换到CAS#之间一定有一个间隔用于保证芯片存储阵列电子元件响应时间，这个时间叫做为tRCD，即RAS to CAS Delay（RAS至CAS延迟）单位是时钟周期。
	![tRCD](tRCD.png)
在列地址确定了以后，只需要将数据通过DQ送给总线上即可。但是从CAS#到真正的数据输出到总线其实还需要一段时间，被称为CL/RL（CAS Latency，CAS潜伏期）。为什么需要这个时间呢，主要是因为存储单元的需要一定的反应时间，所以不可能和CAS在同一个上升沿触发。另外电容容量很小，信号需要放大（S-AMP)识别以后才能送到总线上，这也需要一段时间被称为tAC （Access Time from CLK，时钟触发后的访问时间）。tAC的单位是ns。
	![tAC](tAC.png)
数据写入时也是在tRCD之后进行，但是并不需要CL. 虽然数据可以和CAS#同时送出，但是因为选通三极管与电容的充电必须要要有一段时间，所以真正的数据写入需要一定的周期，都会留出足够的写入/校正时间（tWR，Write Recovery Time），这个操作也被称作写回（Write Back）。tWR至少占用一个时钟周期或再多一点。
	![tWB](tWB.png)
# **Burst Mode:# **
是指同一行的相邻的存储单元连续进行传输的方式，连续传输的数量称之为突发长度 BL(Burst Length)。在未使用Burst的情况下，连续读写多个数据会导致内存控制资源被占用（要一直发读取cmd和列地址），在数据传送期间无法输入新的命令。
	![BL1](BL1.png)
使用了突发传输模式以后只要指定列起始地址和BL，内存就会自动依次读取后续相应数量的存储单元而且并不需要一直提供列地址。BL 有 1，2，4，8。
	![BL2](BL2.png)
# **Memory 地址映射:# **
SDRAM Controller 的主要功能之一是将 CPU 对指定物理地址的内存访问操作，转换为 SDRAM 读写时序，完成数据的传输。在实际的产品中，通常需要考虑 CPU 中的物理地址到 SDRAM 的 Bank、Row 和 Column 地址映射。下图是一个 32 位物理地址映射的一个例子：
	![AddMap](AddMap.png)
# **DDR2 VS DDR3 VS DDR4：# **
DDR从发明之初直到DDR4并没有太大的变化，都是通过调整prefetch bits 不断的增加传输速度，核心频率一直保持在100-266MHZ之间，prefetch最大8n。但是到了DDR4以后增加prefetch的方法玩不下去了，因为cache line最大只有64Bytes，8bit prefetch时一条cahe line 需要BL8就可以满足了，如果是16 bit prefetch一次取128bits BL8就会是128 Bytes，其中的64Bytes被浪费了，所以DDR4开始提升核心频率到200-400MHZ了。
	![DDRVS](DDRVS.png)
另外DDR4 还引入了Bank Group的机制用于提升性能。具体来说就是每个Bank Group可以独立读写数据，这样一来内部的数据吞吐量大幅度提升，可以同时读取大量的数据，内存的等效频率在这种设置下也得到巨大的提升。DDR4架构上采用了8n预取的Bank Group分组，包括使用两个或者四个可选择的Bank Group分组，这将使得DDR4内存的每个Bank Group分组都有独立的激活、读取、写入和刷新操作。类似于多路传输，在一个工作时钟周期内可以最多同时处理4组数据，从而改进内存的整体效率和带宽。
	![DDR4BG](DDR4BG.png)
# **Memory Training :# **
从CPU 角度来看每个在DIMM 上的DRAM距离CPU的距离是不同的。从DIMM本身来看 DRAM Chip的CLK 和 Data信号是不等长的（有偏差）所以需要Training. Training是用来补偿来自板子和DRAM的延时。

**DQS Receiver Enable：**
我们在读取数据的时候要先发送Read cmd, 然后再等一段时间数据才会出现在总线上 然后Memory controller需要 enable DQS receiver pad 去接收数据。DRAM通过DQS信号通知有效的数据送出了，再此之前DRAM会先拉低半个周期的DQS信号叫做"read preamble",这个read preamble 就是用来让CPU知道下个DQS信号到来时会有有效的数据。但是在preamble 到来之前DQS容易产生干扰信号，那么就会让receiver pad误判。DQS Receiver Enable training 就是通过调整delay时间让pad刚好在preamble信号的中间的时候打开。
	![DQSReEn](DQSReEn.png)
对于每个Channel上存在rank，选定两个地址（64bytes cache line对齐，相距2M）,CPU 向其写入特定的类型（55,AA）的64bytes数据。CPU会使用第一个QWORD的数据来train Receiver PAD。程序会不断的调整delay，然后通过从这两个地址读取数据并和已知的数据比较。一旦可以获得正确的数据就意味着 DQS Receiver刚好在preamble的左边。我们就可以把这个delay保存并写入相关的寄存器。
Q:如何保证test pattern 可以成功地写入内存？SPEC上说写入一个cache line 特定的pattern，可以保证第一个QWord可以被正确的写入DRAM。

**Write Leveling:**
从DDR3开始引入了fly-by布线，指地址、命令和时钟的布线依次经过每一颗DDR memory芯片,但是DQ,DQS仍然是点对点的连接。有助于降低同步切换噪声。但是这种布线就造成了CLK和DQ/DQS信号的偏移。
	![WL](WL.png)
Write Leveling 的功能是调整DRAM颗粒端DQS信号和CLK信号边沿对齐；Training 的具体过程如下：通过将MR1寄存器A7设置为1进入Write Leveling模式，然后DDR controller 不断的调整DQS相对CLK的延迟，DRAM芯片会在DQS的上升沿采样CLK管脚上的时钟信号，如果采样值为低则通过将所有的DQ[n]保持低电平通知DDR controller tDQSS相位关系还未满足。如果发现在某一个DQS上升沿，采样到的CLK电平变为了高，则认为刺史tDQSS相位关系已经满足要求，则通过DQ[n]拉高通知DDR controller通知一个Write Leveling 成功。同时DDR controller 会锁住这个相位差。这时从DRAM端看到的CLK和DQS都是边沿对齐的。
	![WLTiming](WLTiming.png)
参见上图，写入均衡的修调过程：
t1：将ODT拉起，使能on die termination；
t2：等待tWLDQSEN时间后（保证DQS管脚上的ODT已设置好），DDR控制器将DQS置起；DDR memory在DQS上升沿采样CK信号，发现CK=0，则DQ保持为0。
t3：DDR控制器将DQS置起；DDR memory在DQS上升沿采样CK信号，发现CK=0，则DQ仍然保持为0。
t4：DDR控制器将DQS置起；DDR memory在DQS上升沿采样CK信号，发现CK=1，则等待一段时间后，DDR memory将dq信号置起。

采取以上策略的原因：对于DDR controller来说，其无法测定clk边沿和dqs边沿的绝对位置，故采用了不断调整dqs delay，在dqs上升沿判断clk从0到1或1到0的一个变化，一旦检测到变化，则写入均衡停止。
tDQSS（DQS, DQS# rising edge to CK, CK# rising edge，在标准中要求为+/-0.25 tCK。tCK为CLK时钟周期）

布线要求：1. 只有使用了fly-by的情况下需使能write leveling 2.CPU内部的内存控制器只能对DQS信号做延迟，不能做超前处理，所以CK要大于DQS信号线的长度，否则将不能满足tDQSS。

**DQS Positioning:**
DQS positioning 分为 Read DQS Timing和 Write Data Timing两种，他们的目的是用来保证DQS的信号要在DQ data eye的中间。
- Read DQS Timing: 通过在读数据的时候， CPU/Memory controller 通过调整内部的DLL延迟锁定回路电路，延迟DQS 使得它在DQ data eye的中间。具体的training过程是：
	- 对于Memory的每个channel的每个rank写一个cacheline的特定类型的数据。将ReadDQSTiming 和Write Data Timing 都初始化为0x00。
		- 然后读回数据并标记基于现在的ReadDQS timing的情况 PASS或者Fail。
		- 增加ReadDQSTiming delay,继续步骤a.直到找到最大可以pass的ReadDQSTiming delay。
	- 增加Write Data Timing delay,继续步骤1。当且仅当出现连续3组pass的情况，取中间的一组数据并记录ReadDqsTiming的平均值，如：这里的ReadDQSTiming delay = (0x00 + 0x24)/2 = 0x12.
		- Write Delay = x05h -> No passes 
		- Write Delay = x06h -> Read Delays of x11h -> x14h pass 
		- Write Delay = x07h -> No passes 
		- Write Delay = x08h -> Read Delays of (x01h -> x23h) pass 
		- Write Delay = x09h -> Read Delays of (x00h -> x24h) pass 
		- Write Delay = x0Ah -> Read Delays of (x00h -> x24h) pass 

- Write Data Timing: 在写数据的时候，ReadDqsTiming已经找到了。另外因为Memory没有DLL电路，所以只能通过调整CPU/Memory controller端的Write Data timing去配合DQS 信号使得DQS在DQ data eye的中间。具体的training过程如下：
	- 对于Memory的每个channel的每个rank写一个cacheline的特定类型的数据。
		- 然后读回数据并记录基于现在的Write Data timing的情况 PASS或者Fail。
		- 增加Write Data Timing Delay,继续步骤a.直到找到最大可以pass的Write DQ Delay timing。
		例如：
		Read DQS Delay = x12h -> Write DQ Delays of （x07h -> x23h） pass
	- 计算出他们的中点（平均值）并且设置相应的Write DQ Delay timing的值。这里的平均值是（0x7+0x23) / 2 = 0x15.
	![DQS](DQS.png)
	
**Max Read Latency：**
它是用来告诉CPU/Memory Controler 什么时候可以读到来自于DRAM的数据。
- 它会基于一些固定的内部延迟以及物理DDR的配置来计算。
- 挑选一个ReadEnableDelay最大的DIMM。
- 向该dimm写3个cacheline的数据。
- 调整MaxRdLatency直到读测试PASS.

Refer: 
- [1. DDR工作原理](https://www.cnblogs.com/shengansong/archive/2012/09/01/2666213.html) 
- [2. Write leveling（写入均衡）](https://blog.csdn.net/tbzj_2000/article/details/88304245) 
- [3. DDR扫盲——关于Prefetch与Burst的深入讨论 ](http://blog.chinaaet.com/justlxy/p/5100052027)
- [4. DDR3 vs DDR4](https://zhuanlan.zhihu.com/p/62234511)
- [5. DDR4 記憶體](https://www.techbang.com/posts/18097-ddr4-memory-import-pc-platforms-why-deny-ddr3-ddr4)
- [6. 圖解RAM結構與原理，系統記憶體的Channel Chip與Bank](https://www.techbang.com/posts/18381-from-the-channel-to-address-computer-main-memory-structures-to-understand)
- [7. DDR4 SDRAM - Understanding the Basics](https://www.systemverilog.io/ddr4-basics)
- [8. DDR training receive enable calibration](https://www.weibo.com/p/1001603802789183630161?pids=Pl_Official_CardMixFeed__5&feed_filter=1)
- [9. Understanding DDR Memory Training](https://github.com/librecore-org/librecore/wiki/Understanding-DDR-Memory-Training)