---
title: Ditto（SOSP 23）
date: 2023-12-29 12:52:57
description: Ditto（SOSP 23）论文阅读
tags:
- Cache
- 内存分离
- 论文阅读
categories:
- Cache
---

内存缓存系统是云服务的重要组成部分。然而，由于服务器上 CPU 和内存的耦合，现有缓存系统无法以资源有效和快速的方式调整资源。

作者建议将内存缓存系统移植到分离内存（DM）架构中以应对上述问题。但是，在分离内存上构建弹性缓存系统具有挑战：对远程内存的访问绕过了远程缓存服务器上的 CPU，会阻碍缓存算法（promotion 和 demotion）的执行；此外，计算和内存资源的弹性变化（并发客户端数量的改变，缓存容量的改变）会影响数据的访问模式，从而影响缓存算法的命中率。

作者设计了 Ditto，第一个分离内存上的缓存系统，（1）提出以客户端为中心的缓存框架，仅依赖于远程内存访问，就可以在分离内存的计算池中高效地执行不同的缓存算法；（2）采用分布式自适应缓存机制（类 Cacheus）动态选择合适的缓存算法。

# Introduction

内存缓存系统广泛应用于云服务中，以提高吞吐量和降低延迟。然而，由于云服务请求负载的动态变化和突发性，弹性（根据负载变化调整计算和内存资源的能力）对于内存缓存系统至关重要。

然而现有的缓存系统构建和部署在将 CPU 和内存紧耦合的单体服务器上，这将导致动态资源调整上存在两个问题：（1）由于 CPU 和内存必须作为大规模集群上的固定大小虚拟机一起添加或减少，资源利用率收到影响（服务可能只想添加更多内存或 CPU 核心来增加缓存容量或请求吞吐量）；（2）调整资源需要进行长时间的数据迁移，导致资源调整的速度太慢，无法应对请求量的爆发；（**现有问题：资源利用率低，资源分配调整速度慢**）

DM 有机会解决上述问题。它将单台服务器上的 CPU 和内存解耦为独立的计算和内存池，并使用高速互连技术（RDMA 和 CXL）将计算池和内存池互相连接。因此，CPU 和内存可以根据应用需求独立调整，从而提高资源利用率；此外，由于数据被计算池中的所有 CPU 核心共享，因此可以大大降低数据迁移的频率（只有在特殊情况才需要迁移）。

但要在 DM 上实现缓存系统，必须解决两个挑战：

1. 绕过远程缓存服务器上的 CPU 后会阻碍缓存算法的执行。缓存算法通过各自的考量指标维护缓存对象的驱逐优先级，一旦数据访问改变了对象的热度，现有缓存算法**依赖缓存服务器上的 CPU 监视对象热度并维护缓存队列数据结构。**然而，在 DM 上的缓存系统中，计算池里的应用程序客户端在访问对象时将绕过内存池中的 CPU，导致在数据路径上无法更新请求对象的热度；此外如何高效地选择驱逐对象也是一个问题，缓存数据结构只能通过额外的高开销 RDMA 操作进行对象提升和降级，而这些数据结构在高并发场景下又会存在大量锁竞争。
2. 调整资源会影响缓存命中率。缓存命中率和工作负载的访问模式、缓存大小有关。在 DM 上，这两个属性都会随着动态资源的调整而变化。数据访问模式随着并发客户端数量（即计算资源）的变化而变化，缓存大小随着分配的内存空间（即内存资源）的变化而变化。因此，最大化命中率的最佳缓存算法会随着资源设置而动态变化。具有固定缓存算法的缓存系统无法适应 DM 的这些动态特征，并可能导致命中率较低。

作者首先提出带有分布式热度监控、基于采样驱逐功能的以客户端为中心的缓存框架...使用 MAB 动态选择驱逐算法...

贡献：

1. 讨论在 DM 上构建缓存系统的优势和挑战；
2. 提出以客户端为中心的缓存框架，不同的缓存算法可以被灵活地集成并高效地在 DM 上执行；提出一个采样友好地哈希表和频率计数缓存以提高框架效率；
3. 提出分布式自适应缓存，根据动态资源变化和数据访问模式改变选择最佳缓存算法；并通过轻量级驱逐历史和懒惰权重更新机制使得策略在 DM 上高效执行；
4. 合成+真实工作负载测试 Ditto 的性能。

# Background and Motivation

## 单体服务器上的缓存系统问题

