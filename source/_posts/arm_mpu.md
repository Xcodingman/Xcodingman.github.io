---
title: ARMv8-M MPU Introduction
catalog: true
date: 2020.9.20
subtitle: 
header-img: 
top: false
tags: peripherals
categories: MCU 
---
# Overview

内存保护单元(Memory Protection Unit,以下简称 MPU)是 ARM 为低端芯片提供的可编程内存保护单元，可以让运行在 privileged 模式下的代码定义不同的内存区域的访问权限。根据处理器的实现，MPU 至多可以实现16个区域的划分，每个区域有独立的访问权限。

<img style="display: block; margin: 0 auto;" src="images/cortexm23_architecture.jpg" alt="" /><br>

上图为 ARM Cortex-M23 处理器的整体架构图，从图上可以看到，每次内存访问都会经过 MPU。
一些特殊的内存访问不受 MPU 的配置影响，比如异常向量表，系统控制空间( System Control Space，其中包括 MPU， NVIC, SysTick 以及 Private Peripheral Bus(PPB, 其中包括debug组件)。而且，对 MPU 的配置不能够定义对 debug 访问权限。

ARMv8-M MPU 支持每个安全状态(non-secure 和 secure)0-8个区域的配置。MPU 的主要特性如下:
* 区域最小大小为32字节，最大为4GB，但必须为32字节的整数倍
* 所有的区域必须以32字节对齐
* 每个区域对两个处理器模式(privileged 和 unprivileged)拥有独立的读/写权限
* eXecure Never(XN)属性可以用来分割代码段和数据段

# Memory type definitions
在 ARMv8-M 架构下，内存空间类型被划分为 Normal Memory 和 Device Memory。如果 ARMv8-M 架构实现了安全扩展，那么内存空间又可以分为 Secure 和 Non-secure 内存区域。
对于 Normal Memory 来说，有以下几个属性设置：

* Cacheability: Cache policy; Allocation; Transient hint
* Shareability: Non-shareable memory, Inner shareable memory, Outer shareable memory
* eXecute Never
  
# Memory Configuration
MPU 可以通过一串连续的寄存器配置，这些寄存器由内存映射到了SCB地址空间，并且这些寄存器在 Secure 和 Non-secure 两个状态下是banked的(banked是指处理器在访问某一地址的寄存器时，实际对应的寄存器实体取决于不同的状态)。在 Secure 和 Non-secure 状态下配置MPU的过程是一样的。在 ARMv8-M 架构下，MPU的寄存器范围为 0xE000ED90 至 0xE000EDC4，此外，Secure 状态下可以通过额外的地址访问 Non-secure 的寄存器。MPU 寄存器只可以在 privileged 模式下访问。MPU 默认关闭在重启之后。下表为 MPU 所有寄存器的地址信息。

<img style="display: block; margin: 0 auto;" src="images/mpu_registers.jpg" alt="" /><br>

# Memory Register Definitions

## MPU_TYPE
MPU 类型寄存器是只读寄存器，它表明 MPU 可用的区域总数

<img style="display: block; margin: 0 auto;" src="images/mpu_type_register.jpg" alt="" /><br>

## MPU_CTRL
由下图可见MPU_CTRL 寄存器有三个功能，第一个是定义使能 privileged background 区域，即没有在mpu区域内的内存空间。第二个是使能 HardFault 和 NMI 受到 MPU 配置影响。第三个是使能 MPU 配置生效。

<img style="display: block; margin: 0 auto;" src="images/mpu_ctrl_register1.jpg" alt="" /><br>
<img style="display: block; margin: 0 auto;" src="images/mpu_ctrl_register2.jpg" alt="" /><br>

## MPU_RNR
MPU_RNR 寄存器负责选择 MPU 的区域号

<img style="display: block; margin: 0 auto;" src="images/mpu_rnr_register.jpg" alt="" /><br>

## MPU_RBAR 
MPU_RBAR 寄存器主要负责定义一个区域的起始地址和访问权限

<img style="display: block; margin: 0 auto;" src="images/mpu_rbar_register1.jpg" alt="" /><br>
<img style="display: block; margin: 0 auto;" src="images/mpu_rbar_register2.jpg" alt="" /><br>

## MPU_RLAR
MPU_RLAR 负责定义区域的大小，选择区域属性配置项以及区域的使能。

<img style="display: block; margin: 0 auto;" src="images/mpu_rlar_register.jpg" alt="" /><br>

## MPU_MAIR0/1
MPU_MAIR0/1 用于设定内存的类型(device memory or normal memory)

## Other registers
除了以上寄存器，MPU 还提供了其他一些寄存器，比如 MPU_RBAR_A1/2/3，MPU_RLAR_A1/2/3。这些寄存器主要是为了加速开发流程，不需要每次配置一个区域需要设置区域号，可以一次配置多个区域。

<img style="display: block; margin: 0 auto;" src="images/mpu_rlara_register.jpg" alt="" /><br>

# Configure an MPU region

<img style="display: block; margin: 0 auto;" src="images/configure_mpu.jpg" alt="" /><br>

# MPU Configuration in FreeRTOS
FreeRTOS 源码中对 background 设置

<img style="display: block; margin: 0 auto;" src="images/backgroud_freertos.jpg" alt="" /><br>

FreeRTOS 源码中对 MPU_MAIR0/1 的设置

<img style="display: block; margin: 0 auto;" src="images/mair_freertos.jpg" alt="" /><br>

FreeRTOS 源码中对用户 stack 的设置

<img style="display: block; margin: 0 auto;" src="images/freertos_stack.jpg" alt="" /><br>

FreeRTOS 源码中对指定 MPU 区域的设置

<img style="display: block; margin: 0 auto;" src="images/freertos_region_configuration.jpg" alt="" /><br>