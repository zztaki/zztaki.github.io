---
title: 初识 CXL
date: 2023-11-14 14:52:57
description: 初识 CXL
tags:
- CXL
categories:
- CXL
- 新硬件
---

# CXL

CXL（**Compute Express Link**，计算快速链接）是处理器、加速器、智能网卡和内存设备之间的高速互连且支持缓存一致性的标准。

CXL 技术保持了 CPU 内存空间和连接设备上的内存之间的一致性，这样就可以实现资源共享（或池化）以提高性能。

## CXL 协议和标准

CXL 标准通过三种协议支持各种应用场景：CXL.io、CXL.cache 和 CXL.memory。

1. CXL.io：该协议在功能上等同于 PCIe 协议。作为基本的通信协议，CXL.io 用途广泛，可满足各种应用场景的需求；
2. CXL.cache：此协议专为更具体的应用程序而设计，使加速器能够有效地访问和缓存主机内存，从而优化性能；
3. CXL.memory：此协议使主机（如处理器）能够使用 load/store 命令访问设备连接的内存。

这三种协议共同促进了计算设备（例如 CPU 主机和 AI 加速器）之间内存资源的一致共享。从本质上讲，这通过共享内存实现通信来简化编程。

## CXL 与 PCIe 的关系

CXL 建立在 PCIe 的物理和电气接口之上（CXL1.1 和 CXL2.0 使用 PCIe5.0 物理层，CXL3.0 使用 PCIe6.0 的物理层），允许备用协议使用 PCIe 的物理层。

支持 CXL 的设备插入插槽后将默认作为 PCIe 设备，而如果主机处理器也支持 CXL，则它们的 CXL 事务协议将得到激活。 

## 为什么需要 CXL？

1. 为提高性能，数据中心使用大量的异构设备，比如从 CPU 卸载专用工作负载到专用加速器上。有了 CXL 后，**CXL 的内存缓存一致性允许在 CPU 和加速器之间共享内存资源，而无需软件层的控制**；
2. **CXL 支持部署新的内存层**，以弥合主内存和 SSD 存储之间的性能差距；
3. **CXL 支持内存扩展功能**，可在当今服务器中提供超出直连 DIMM 插槽的额外容量和带宽，通过连接 CXL 的设备向 CPU 主机处理器添加更多内存，提高内存容量敏感型工作负载（如 AI）的性能。

## CXL 的特性和优势

1. 简化和改进了低延迟连接和内存一致性，从而显著提高了计算性能和效率;
2. CXL 内存扩展功能；

## 哪些设备从 CXL 互连中受益？

1. **智能网卡等通常缺少本地内存的加速器**。通过 CXL.io 和 CXL.cache 协议，这些设备可以与主机处理器的 DDR 内存进行通信；
2. **CPU、GPU、ASIC 和 FPGA 等配备有 DDR 或 HBM 内存的设备**。通过 CXL.io、CXL.cache 和 CXL.memory 协议，CPU 的内存在本地可以供 GPU 等加速器使用，GPU 等加速器的的内存在本地也可以供 CPU 使用，有助于提升异构工作负载；
3. **内存设备**。通过 CXL.io 和 CXL.memory 协议，内存设备可以被扩展和池化，为主机处理器提供额外的内存容量。





## 参考

1. [Compute Express Link (CXL): All you need to know](https://www.rambus.com/blogs/compute-express-link/)
2. 

