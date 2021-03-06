---
layout:     post
title:      Microcontroller Trustzone隔离技术简介
subtitle:   TrustZone-M简介
date:       2021-01-23
author:     Weizhou
header-img: img/p7_bg.jpeg
catalog: true
tags:
    - TEE
---

## 前言
随着安全相关的热度在IOT设备场景下的不断升温，越来越多的小型设备生产厂商愿意在设备的安全性上进行投资。本文主要简单介绍TrustZone-m相关技术（不仅仅局限于ARM TrustZone，包括近似的安全隔离技术）。ARM自Arm-v8开始对Trustzone技术进行了适配和普及，但在市场上好像并没有很大的使用量，大部分的设备还是基于Arm-v7架构的芯片进行开发。但从2020年开始，因为产品安全性方面的要求，各大厂商也开始选型Arm-v8架构的芯片，此文主要简要讨论该技术的实现方式，以及简要说明TrustZone-M与TrustZone-A机制的区别。

## 正文
### 芯片安全隔离技术简介
芯片相关的隔离安全技术一般可分为两种：一种是类似于TrustZone这种的在一个核上扩展出的安全可执行模式；另一种是SE（Secure Element）是一种多芯片的解决方案，一个核做为A（Application）核，另一个芯片专门作为S（Security）核，同常S核的制成会有一些放回滚抗低温和黑盒相关的硬件安全机制作为防护，安全相关的业务通过IPC（Inter-processor communication）机制完成A核向S核的请求。关于这两种方式哪种更好，网上有不少的文章和博文进行讨论。参考多篇文档后，基本可得到的共识是，两种方式各有千秋，在富设备上很多是两种方案并行的方式。SE因为硬件的安全性更高，可被使用在secure boot阶段作为信任根，或者在里面存储重要的root key，因为这类芯片本身就会有防dump的功能。<br>
**本文主要集中讨论在单一核心上多种执行环境的这种技术场景。**