1. 资源利用率低。单体服务器上现有的缓存服务的资源（例如 AWS 的 ElastiCache）被以固定大小的虚拟机粒度（1个 CPU，2GB DRAM）进行分配。然而，应用程序有时可能只需要 CPU 或者内存，导致资源利用率低。
2. 资源调整速度慢。现有内存缓存系统将数据分片到多个虚拟机，以利用更多的 CPU 和内存资源。然而，当新虚拟机添加到缓存集群后，缓存数据必须重新分片和迁移，将造成一定程度的性能下降；而在虚拟机回收时，又因为迁移将导致资源回收延迟。

## 分离内存

分离内存被提出以降低总成本和改善云数据中心应用程序的弹性。它将单体服务器的计算和内存资源解耦为独立的计算和内存池，计算池中包含计算节点，具有丰富的 CPU 核心和少量用作运行时缓存的 DRAM，而内存池则具有足够的内存节点和计算能力较弱的控制器（1-2 个 CPU 核心）来执行管理任务（网络连接和内存管理）。**计算节点和内存节点通过高带宽、微秒级延迟的 CPU 旁路互连（RDMA 和 CXL）连接，**保证内存访问的性能要求。计算节点可以通过控制器提供的 ALLOC 和 FREE 接口来分配和释放内存池中可变大小的内存块。

DM 解耦了计算和内存资源后，它们就可以以更细的粒度单独分配，并根据应用程序需求分配准确数量的资源，实现高资源利用率的目标；其次，因为内存池中的缓存数据可以被计算池中的所有计算节点访问，所以它可以大大降低数据迁移的频率，无需在扩容或缩容内存时进行数据迁，大大降低数据迁移成本，使得资源调整迅速生效。

# Challenges

## 在 DM 上执行缓存算法

现有缓存算法都是为以服务器为中心的缓存系统设计的，所有数据都是被服务端的 CPU 访问和驱逐，这种设计模式无法应用于 DM，因为（1）DM 上的缓存系统是以客户端为中心的，所有客户端都是以 CPU 旁路的形式直接访问和驱逐数据的；（2）内存池上的计算能力太弱，难以在数据路径上执行缓存算法。这将导致两个问题。

首先是如何在客户端为中心的环境下评估缓存对象的热度。现有缓存算法通过监视和统计所有数据访问来评估对象热度。由于单体缓存服务器上的 CPU 访问所有数据，所以可以在以服务器为中心的缓存系统上轻松实现监控。但是对于 DM，无法在内存池或客户端上监控缓存对象的访问，因为（1）RDMA 绕过了内存池中的 CPU，（2）**计算池中的各个客户端不知道全局数据访问。**

其次是如何在客户端高效地选择驱逐对象。DM 上的缓存数据结构必须由计算池中的客户端进行维护，导致需要多个 RTT 完成数据获取和缓存数据结构更新操作，导致维护缓存数据结构效率低，并且还可能存在大量的锁冲突。微秒级的锁延迟和多次锁失败导致的网络争用将严重成为缓存系统吞吐量的瓶颈。

左图说明，由于额外的 RDMA 操作，导致 KVC 和 KVC-S 的吞吐率和延迟性能较差，右图说明由于锁竞争，它们的吞吐量差。

{% asset_img execute_cache_algo_on_dm_challenge.jpg execute_cache_algo_on_dm_challenge %}

## 动态资源变化影响命中率

缓存命中率与数据访问模式、缓存大小密切相关。当动态调整计算和内存资源时，这两者都会受到影响。此时，具有固定缓存算法的缓存系统无法适应，可能导致命中率较低。

1. 改变计算资源影响命中率；

    在 DM 的缓存系统上，应用程序在计算池中执行多个客户端线程去并发访问内存池中的缓存数据，缓存对象的访问模式将是所有应用程序访问模式的混合。

    当计算资源变化时，一个应用程序的客户端线程数量也将变化，会改变访问模式的整体组合，并以两种方式影响单个缓存算法的命中率。

    - 首先，一个应用程序的数据访问占比随着其客户端线程的数量而变化；下图显示，随着 LRU 友好应用程序的客户端线程数占比增加，其命中率增高。（占比低时命中率低，应该是由于其它应用程序挤占了命中机会）

        {% asset_img different_application_client_affect_hit.jpg different_application_client_affect_hit %}

    - 其次，客户端的并发执行改变了工作负载的原始访问模式。左图坐标（0.25, 0.5）表示，使用 LRU 算法，在 50% 的工作负载上，它们相对命中率变化高于 0.25，即并发执行存在不确定性，同一工作负载命中率可能不稳定；右图显示，对于同一个应用程序，计算资源的变化将导致最优缓存算法改变。

        {% asset_img concurrent_number_affect_hit.jpg concurrent_number_affect_hit %}

