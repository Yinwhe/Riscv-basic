# MODE/CSR相关

[TOC]

# Mode介绍

- RISC-V有三个特权模式：U（user）模式、S（supervisor）模式和M（machine）模式。
- 它通过设置不同的特权级别模式来管理系统资源的使⽤。
    - 其中M模式是最高级别，该模式下的操作被认为是安全可信的，主要为对硬件的操作；
    - U模式是最低级别，该模式主要执⾏⽤户程序，操作系统中对应于⽤户态；
    - S模式介于M模式和U模式之间，操作系统中对应于内核态，当⽤户需要内核资源时，向内核申请，并切换到内核态进⾏处理。

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/058ae3ad-47d4-42e3-8c75-244cc4fc9d49/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043420Z&X-Amz-Expires=86400&X-Amz-Signature=f332393b3ff1a6e14ea7d52648ebf6bd88ce96d4aae6149000f1935f2ccdc586&X-Amz-SignedHeaders=host)

## 机器模式

- 机器模式（缩写为 M 模式，M-mode）是 RISC-V 中 hart（hardware thread，硬件线程）可以执行的最高权限模式。在 M 模式下运行的 hart 对内存，I/O 和一些对于启动和配置系统来说必要的底层功能有着完全的使用权。
    
    - Hart：即硬件线程(hardware thread)。hart与软件线程不同，软件线程在 harts 上进行分时复用，大多数处理器核都只有一个hart。
- 机器模式最重要的特性是拦截和处理异常（不寻常的运行时事件）的能力。
- 从广义上来讲，中断和异常都被认为是一种广义上的异常。
    
    - 处理器广义上的异常，通常只分为**同步异常**（Synchronous Exception）和**异步异常**（Asynchronous Exception）。
- RISC-V 中实现**精确异常**：保证异常之前的所有指令都完整地执行了，而后续的指令都没有开始执行（或等同于没有执行）。
- 在 M 模式运行期间可能发生的**同步异常**有五种：
    - 访问错误异常 - 当物理内存的地址不支持访问类型时发生（例如尝试写入 ROM）。
    
    - 断点异常 - 在执行 ebreak 指令，或者地址或数据与调试触发器匹配时发生。
    
    - 环境调用异常 - 在执行 ecall 指令时发生。
    
    - 非法指令异常 - 在译码阶段发现无效操作码时发生。
    
    - 非对齐地址异常 - 在有效地址不能被访问大小整除时发生，例如地址为 0x12 的amoadd.w。
    
      
- **三种标准的中断源**：软件、时钟和外部来源。
    
    - **软件中断：**通过向内存映射寄存器中存数来触发，并通常用于由一个 hart 中断另一个 hart（在其他架构中称为处理器间中断机制）。当
    - **时钟中断：**当实时计数器 mtime 大于 hart 的时间比较器（一个名为 mtimecmp 的内存映射寄存器）时，会触发时钟中断。
    - **外部中断：**由平台级中断控制器（大多数外部设备连接到这个中断控制器）引发。不同的硬件平台具有不同的内存映射并且需要中断控制器的不同特性，因此用于发出和消除这些中断的机制因平台而异。



### 机器模式下的异常处理

- 八个控制状态寄存器（CSR）是机器模式下异常处理的必要部分：
    - mtvec（Machine Trap Vector）它保存发生异常时处理器需要跳转到的地址。
    - mepc（Machine Exception PC）它指向发生异常的指令。
    - mcause（Machine Exception Cause）它指示发生异常的种类。
    - mie（Machine Interrupt Enable）它指出处理器目前能处理和必须忽略的中断。
    - mip（Machine Interrupt Pending）它列出目前正准备处理的中断。
    - mtval（Machine Trap Value）它保存了陷入（trap）的附加信息：地址异常中出错的地址、发生非法指令异常的指令本身，对于其他异常，它的值为 0。
    - mscratch（Machine Scratch）它暂时存放一个字大小的数据。
        - 为避免覆盖整数寄存器中的内容，中断处理程序先在最开始用 mscratch 和整数寄存器（例如 a0）中的值交换。
        - 通常，软件会让 mscratch 包含指向附加临时内存空间的指针，处理程序用该指针来保存其主体中将会用到的整数寄存器。
    - mstatus（Machine Status）它保存全局中断使能，以及许多其他的状态。
