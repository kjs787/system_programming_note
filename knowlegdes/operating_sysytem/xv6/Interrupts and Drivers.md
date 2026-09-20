# 分配中断的硬件

中断和系统调用，页错误不同。他是由硬件引起的，中断生成和当前进程可以没有任何关系。cpu和中断设备是并行的，硬件设备的手册往往不太清楚，这让编写设备驱动程序变得更加复杂。



## 设备驱动概述

管理设备的代码称为驱动。



**中断程序一般分成top/bottom两个部分：**

bottom部分通常是Interrupt handler。当一个中断送到了CPU，并且CPU设置接收这个中断，CPU会调用相应的Interrupt handler。Interrupt handler并不运行在任何特定进程的context中，它只是处理中断。

top部分，是用户进程，或者内核的其他部分调用的接口。对于UART来说，这里有read/write接口，这些接口可以被更高层级的代码调用。



有一块队列（或buffer），top和bottom部分的代码都可以向该区域读写数据，这里的队列可以将**并行运行的设备和CPU解耦开来**。



## 对设备的编程

一般来说，对设备的编程是通过 `mmap I/O`完成的。设备地址会出现在物理地址的特定区间中，这是主板制造商决定的。通过load/store指令，可以对这些地址进行编程。

下图中是SiFive主板中的**对应设备的物理地址**：

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1786930003776-4509f47b-4076-47c4-a1fa-d8bcb4ae2432.png)



**与中断相关的寄存器**：

- SIE（Supervisor Interrupt Enable）寄存器。这个寄存器中有一个bit（E）专门针对例如UART的外部设备的中断；有一个bit（S）专门针对软件中断，软件中断可能由一个CPU核触发给另一个CPU核；还有一个bit（T）专门针对定时器中断。我们这节课只关注外部设备的中断。
- SSTATUS（Supervisor Status）寄存器。这个寄存器中有一个bit来打开或者关闭中断。每一个CPU核都有独立的SIE和SSTATUS寄存器，除了通过SIE寄存器来单独控制特定的中断，还可以通过SSTATUS寄存器中的一个bit来控制所有的中断。
- SIP（Supervisor Interrupt Pending）寄存器。当发生中断时，处理器可以通过查看这个寄存器知道当前是什么类型的中断。
- SCAUSE寄存器，这个寄存器我们之前看过很多次。它会表明当前状态的原因是中断。
- STVEC寄存器，它会保存当trap，page fault或者中断发生时，CPU运行的用户程序的程序计数器，这样才能在稍后恢复程序的运行。



## 关联硬件

**UART**（Universal Asynchronous Receiver/Transmitter，通用异步收发传输器）是计算机和嵌入式系统中最经典、最基础的**串行通信协议和硬件电路**。允许设备之间进行通信，以**帧**的形式串行传输信息，通过`波特率`实现异步，几乎所有MCU都兼容。

**CLINT**（Core Local Interruptor，核心本地中断控制器）是 RISC-V 架构中专门用于管理**软件中断**和**定时器中断**的硬件模块。

**PLIC**（Platform-Level Interrupt Controller，平台级中断控制器）是 RISC-V 架构中专门用于管理**外部全局中断**的核心硬件模块

# 解析UART驱动程序

这里原文做了大量代码讲解，我准备在做编写网卡驱动的实验再进行补充。

## consumer和producer的解耦

通过buffer实现两者的解耦，当读指针和写指针相等时，sleep读状态，直到缓冲区有空位wake读状态。这样，硬件设备和cpu便能以自己的速度并行运行。