2. 改变内存资源影响命中率；

    对于同一工作负载，改变内存资源会影响命中率，甚至于改变其访问模式（比如从搅拌变成 LFU 友好型）。

    {% asset_img memory_resize_challenge.jpg memory_resize_challenge %}

# Design

## Overview

每个应用程序拥有一个本地 Ditto 客户端作为子进程，而每个 Ditto 客户端有多个线程，可以通过增加或删除分配给 Ditto 的线程数和 CPU 核心数来自由扩展计算资源。

Ditto 客户端通过使用单边 RDMA 动作执行 get 或 set 操作。get 操作将使用一次 RDMA_READ 获取缓存对象在哈希表中的地址，再使用一次 RDMA_READ 从这个地址中获取对象的实际数据；set 操作需要通过一个 RDMA_READ 获取哈希表中的一个合适槽，一个 RDMA_WRITE 将新对象写入到内存池，一个 RDMA_CAS 原子修改槽中的指针。

{% asset_img overview.jpg overview %}

## 以客户端为中心的缓存框架

以客户端为中心的缓存框架可以解决在 DM 上执行缓存算法时评估对象热度和选择驱逐对象的挑战。

首先，Ditto 将每个对象与记录其全局访问信息（例如访问时间戳、频率等）的元数据相关联，在每次客户端执行 get 或 set 操作后，元数据同样由具有单边 RDMA 动作的客户端进行更新。在客户端，Ditto 提供了两个接口来继承缓存算法，分别是优先级接口（`double priority(Metadata)`）和元数据更新接口（`void update(Metadata &)`）。优先级函数将对象的元数据映射到指示其热度的实际值，元数据更新函数在数据重用时更新元数据信息（频率、新近度等等）。通过实现这两个接口可以轻松集成多种缓存算法。

其次，为了有效选择驱逐对象，Ditto 客户端对内存池中的缓存对象进行采样，并对采样对象应用优先级函数进行评估，驱逐优先级最低的对象。如此一来，就避免了维护缓存数据结构的昂贵开销。（**一般的缓存算法往往需要一个哈希表和一个链表，MAB 使得需要 N 个哈希表和 N 个链表，而此时降低精确度，使用采样技术，又变成只需一个哈希表了，缓存对象的优先级顺序不再需要维护，只需通过 RDMA 更新其访问信息即可**）

为了有效在 DM 上执行此框架，**Ditto 提出一个采样友好的哈希表和一个频率计数器缓存，以减少在 DM 上采样和记录访问信息的开销**。

### 采样友好哈希表

采样友好哈希表有助于减少 DM 上的采样和更新访问信息的开销，这是因为在 DM 上采样需要进行多次 RDMA_READ 操作，延迟很高；此外，更新访问信息也会影响整体吞吐量，因为额外的 RDMA 操作会消耗内存池中 RNIC  的有限消息速率。

Ditto 将广泛使用的元数据与哈希索引中的槽存储在一起，同时保留其它不广泛的元数据（高级缓存算法需要）扩展功能。如此一来，通过直接获取哈希表中具有随机偏移量的连续槽，仅用一次 RDMA_READ 便可以进行采样（？）；其次，根据对象的更新频率组织它们的访问信息，从而减少更新对象元数据的 RDMA 操作数量，组织良好的访问信息可以使用单个 RDMA_WRITE 更新多个访问信息。

1. 哈希表结构

    哈希表有多个桶，每个桶有多个槽，每个槽由两个部分组成，即原子部分和元数据部分。原子部分长度为 8 个字节，包含指向对象的指针，加速对象搜索的指纹和对象大小信息。在插入、更新或删除对象时使用 RDMA_CAS 进行原子修改；元数据部分记录了大多数缓存算法所需的访问信息，此外，还为分布式自适应缓存方案记录了一个附加的哈希字段（后续介绍）。

    {% asset_img sample_friendly_hash_table.jpg sample_friendly_hash_table %}

2. 访问信息组织

    首先，Ditto 通过区分本地信息和全局信息来减少必须包含在元数据中的访问信息数量。全局信息需要由所有客户端协作维护，因此必须包含在元数据中；而本地信息（比如延迟和成本）可以由分布式客户端在本地决定，无需包含在内。全局信息又可以分为有状态信息和无状态信息，无状态信息（比如插入时间和新近度）通过覆盖旧值进行更新，而有状态信息（比如频率）根据其旧值进行更新。

    Ditto 通过将无状态信息组织在一起，使用单个 RDMA_WRITE 进行更新；使用 RDMA_FAA 更新有状态信息。