- 当一个 hart 发生异常时，硬件自动经历如下的状态转换：
    - 异常指令的 PC 被保存在 mepc 中，**PC** **被设置为** **mtvec** (对于同步异常，mepc指向导致异常的指令；对于中断，它指向中断处理后应该恢复执行的位置)
    - 根据异常来源设置 mcause，并将 mtval 设置为出错的地址或者其它适用于特定异常的信息字。
    - 把控制状态寄存器 mstatus 中的 MIE 位置零以禁用中断，并把先前的 MIE 值保留到 MPIE 中。
    - 发生异常之前的权限模式保留在 mstatus 的 MPP 域中，再把权限模式更改为M (如果处理器仅实现 M 模式，则有效地跳过这个步骤)
- 返回时：处理程序用 mret 指令（M 模式特有的指令）返回。mret 将 PC 设置为 mepc，通过将 mstatus 的 MPIE 域复制到MIE 来恢复之前的中断使能设置，并将权限模式设置为 mstatus 的 MPP 域中的值。

> RISC-V 还支持向量中断，其中处理器跳转到各类中断各自对应的地址，而不是一个统一的入口点。这种寻址消除了读取和解码 mcause的需要，加快了中断处理速度。将 mtvec [0]设置为1可启用此功能;然后根据中断原因 x 将PC 设置为（mtval-1 +4x ）， 而不是通常的mtval。



### 进入U模式

通过将 mstatus.MPP 设置为U(0)，然后执行 mret 指令，软件可以从 M 模式进入 U 模式。如果在 U 模式下发生异常，则把控制移交给 M 模式。



**嵌入式系统中的用户模式与进程隔离**(Nonfinished)



## 监管者模式

- 与 U 模式一样，S 模式下运行的软件不能使用 M 模式的 CSR 和指令，并且受到 PMP 的限制。
- 默认情况下，发生任何异常（不论在什么权限模式下）的时候，控制权都会被移交到M 模式的异常处理程序。
- **异常委托：**M 模式的异常处理程序可以将异常**重新导向** S 模式，这些额外的操作会减慢大多数异常的处理速度。因此，RISC-V 提供了一种异常委托机制。通过该机制可以选择性地将中断和同步异常交给 S 模式处理，而完全绕过 M 模式。
- **注意，**无论委派设置是怎样的，发生异常时控制权都不会移交给权限更低的模式。在 M 模式下发生的异常总是在 M 模式下处理。在 S 模式下发生的异常，根据具体的委派设置，可能由 M 模式或 S 模式处理，但永远不会由 U 模式处理。

### S模式几个异常处理CSR：

- sepc、stvec、scause、sscratch、stval 和 sstatus，它们执行与 M 模式 CSR 相同的功能。

- 监管者异常返回指令 sret 与 mret 的行为相同，但它作用于 S 模式的异常处理 CSR，而不是 M 模式的 CSR。
- S 模式处理例外的行为和 M 模式非常相似。如果 hart 接受了异常并且把它委派给了S 模式，则硬件会经历几个类似的状态转换，其中用到了 S 模式而不是 M 模式的CSR：
    - 发生例外的指令的 PC 被存入 sepc，且 PC 被设置为 stvec。
    - scause 根据异常类型设置，stval 被设置成出错的地址或者其它特定异常的信息字。
    - 把 sstatus CSR 中的 SIE 置零，屏蔽中断，且 SIE 之前的值被保存在 SPIE 中。
    - 发生例外时的权限模式被保存在 sstatus 的 SPP 域，然后设置当前模式为 S 模式。

### 委托

