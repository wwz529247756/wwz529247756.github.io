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