### 频率计数器缓存

使用客户端频率计数器（FC）缓存进一步减少更新元数据的开销。对于采样友好的哈希表，更新元数据仍然需要两个 RDMA 操作，即更新无状态信息的 RDMA_WRITE 和更新有状态信息的 RDMA_FAA，这些 RDMA 操作会消耗 RNIC 的效率速率，从而限制 Ditto 的正太吞吐量；此外，由于 RNIC 内部锁的争用，在 DM 上执行 RDMA_FAA 的成本很高。FC 缓存旨在减少元数据更新时的 RDMA_FAA 数量。

短时间内多个写入指令可能会针对同一内存区域，FC 缓存则是作为写 buffer，吸收短时间内对同一区域的写操作，将其转换为单个内存写操作，以节省内存带宽。具体来说，**FC 缓存条目包含对象 ID，哈希表中槽的地址以及计数器增量值**。每次访问对象时，其对频率计数器的更新都会缓冲在 FC 缓存中，对象在内存池哈希表中实际的频率计数器更新会被推迟，直到缓存条目被驱逐。

缓存条目在两种情况会被驱逐：缓存容量不足，FIFO 驱逐；计数器增量大于阈值 t。在条目被驱逐时，根据记录的槽地址将缓冲的计数器值增加相应的值。如此一来，RDMA_FAA 的操作数最多减少至了 1/t。

## 分布式自适应缓存

Ditto 提出一个分布式自适应缓存机制来适应工作负载变化和动态资源配置。

在 DM 上实现自适应缓存，必须解决两个挑战。首先，由于访问远程数据结构开销较高，维护全局 FIFO 逐出历史的成本很高；其次，由于客户端需要同步获取更新的权重，在分布式客户端上管理专家权重开销很大。

为了解决上述挑战，Ditto 将驱逐历史条目嵌入到具有轻量级逐出历史的哈希表中，以避免在 DM 上维护额外的 FIFO 队列；Ditto 提出一种懒惰权重更新方案，以避免客户端之间昂贵的同步开销。

### 轻量级驱逐历史

**重用**采样友好哈希表的槽来存储和索引历史条目，从而无需分配额外的空间和为历史条目构建额外的哈希索引；提出一种具有懒惰驱逐方案的逻辑 FIFO 队列，以有效地实现对历史条目的 FIFO 替换，从而无需维护额外的 FIFO 队列。

1. 嵌入历史条目

    size 域 0xFF 指代这是个驱逐历史条目，pointer 域不再存储对象的地址，而是一个历史 ID；insert_ts 域则是改为存储专家位图，指示先前是哪些专家决定驱逐该对象；hash 域记录被驱逐对象 ID 的哈希值，以检查驱逐历史中是否包含缓存缺失的对象。

    {% asset_img lightweight_history.jpg lightweight_history %}

2. 逻辑 FIFO 队列

    有一个 6 字节的全局循环计数器为新的历史条目生成历史 ID，所有客户端都知道它的地址。

    {% asset_img logic_fifo.jpg logic_fifo %}

    - 驱逐历史条目插入：当客户端决定从缓存中逐出对象时，客户端将首先通过在全局历史计数器上执行 RDMA_FAA 来获取历史 ID，并使计数器自动加一，然后发送 RDMA_CAS 以原子方式将驱逐对象的大小和指针分别修改为 0XFF 和获取的历史 ID，再使用 RDMA_WRITE 将专家位图异步写入槽元数据部分的 insert_ts 字段；
    - 懒惰历史条目驱逐：在客户端实行过期检查，如果全局历史计数器值为 v1，历史 ID 为 v2，并且 FIFO 历史大小为 L。当 v1 > v2，则当 (v1 + 2^48^ - v2) mod 2^48^ > L 时条目将无效。然而，实际的历史条目驱逐发生在有新的缓存对象（还是新的历史条目？）将要插入时，过期的槽被认为是空的，从而做到透明地驱逐历史条目。
    - 遗憾收集：遗憾被定义为客户端发现缓存中缺失的对象包含在驱逐历史中。客户端搜索对象时，会计算对象 ID 的哈希值，根据哈希值定位到一个桶，并迭代匹配桶中的槽，看看指向的对象是否与目标具有相同的对象 ID。在此过程中，客户但还会匹配桶中遇到的历史条目的哈希值，如果尚未找到对象但历史条目具有匹配的哈希值，则可以收集遗憾。

    {% asset_img history_insert_evict.jpg history_insert_evict %}