- **RISC-V**架构所有**mode**的异常在默认情况下都跳转到**Machine mode**处理。为了提高性能，RISC-V支持将低权限mode产生的异常委托给对应mode处理，而完全绕过Mmode，这里使用了mideleg和medeleg两个寄存器：
    - **mideleg**（Machine Interrupt Delegation）：控制将哪些异常委托给对应mode处理，它的结构可以参考mip寄存器，比如将mip中stip对应的位置位会将Supervisor mode的时钟中断委托给Supervisor mode处理
    - **medeleg**（Machine Exception Delegation）：控制将哪些异常委托给对应mode处理，它的各个位对应mcause寄存器的返回值
- 委托给 S 模式的任何中断都可以被 S 模式的软件屏蔽：
    - **sie**（Supervisor InterruptEnable，监管者中断使能）和 **sip**（Supervisor Interrupt Pending，监管者中断待处理）CSR是 S 模式的控制状态寄存器，他们是 mie 和 mip 的子集。它们有着和 M 模式下相同的布局，但在 sie 和 sip 中只有与由 mideleg 委托的中断对应的位才能读写。那些没有被委派的中断对应的位始终为零。
- M 模式还可以通过 **medeleg** CSR 将同步异常委托给 S 模式。该机制类似于中断委托，但 medeleg 中的位对应的不再是中断，而是图同步异常编码，详见mcause。

### Supervisor Mode下时钟中断处理流程

- 事实上，虽然在mideleg中设置了将Supervisor mode产生的时钟中断委托给Supervisor mode，委托并没有完成。因为硬件产生的时钟中断仍会发到Machine mode（mtime寄存器是Machine mode的设备），所以需要手动触发Supervisor mode下的时钟中断。
- 此前，假设设置好[m|s]status以及[m|s]ie，即已经满足了时钟中断在两种mode下触发的使能条件。接下来一个时钟中断的委托流程如下：
    1. 当mtimecmp小于mtime时，触发时钟中断并且硬件自动置位mip[mtip]。
    2. 此时mstatus[mie]=1，mie[mtie]=1，且mip[mtip]=1 表示可以处理machine mode的时钟中断。
    3. 此时hart发生了异常，硬件会自动经历状态转换，其中pc被设置被mtvec，即跳转到我们设置好的machine mode处理函数入口。
    4. machine mode处理函数分析异常原因，判断为时钟中断，为了将时钟中断委托给supervisor mode，于是将mip[stip]置位，并且为了防止在supervisor mode处理时钟中断时继续触发machine mode时钟中断，于是同时将mie[mtie]清零。
    5. machine mode处理函数处理完成并退出，此时sstatus[sie]=1，sie[stie]=1，且sip[stip]=1(由于sip是mip的子集，所以第4步中令mip[stip]置位等同于将sip[stip]置位)，于是触发supervisor mode的时钟中断。
    6. 此时hart发生了异常，硬件自动经历状态转换，其中pc被设置为stvec，即跳转到我们设置好的supervisor mode处理函数入口。
    7. supervisor mode处理函数分析异常原因，判断为时钟中断，于是进行相应的操作，然后利用ecall触发异常，跳转到machine mode的异常处理函数进行最后的收尾。
    8. machine mode异常处理函数分析异常原因，发现为ecall from S-mode，于是设置mtimecmp+=100000，完成后将mip[stip]清零，表示Smode处理完毕，并且设置mie[mtie]恢复machine mode的中断使能，保证下一次时钟中断可以触发。
    9. 函数逐级返回，整个委托的时钟中断处理完毕。

## 一些寄存器

---

### mstatus寄存器

- mstatus寄存器，即Machine Status Register，此寄存器中保持跟踪以及控制hart(hardware thread)的运算状态。
    - ⽐如mie和mip都对应mstatus上的某些bit位，所以通过对mstatus进⾏⼀下按位运算，可以实现对不同bit位的设置，从⽽控制不同运算状态
    - 注意此处的mie和mip指的是mstatus上的状态位。

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/8fc9e2cc-e3b8-4e35-a141-e06c4c0594a7/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043420Z&X-Amz-Expires=86400&X-Amz-Signature=765d9f07a1d74d23f36bf4aa715664a4801b46754a797f44b5f0d1e63e5eed0d&X-Amz-SignedHeaders=host)