### TrustZone技术简介
&emsp;&emsp;TrustZone首先是由ARM公司在2004年引入，首先是在Cortex-A核上进行了应用。TrustZone技术在Cortex-A架构上的应用可以说是非常的广泛，现代基于ARM架构的手机或多或少都会有过或正在使用ARM的TrustZone-A技术，用于保证系统以及安全敏感相关业务的安全。<br>
&emsp;&emsp;TrustZone-A引入了安全世界（secure world）和非安全世界（none secure world）的概念，并且能够通过读取SCR（Secure Configuration Register）寄存器的第33位NS（None Secure）比特位进行判断。并且该值可用于外设总线（即通过NS比特位判断哪些设备是安全设备，只能在安全世界进行访问，在硬件层面对外设寄存器的访问进行了限制），以及管控对内存区域的访问（哪片地址空间只能够在安全世界访问，而非安全世界无权访问）。除此之外为CPU引入了新的模式，称为Monitor Mode。Monitor模式是作为安全世界和非安全世界间沟通的存在，无论是从安全世界退出到非安全世界，还是从非安全世界进入安全世界都要经过Monitor模式。从官方的介绍可知，从非安全世界和安全世界进入Monitor Mode有两种方式，一种是通过SMC（Secure Monitor Call）指令，另一种方式是通过在安全世界注册的中断。<br>
&emsp;&emsp;可以说TrustZone-M是基于这种思想在低端Cortex-M芯片上进行的重新设计。同样Trustzone-M也分为安全世界和非安全世界两中模式。但因为Cortex-M芯片本身的性能，主频限制和成本的原因，没有引入Monitor Mode做为安全世界和非安全时间间的桥梁。而是通过引入NSC（None Secure Callable）区域，在非安全世界提供了一组进入安全世界的固定入口实现了从非安全世界进入安全世界的通道。TrustZone-A与TrustZone-M之间的区别如下图所示：
![1.png](https://github.com/wwz529247756/wwz529247756.github.io/blob/master/img/p11/1.PNG?raw=true)<br>
从上图可看出，TrustZone-A的机制向较于TrustZone-M会更加复杂，但入口也更为单一。<br>

### TrustZone-M技术介绍
上述篇幅已经简单介绍了TrustZone-M的一些特点和功能，这一部分主要介绍TrustZone-M实现上述功能的原理和方法。<br>
&emsp;&emsp;首先可以明确，在TrustZone-M的架构下内存空间可分为三个部分，分别是：非安全世界地址空间，安全世界地址空间，和NSC（none-secure callable）地址空间。TrustZone-M架构通过一种各自代码只能在各自世界执行的机制，实现了安全世界和非安全世界的隔离。即安全世界代码只能够在安全世界执行，非安全世界代码只能够在非安全世界执行，防止了跨域执行的可能风险。NSC区域用于从非安全世界进入安全世界，这部分的代码能够被非安全世界调用但要遵循一定的规则。NSC区域由SAU（Security Attribution Unit）或IDAU（Implementation Defined Atrtribute Unit）定义，后续会对这两个单元的机制进行深入的剖析。
&emsp;&emsp;TrustZone-M为实现从非安全世界到安全世界的切换引入了一个新的指令SG（Secure Gateway），并且该指令只能够在NSC区，在其他区掉用改指令会使系统陷入fault。根据官方的介绍，NSC的存在主要的原因是：<br>

`“The reason for introducing NSC memory is to prevent other binary data,for example, a lookup table, which has a value the same as the opcode as the SG instruction, being used as an entry function in to the Secure state.”`<br>

也就是说，防止在非安全世界的任务在被compromised之后任意构造含有SG指令的二进制代码，实现原本在NSC区域中不提供的入口和功能，从而影响安全世界的执行。<br>
&emsp;&emsp;除了SG指令之外，TrusZone-M还提供了从安全世界返回到非安全世界的指令：BXNS（Branch eXchange to None Secure）和BLXNS（Branch with Link eXchange to None Secure）。BXNS指令在当安全世界函数执行结束后反汇到安全世界时执行；BLXNS则是一种从安全世界调用非安全函数的用法，调用的非安全世界函数执行结束后能够根据`Link`返回到安全世界继续执行。除此之外，思路与TrustZone-A相类似的，安全世界和非安全世界的切换同样可以通过exception（异常）和interrupt（中断）注册的handler完成。简单理解就是，在安全世界注册了一个安全设备的中断回调函数，当对应的中断号的信号被送到CPU时，若当前执行在非安全世界，CPU的模式会从非安全世界切换至安全世界，并执行在安全世界注册的回调函数。<br>
安全世界与非安全世界间切换的方式如下图所示：<br>
![3.png](https://github.com/wwz529247756/wwz529247756.github.io/blob/master/img/p11/3.PNG?raw=true)<br>
安全世界与非安全世界间切换代码示例如下图所示：<br>
![4.png](https://github.com/wwz529247756/wwz529247756.github.io/blob/master/img/p11/4.PNG?raw=true)<br>
### SAU与IDAU
&emsp;&emsp;SAU和IDAU分别时`Security Attribution Unit`和`Implementation Defined Atrtribute Unit`的缩写，其实际用途是将一整块物理内存区划分为不同的区域，并为其打上安全世界、非安全世界或者NSC区域的标签，从而在硬件层面防止了代码的跨域访问和执行。
![2.png](https://github.com/wwz529247756/wwz529247756.github.io/blob/master/img/p11/2.PNG?raw=true)<br>
&emsp;&emsp;SAU与IDAU看似功能相似，但出功能类似之外还是有一定的区别。SAU是Arm-v8中必定会包含的单元，但不同的SoC设计可能IDAU并不是必需的。所以SAU可以看作是CPU内置的一种单元，并且可进行变成，IDAU可理解为是外接的原件，并且在出厂后不能进行编程。Arm-v8架构系统boot后是出于安全世界的，此时可能有外部的IDAU已经分配了安全世界和非安全世界的内存片区，但在安全世界中可通过类似于MPU编程的方式修改SAU覆盖已设置的IDAU设置的内存区。<br>
&emsp;&emsp;从上图中可知，因为SAU/IDAU的引入，安全世界和非安全世界各自有一组MPU对内存进行管控。非安全世界中的MPU寄存器数量Arm-v7基本保持一致，都为8个。在安全世界也同样有8个MPU寄存器，用于管控安全世界程序对内存地址的访问，功能与非安全世界一致。安全世界的MPU配置只有当CPU处于安全世界时才能生效，同理非安全世界的MPU亦然。