### 懒惰专家权重更新

当客户端发现遗憾时，需要降低决定驱逐该对象的专家的权重。然而，由于 DM 上高同步开销，更新和使用分布式客户端的专家权重会产生不可忽略的开销。懒惰权重更新方案的思想是让客户端在本地批处理遗憾并将权重更新懒惰地卸载到内存节点地控制器上，这样旧减少了更新权重的频率，避免了同步的开销，同时，由于更新不频繁，内存节点的弱控制器也不会成为瓶颈。

{% asset_img lazy_weight_update.jpg lazy_weight_update %}

上图展示了懒惰权重更新方案的流程。每个客户端在本地维护专家权重以做出驱逐决策，当客户端发现遗憾时，它根据历史条目中的专家位图将惩罚应用于本地专家权重，惩罚被记录在惩罚 buffer 中。当缓冲的惩罚超过阈值时，客户端通过基于 RDMA 的 RPC 请求将所有惩罚发送到保存专家权重的内存节点的控制器。在受到客户端的惩罚后，内存节点的控制器首先将惩罚应用于全局专家权重，然后将更新后的全局权重回复给所有客户端。

为了减少通过网络传输惩罚的带宽消耗，Ditto 利用指数函数的属性（指数相加）压缩惩罚，从而使得惩罚列表变成单个值。

尽管本地客户端的专家权重并不总和全局权重同步，但是作者说这个并没有影响 Ditto 的适应性。

## Discussions

**元数据扩展。**如果有额外元数据（没有存储在哈希表中），执行 Get 和 Set 操作后，需要额外的 RDMA_WRITE 来异步更新额外元数据；在缓存驱逐时，需要额外的 RDMA_READ 获取额外元数据一同输入到优先级函数中。

**元数据开销。**每个历史条目占 40 字节，历史条目总数和缓存对象的最大数量相等；对于每个缓存对象，在哈希表中需要占据 40 字节；对于每个专家，需要一个 4 字节浮点数。

# Evaluation

**mark：如何将 trace 分发到多个客户端？**

缓存驱逐时的采样大小：5（和 Redis 一样）；

历史记录大小：Cache Capacity（和 LeCaR 一样）；

FC Cache 的阈值：10，FC Cache 的容量：10 MB（实验结果评估）；

MAB 学习率：0.1，本地 100 次的权重更新再传输给全局权重。

## 弹性

基于 DM 的缓存系统 Ditto 相比于基于传统单体服务器的 Redis 来说，资源利用率以及资源切换时的调整速度都更好。

{% asset_img elasticity.jpg elasticity %}

## 效率

为了评估 Ditto 在 DM 上可以高效执行缓存算法，作者在 YCSB 基准上评估了其与 Shard-LRU，CliqueMap 的吞吐量和尾延迟。Ditto 的性能瓶颈仅在 RNIC 的传输速率上；而 Shard-LRU 会随着客户端数量的增加，远程锁争用成为瓶颈；CliqueMap 的瓶颈则在于内存池的计算能力

{% asset_img efficiency.jpg efficiency %}

{% asset_img more_cores_on_MN.jpg more_cores_on_MN %}

## 适应性

### 适应不同的真实工作负载

MAB 集成多种缓存算法，使得可以根据不同的访问模式切换最合适的缓存算法，能够适应大部分工作负载。

### 适应动态资源调整

计算资源变化（客户端数量改变）和内存资源变化（缓存大小改变），Ditto 都能够很好地适应。

## 灵活性

只需少量的额外代码，就可以集成多种缓存算法。

## 消融实验

Design 部分各自的技术对于吞吐量的提升。

# 总结

本文提出了 Ditto，第一个基于内存分离架构的缓存系统，可以实现更好的弹性。

Ditto 解决了在 DM 上构建缓存系统的挑战，即**执行现有以服务器为中心的缓存算法并处理动态变化的资源和数据访问模式导致的低命中率**。具体地说，它通过提出一种以客户端为中心的缓存框架，以有效地在 DM 上执行缓存算法；提出一种分布式自适应缓存方案来适应资源和工作负载的变化。

实验结果表明，Ditto 有效地适应了 DM 上的资源和工作负载变化，并且在 YCSB 合成工作负载上优于单片服务器上最先进的缓存系统高达 9 倍，在现实世界的键值跟踪上优于最先进的缓存系统 3.6 倍。