- **WARL：**write any value, read legal values
- **MIE/SIE：**当运行在模式x下时，如果xIE=1，则全局中断启用；如果xIE=0，则全局中断禁用。
    - 中断权限w<x时，其总是被全局禁止的；而当全局中断y>x时，其总是被全局启用的
    - 高权限级别的代码可以通过单独的中断控制位来启用或者禁止某些中断
- **MIPE/SPIE：**xPIE保存进入trap前(嵌套trap)的中断使能位值，即当前xIE，并在进入trap后自动设置当前xIE为0
- **MPP/SPP：**xPP保存进入trap前的优先级模式；其最高只会是x，故MPP为两位，SPP为1位
- 例：当从模式y进入模式x时，xPIE被设为xIE，xIE被设为0，xPP被设为y
- **MRET/SRET：**这是两个指令，用于从M/S模式的trap返回。
    - 当执行xRET时(假定xPP=y)，xIE被还原为xPIE，同时xPIE会被置为1。优先级被设置为y，xPP被设置为U(或者M，如果U不支持的话)
    - 如果xPP!=M，那么xRET还会将MPRV设为0
- SXL/UXL：控制S/U模式下的XLEN的值，32位下不存在xXL
    - 当S/U模式不支持时，SXL/UXL被硬件接0
    - 当XLEN小于最大位时，所有操作会忽略源寄存器中高于XLEN的位，对目的寄存器进行符号扩展填充
- MBE/SBE/UBE：与大小端有关



### mie/mip寄存器

- mie/mip，即Machine Interrup Registers，是MXLEN位的寄存器
- mie保存中断使能位，mip保存中断挂起
    - 中断号i与mip/mie的第i位相关
    - 位0-15只用于标准中断，位16+用于一些自定义的中断
- 当全局中断启用，并且mie/mip的位i置1时，中断i会被响应。
- 一个挂起的中断可通过将mip的第i位清0来清除

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/a7d5c912-f212-4eec-bf14-175615fe6bd7/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043422Z&X-Amz-Expires=86400&X-Amz-Signature=0188a2e4a0a4f82e177bcc46fc98fa42ac3506b01238936784881a3f9c5b551c&X-Amz-SignedHeaders=host)

- MEIP与MEIE用于控制机器级的外部中断；MEIP是只读的，只能通过特定平台的中断控制器进行控制
- METP与METE用于控制机器级计时器中断；MTIP是只读的，通过写入内存映射的machine-mode timer compare register来修改
- MSIP和MSIE用于控制机器级的软件中断；MSIP是只读的，通过写入内存映射的控制寄存器修改
- 当S模式不支持时，SEIP，STIP，SSIP和SEIE，STIE，SSIE被硬件接0；当S模式支持时：
    - SEIP和SEIE用于控制S级的外部中断；SEIP是可读写的，可通过M级的软件修改以表明有一个S级的中断正被挂起
        - 事实上，平台级的中断控制器也可能产生S级外部中断，S级外部中断的挂起其实基于SEIP位和中断控制器信号的或运算
        - 当使用CSR获取SEIP时，得到的就是这样一个或运算的结果；而使用CSRRS/CSRRC获取的SEIP仅基于SEIP
        - 这么做是为了有利于高优先级层模拟外部中断的产生
    - STIP和STIE用于控制S级计时器中断；STIP是可读写的，可以被M级的软件修改以产生计时器中断
    - SSIP和SSIE用于控制S级的软件中断；SSIP是可读写的
- 同时发生的中断执行顺序：MEI, MSI, MTI, SEI, SSI, STI
    - 同步异常优先级低于所有的中断



### mtvec寄存器

- mtvec(即Machine Trap-Vector Base-Address Register)寄存器，是一个MXLEN位的寄存器
- 主要保存machine mode下的trap vector（可理解为中断向量）的设置，其包含⼀个基地址(Base)以及⼀个vector mode。

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/9b144fa3-738a-44f6-8eb8-6946bcf7307f/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043422Z&X-Amz-Expires=86400&X-Amz-Signature=4369a37482b6b9c77abe36eff262bfa2c5a76bfb22a351b5aa6e9d3176b53b7b&X-Amz-SignedHeaders=host)

