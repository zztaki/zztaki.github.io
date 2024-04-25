---
title: curve_source_code
date: 2023-12-04 15:17:54
tags:
- 分布式存储
hidden: true
---

# curve 源码分析

## curvefs

### 创建文件系统

**curve-fuse**：（在 curveadm mount 指令执行后作为守护进程存在）

1. InitFuseClient，初始化 fuse 文件系统客户端及客户端配置；

    - 初始化 mds 客户端；

    - 初始化 meta 客户端；

        ......

    - mdsClient_->MountFs 发送 RPC 请求给 mds 服务进行挂载；

**mds**：



