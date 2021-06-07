---
layout:     post
title:      使用Qemu运行Trusted Firmware-M
subtitle:   TF-M入门指南
date:       2021-06-07
author:     Weizhou
header-img: img/p7_bg.jpg
catalog: true
tags:
    - TEE
---
## 前言
Trusted Firware-M是ARM针对带有TrustZone-M的小型MCU提供的适配层，提供了PSA接口，
打通了安全世界与非安全世界的通道。TFM中提供了Attestation，Crypto和Secure Storage服务，
并且不同服务提供了隔离机制。本文将叙述如何使用Qemu模拟arm-mps2-an521平台运行TF-M。

## 正文
### 软件版本
**trusted-firmware-m-TF-Mv1.3.0**<br>
官网：<br>
`https://git.trustedfirmware.org/TF-M/trusted-firmware-m.git/`<br>
**Qemu 6.0.0**（源码编译，支持的平台更多）<br>
官网：<br>
`https://download.qemu.org/`<br>
**arm-none-eabi** (GNU Arm Embedded Toolchain 10-2020-q4-major) 10.2.1 20201103 (release)<br>
官网：<br>
`https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads`<br>

### TF-M下载
下载最新版本的TF-M：<br>
`git clone https://git.trustedfirmware.org/TF-M/trusted-firmware-m.git`<br>
一定要使用git下载源码包，在cmake时还会自动下载mbedtls等依赖包。<br>

### 编译TF-M
1. 编译前首先确认python3是否缺少`Ninja`包，缺少的话使用pip安装：<br>
`pip3 install ninja`<br>
2. 随后进入到TF-M根目录下进行cmake配置:<br>
`cmake -S . -B cmake_build -DTFM_PLATFORM=arm/mps2/an521 -DTFM_TOOLCHAIN_FILE=toolchain_GNUARM.cmake -DTEST_NS=ON -DTEST_S=ON -DTFM_PSA_API=ON -DBL2=OFF`<br>
参数说明：<br>
`TFM_PLATFORM`：宏用于指定平台，使用的是arm-mps2-an521平台，该平台模拟的是cortex-m33。<br>
`DTFM_TOOLCHAIN_FILE`：使用编译器类型，这里指定使用arm-none-eabi编译工具链。<br>
`TEST_NS=ON`：编译none-secure测试套。<br>
`TEST_S=ON`：编译secure测试套。<br>
`DTFM_PSA_API=ON`：使用编译PSA接口。<br>
`BL2=OFF`：不使用BL2安全启动，qemu-system-arm运行BL2的镜像会发生异常。<br>
**注：更多编译宏选项详细请参考docs/getting_started/tfm_build_instruction.rst**<br>
3. 进入到`cmake_build`文件夹：<br>
执行`make`<br>
编译成功后会在cmake_build文件夹下生成bin文件夹。

### 使用Qemu运行
1. 首先查看`qemu-system-arm`支持的平台是否包含**mps2-an521**平台：<br>
`qemu-system-arm -machine help`<br>
2. 确认支持该平台后进入cmake_build/bin目录，并运行：<br>
`qemu-system-arm -M mps2-an521 -kernel "tfm_s.axf" -device loader,file="tfm_ns.bin",addr=0x00100000 -serial stdio -display none`<br>
3. 运行成功输出信息：<br>
![tfm_log.png](https://github.com/wwz529247756/wwz529247756.github.io/blob/master/img/p12/tfm_log.PNG?raw=true)<br>

## 结语
后续将继续对TF-M进行进一步的分析。