- Base段的值必须与4对齐，Mode设置可能会对Base的对齐有其他的限制。

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/01fb2c97-1768-4a29-9793-eff806db5c46/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043422Z&X-Amz-Expires=86400&X-Amz-Signature=dcf4a31e982f553506d64a8e861c6f64f84d571fbfea2f8717b640fe6f4df3bd&X-Amz-SignedHeaders=host)

- 当Mode=0，Direct模式下，所有进入M模式的trap都会使PC被设置为Base的值
- 当Mode=1，vectored模式下，所有进入M模式的同步异常都会使PC被设置为Base的值；所有狭义中断会使PC被设置为Base的值加上中断号乘4。
    - 如机器级计时器中断号为7，则跳转地址为base+4*7=base+0x1C



### mcause寄存器

- 进入异常时，机器模式异常原因寄存器mcause被同时更新，以反映当前的异常种类，软件通过读取此寄存器获取异常原因
- mcause的最高位为interrupt位，为1表示中断，为0表示异常，低31位为异常编号位（RV32）

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/d23dd9b2-6484-4514-ba78-f447af2bbce1/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043422Z&X-Amz-Expires=86400&X-Amz-Signature=25b27ac018fb67fb4dab75376290642d3a2f37ea142bbaa753414ea6b64fa987&X-Amz-SignedHeaders=host)



### mtime与mtimecmp寄存器

- 这两个都是64位通过**内存映射**的读写寄存器
- mtime以固定的频率自增(硬件实现)
- **当mtime的值大于mtimecmp时，会产生计时器中断(unsigned number)**

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/d38d3571-23a6-4649-a1d8-b79ca09408ed/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043422Z&X-Amz-Expires=86400&X-Amz-Signature=cb2a2c7ed12561c1072b056f8083ddcddacc8bc084c5d1e4336c33f3060a70c9&X-Amz-SignedHeaders=host)

- 这个中断会一直持续，直到mtimecmp大于mtime，这一般通过修改mtimecmp实现
- 只有当全局中断启用且mie的MTIE置位时才会产生中断



### mtval寄存器

- 在进入异常时，硬件将自动更新机器模式异常值寄存器 mtval (Machine Trap Value Register)，以反映引起当前异常的存储器访问地址或者指令编码。
    - 如果是由存储器访问造成的异常，譬如遭遇硬件断点、取指令、存储器读写造成的异常，则将存储器访问的地址更新到mtval寄存器中。
    - 如果是由非法指令造成的异常，则将该指令的指令编码更新到mtval寄存器中。

> 注意∶mtval 寄存器又名 mbadaddr寄存器，在某些版本的 RISC-V编译器中仅识别mbadaddr名称。



### sstatus寄存器

- 同mstatus，是其的一个子集

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/5f121d19-f93b-485b-8ca9-18c7a1f76028/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043422Z&X-Amz-Expires=86400&X-Amz-Signature=85bbb4c768bb524d5b77e311fbffe5ded8fa65d4832e71232b7f7e82e33ffd76&X-Amz-SignedHeaders=host)

###stvec寄存器

- stvec(即Supervisor Trap Vector Bass Address Register)寄存器，是SXLEN位的可读写寄存器
- 其作⽤与mtvec类似，区别是其保存的是supervisor mode对应的base和mode。

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/0d360e8b-ff78-4e45-8534-b1aace99235a/untitled?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20201113%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20201113T043422Z&X-Amz-Expires=86400&X-Amz-Signature=3ed565da7739a153fda3823020ef9b18ab26c663d20eaede38a489866060c8ad&X-Amz-SignedHeaders=host)

### sp寄存器

- sp寄存器即栈顶指针寄存器，栈是程序运⾏中⼀个基本概念，栈顶指针寄存器是⽤来保存当前栈的寄存器