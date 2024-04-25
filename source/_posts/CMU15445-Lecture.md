---
title: CMU15445-Lecture
date: 2023-3-14 20:13:46
description: CMU15445-Lecture
tags:
- CMU15445
- 公开课
- 数据库
categories:
- 数据库
---

# Lec1 关系模型

1. 每个关系（Relation）都是一个无序集合，也叫数据库表，集合中每个元素都是一个元组（tuple），每个 tuple 由一组属性构成，这些属性在逻辑上通常有内在联系。
2. 主键（primary key）在一个关系中唯一确定一个 tuple。
3. 外键（foreign key）唯一确定另一个关系中的一个 tuple。

4. Data Manipulation Language（DML），数据操作语言，比如增删改查。
5. Data Definition Language（DDL），对数据结构进行修改的语言，比如加索引，建表等。
6. 在关系模型中查找和查询数据的两种方式，使用哪种方式是具体的实现问题，与 Relational Model 本身无关。
    - Procedural：查询命令需要指定 DBMS 执行时的具体查询策略，如关系代数（Relational Algebra），会因操作步骤影响执行效率；
    - Non-Procedural：查询命令只需要指定想要查询哪些数据，无需关心幕后的故事，如 SQL。

# Lec2 SQL 进阶

## SQL 历史

Structured Query Language，结构化查询语言。

## SQL 特性

1. Aggregates：聚类函数，通常可以将查询结果聚合成一个值。
    - AVG(<distinct> col)
    - MIN(col)
    - MAX(col)
    - SUM(<distinct> col)
    - COUNT(<distinct> col)
2. Group By：group by 就是把记录按某种方式分成多组，对每组记录分别做 aggregates 操作。**所有非 aggregates 操作的字段，都必须出现在 group by 语句**。
3. Having：基于 aggregation 结果的过滤条件不能写在 WHERE 中，而应放在 HAVING 中，从而对 group by 结果进行进一步筛选。
4. Output Control：
    - `ORDER BY <column*> [ASC|DESC]`
    - `LIMIT <count> [offset]`
5. Common Table Expressions：with 视图（虚拟中间表）。
6. Nested Queries：子查询。
7. String Operations
    - string match
    - string functions
8. Window Functions：`ROW_NUMBER() OVER(<PARTITION BY ...>)`
9. Date/Time Operations

# Lec3&4 Database Storage

## Disk Manager

1. 为什么不适用 OS 磁盘管理的轮子？主要原因在于，OS 的磁盘管理模块并没有、也不可能会有 DBMS 中的领域知识，因此 DBMS 比 OS 拥有更多、更充分的知识来决定数据移动的时机和数量（OS 只是提供一种普适化的接口，但现在既然具体化到了数据库的场景，那么我们在拥有这种特定场景下的负载特征，自然就可以特制化一些更优秀的策略），具体包括：
    - Flushing dirty pages to disk in the correct order
    - Specialized prefetching
    - Buffer replacement policy
    - Thread/process scheduling
2. DBMS 的磁盘模块主要解决两个问题：
    - How the DBMS represents the database in files on disk（如何使用磁盘文件来表示数据库的数据，比如元数据、索引、数据表等）
    - How the DBMS manages its memory and moves data back-and-forth from disk（如何管理数据在内存与磁盘之间的移动）

## 在磁盘文件中表示数据库

### File Storage

DBMS 通常将自己的所有数据作为一个或多个文件存储在磁盘中，而 OS 只当它们是普通文件，并不知道如何解读这些文件。

1. Storage Manager：存储管理器负责维护数据库文件，为读写操作安排合适的调度，从而改善页面的时间和空间局部性
2. Database Pages：一个页是固定大小的数据块，每个 page 内部可能存储着 tuples、meta-data、indexes 以及 logs 等等，大多数 DBMS 不会把不同类型数据存储在同一个 page 上。每个 page 带着一个唯一的 id，DBMS 使用一个 indirection layer 将 page id 与数据实际存储的物理位置关联起来。
3. Database Heap：有几种方法可以找到 DBMS 想要的页面在磁盘和堆文件中的位置，heap 就是其中一种方式。堆文件是页面的无序集合，以随机顺序存储元组，也就是说我们保存的数据无须按照我们插入时的顺序进行保存。（Linked List 和 Page Directory）。

### Page Layout

{% asset_img page_layout.jpg page_layout  %}

每个 page 被分为两个部分：header 和 data。

header 中通常包含以下信息：

- Page Size
- Checksum
- DBMS Version
- Transaction Visibility
- Compression Information

data 中存储真正的数据，包含数据本身和数据的操作日志：Tuple-oriented 和 Log-structured

1. Tuple-oriented：

    - Strawman Idea：在 header 中记录 tuple 的个数，然后不断的往下 append 即可。这种方法有两种明显问题，（1）一旦出现删除操作，每次插入就需要遍历一遍，寻找空位，否则就会出现碎片；（2）无法处理变长的数据记录（tuple）

        {% asset_img Strawman.jpg Strawman %}

    - Slotted Pages：常用。

        {% asset_img slot_page.jpg slot_page %}

2. Log-structured：KV 数据库中好用，因为 KV 数据库中每个键只有一个值，因此只需从日志栈顶往下找第一个 update 就行了，而关系数据库有多个字段，只找到一个 update 可能是不够的。

    - Slotted-Page Design 存在一些问题：（1）碎片化：删除元组会在页面中留下空白（即时能够迁移也需要额外开销）；（2）无用的磁盘 I/O：由于非易失性存储的面向块的特性，整个块需要被读取以获取元组；（3）随机磁盘 I/O：磁盘读取器可能必须跳转到 20 个不同的位置才能更新 20 个不同的元组，这非常慢。因此日志结构存储模型仅允许创建新数据而不允许覆盖，解决了上面列出的一些问题。

    - 写入速度快，读取速度可能较慢。磁盘写入是顺序的，现有页面是不可变的，导致随机磁盘 I/O 减少。
    - 为了加快读取，可以创建索引以跳转到日志中的一些特定位置。
    - 必要时需要压缩日志。

### Tuple Layout

tuple-oriented 中的数据本身也有别的元信息，比如：

- Visibility information for the DBMS’s concurrency control protocol (i.e., information about which transaction created/modified that tuple).
- Bit Map for NULL values. 
- Note that the DBMS does not need to store meta-data about the schema of the database here.

tuple data 中的属性通常按照创建表时指定的顺序存储。

{% asset_img tuple_data.jpg tuple_data %}

数据库中的每个元组都分配有一个唯一标识符，最常见的是 `page id + (offset or slot)`，DBMS 可以创建一个 map，将不同的 tuple 映射到 `page id + (offset or slot)`。

### Tuple Storage

元组中的数据本质上只是字节数组。由 DBMS 决定如何解释这些字节以导出属性的值。数据表示方案是 DBMS 如何存储这些字节。有五种高级数据类型可以存储在元组中：integers, variable-precision numbers, fixedpoint precision numbers, variable length values, and dates/times。

为了让 DBMS 能够破译元组的内容，它维护了一个内部目录（catelogs）来描述数据库的元数据。这些元数据包括数据库包含的表和列，以及列中的属性的类型以及列的顺序。大多数 DBMS 同样以表的格式将它们的 catelogs 存储在自身内部。

## 总结

1. database is organized in pages;
2. different ways to track pages：Heap File Organization...;
3. different ways to store pages：Linked List 和 Page Directory;
4. different ways to store tuples：Strawman，slotted 和 Log-structured;



# Lec5 Storage Models & Compression

## Database Workload

1. OLTP：Online Transaction Processing，运行速度快、运行时间短、对单个实体进行简单查询。OLTP 工作负载通常会处理更多的写入操作。
2. OLAP：Online Analytical Processing，长时间运行、复杂查询、读取数据库中的大部分数据。在 OLAP 工作负载中，数据库系统通过分析大量现有数据，并期望从中获得结论。
3. HTAP：Hybrid Transaction + Analytical Processing，尝试在同一个数据库上同时执行 OLTP 和 OLAP 的组合。

{% asset_img workloads.jpg workloads %}

## Storage Models

### N-Ary Storage Model (NSM)

将单个元组的所有属性连续存储在单个页中，这种方法非常适合 OLTP 工作负载，其中请求插入量大且事务倾向于操作单个实体。这是理想的，因为只需要一次读取就可以获取单个元组的所有属性。

1. 优点
    - 快速插入、更新和删除。
    - 适用于需要整个元组的查询。
2.  缺点：不适合检索表的大部分元组，或者只需要查找小部分的属性。

### Decomposition Storage Model (DSM)

将所有元组的单个属性连续地存储在一个 page 中，这种存储方式特别适用于 OLAP 场景。

1. 优点
    - 减少了浪费的 I/O 量，因为 DBMS 只读取该查询所需的数据。
    - 更好的查询处理和数据压缩支持。
2. 缺点
    - 由于元组拆分/拼接，点查询、插入、更新和删除速度较慢。
    - 引入新问题：如何跟踪每个元组的不同属性？（1）Fixed-length Offsets：每个属性都是定长的，直接靠 offset 来跟踪（常用），但是不灵活；（2）Embedded Tuple Ids：在每个属性前面都加上 tupleID，但是需要额外的存储开销。

**chose right storage model for the ：OLAP（column store） 和 OLTP（row store）。**

## Database Compression

压缩对于基于磁盘的 DBMS 来说非常重要，因为磁盘 I/O 几乎总是主要的瓶颈；DBMS 可以通过压缩页面以提高每个 I/O 操作能够移动的数据有效量，尤其是在只读分析工作负载中。

如果数据集是完全随机的，那么将没有办法进行压缩。但是，真实世界中的数据集往往存在 key insight，比如：

1. 数据集往往具有高度偏斜的属性值分布（例如，齐夫分布）
2. 同一元组的属性之间具有高度相关性（例如，邮政编码和城市，订购日期和发货日期）。

基于此，我们希望数据库压缩方案具有如下性质：

1. 必须产生固定长度的值。唯一的例外是存储在单独池中的变长数据，这是因为 DBMS 应该遵循字对齐并能够使用偏移量访问数据。
2. 允许 DBMS 在查询执行期间尽可能长时间地推迟解压缩（延迟实现）。
3. 必须是无损方案，因为人们不喜欢丢失数据。任何一种有损压缩都必须在应用程序层级执行。

在向 DBMS 添加压缩功能之前，我们需要确定要压缩的数据类型，这决定了压缩方案是否可用。压缩粒度有四个级别：

1. 块级：压缩一个块，这个块中的所有元组隶属于同一张表。
2. 元组级别：压缩整个元组的内容（仅限 NSM）。
3. 属性级别：在一个元组中压缩单个属性值。可以针对同一个元组的多个属性。
4. 列级：为多个元组压缩一个或多个属性值（仅限 DSM）。这允许更复杂的压缩方案。

## Naive Compression

DBMS 使用通用算法（例如 gzip、LZO、LZ4、Snappy、Brotli、Oracle OZIP，Zstd）。尽管 DBMS 可以使用多种压缩算法，但工程师经常选择具有较低压缩率的压缩算法以换取更快的压缩/解压缩速度。

MySQL InnoDB 中有一个使用简单压缩的示例。DBMS 压缩磁盘页、往里面填充 2 的幂次方 KB 并将它们存储到缓冲池中。然而，每次 DBMS 尝试读取数据时，缓冲池中的压缩数据需要解压。

由于访问数据需要对压缩数据进行解压，这就限制了压缩机制的范围。如果目标是将整个表压缩成一个巨大的块，使用简单的压缩方案
这是不可能的，因为每次访问都需要压缩/解压缩整个表。因此，对于 MySQL，由于压缩范围有限，它将表分成更小的块。

另一个问题是这些简单的方案也没有考虑数据的高级特征或语义。这些算法既不关心数据的结构，也不关心查询计划如何访问数据。因此，这放弃了利用 `late materialization` 的机会，因为这样的话 DBMS 无法知道什么时候可以延迟数据的解压缩。

## Columnar Compression

### Run-length Encoding

### Bit-Packing Encoding

### Bitmap Encoding

### Delta Encoding

### Incremental Encoding

### Dictionary Encoding

# Lec6 Memory Management：Buffer Pools

DBMS 的磁盘管理模块主要解决两个问题，一个是在 Lec3 & Lec4 中提到的**如何使用磁盘文件来表示数据库的数据（元数据、索引、数据表等）**，另一个就是本节将介绍的**如何管理数据在内存与磁盘之间的移动**。

## 介绍

DBMS 负责管理其内存并从磁盘来回移动数据。因为，对于大多数情况，数据不能直接在磁盘上操作，任何数据库都必须能够高效地移动以文件形式表示在磁盘上的数据，将它们装入内存，以便可以使用。

DBMS 面临的问题是将移动数据的延迟降到最低。理想情况下，延迟应该是0，也就是用户使用数据之前，数据已经在内存中了（预取）。而执行引擎不必考虑数据是如何被预取到内存的（内存管理器考虑）。

{% asset_img data_move.jpg data_move %}

考虑这个问题的另一种方法是划分为空间和时间控制。

1. 空间控制策略通过决定将 pages 写到磁盘的哪个位置，使得常常一起使用的 pages 在磁盘的物理距离更近，从而提高 I/O 效率。
2. 时间控制策略通过决定何时将 pages 读入内存，写回磁盘，使得磁盘读写的次数最少，从而提高 I/O 效率。

## Locks vs. Latches

在讨论 DBMS 如何保护其内部元素时，我们需要分清 `lock` 和 `latch`。

**lock**：lock 是一种更高层级别的逻辑原语，用于保护一个事务中数据库的内容（例如，元组、表、数据库）。事务将在其整个持续时间内持有锁。数据库系统可以向用户公开运行查询时持有哪些 lock。此外，lock 需要能够支持回滚修改。

**latch**：latch 是一种低层级的保护原语，DBMS 在其内部数据结构（例如哈希表、内存区域）中使用以保护临界区。latch 仅在执行操作期间保持。latch 不需要支持回滚修改。

{% asset_img lock_latch.png lock_latch %}

## Buffer Pool

缓冲池本质上是数据库内部分配的一个大内存区域，用于存储从磁盘获取的页面。

DBMS 启动时会从 OS 申请一片内存区域，即 Buffer Pool，并将这块区域划分成大小相同的 pages，为了与 disk pages 区别，通常称为 frames。当用户请求一个页面时，DBMS 会先查询缓冲池，如果没有找到页面，则 DBMS 会进一步请求原始的 disk page ，并将该 disk page 复制到 Buffer Pool 的一个 frame 中。

### Buffer Pool 元数据

缓冲池必须维护一定的元数据才能被高效且正确地使用。

首先，**page table** 是一个内存中的哈希表，用于跟踪当前内存中的页面。它将 page id 映射到缓冲池中的帧位置。

注意：不要将 page table 和 page directory 混淆，page directory 是将 page id 映射到该页在数据库文件中的位置。对页面目录的所有更改都必须记录在磁盘上，以便 DBMS 可以在重启后找到。

page table 还维护每个页面的附加元数据、dirty flag 和引用计数器。

1. dirty flag 由线程在修改页面时设置。这向存储管理器表明该页面已被修改，必须写回磁盘以持久化。
2. 引用计数器跟踪当前访问该页面（读取或修改它）的线程数。线程必须在访问页面之前递增计数器，如果页面的计数大于零，则
    不允许存储管理器从内存中逐出该页面。

### Memory Allocation Policies

缓冲池中的缓存有两种分配策略。

全局策略：考虑所有活动事务以找到分配内存的最佳决策，以使正在执行的整个工作负载受益（比如最小化全局的磁盘读写次数，类似缓存分配/分区策略中的 UCP，相对于动态地为缓冲池分区了）。

局部策略：它做出的决策将使单个查询或事务运行得更快，即使这对整个工作负载不利。局部策略将帧分配给特定事务而不考虑正在并发进行的其它事务行为（first-in-first-out，对每个事务 ”屡求屡给“，也就相当于没有对缓冲池分区）。

大多数系统结合使用全局视图和局部视图。

## Buffer Pool 优化

### Multiple Buffer Pools

DBMS 可以拥有多个缓冲池，比如为每个数据库设置一个缓冲池，为每种页面类型（data page、directory page...）设置一个缓冲池，从而每个缓冲池可以为特定的数据库、页面类型设置局部的缓存策略。与此同时，在处理并发事务时，也可以减少 latch 的争用。

### Prefetching

缓存预取策略。比如在全局扫描查询中，可以预先将该表接下来将扫描的 page 预取到缓冲池中；将 B+ 树的根节点预取到缓冲池中。

### Scan Sharing (Synchronized Scans)

Scan Sharing 技术主要用在多个查询存在数据共用的情况。当两个查询 A, B 先后发生，B 发现自己有一部分数据与 A 共用，于是先共用 A 的 cursor，等 A 扫完后，再扫描自己还需要的其它数据。

### Buffer Pool Bypass

缓存准入策略。全表扫描的 page 可能无需加入缓冲池中，因为它们的时间局部性较差，加入缓存可能会驱逐缓冲池中其它一些局部性较好的页面。因此，DBMS 可能单独分配一块局部内存（Bypass，旁路），在该内存中的写入和处理不影响缓冲池中的其它数据。

## OS Page Cache

大部分 disk operations 都是通过系统调用完成，通常系统会维护自身的数据缓存，这会导致一份数据分别在操作系统和 DMBS 中被缓存两次。大多数 DBMS 都会使用 (O_DIRECT) 来告诉 OS 不要缓存这些数据，除了 Postgres。

## Buffer Replacement Policies

缓存替换策略。

### localization

每次事务查询只能驱逐有限的缓冲池页面，或者是拥有一份自己独享的缓冲池区域（缓存分区）。

### priority hints

允许事务告知缓冲池哪些页面非常重要，比如根据页面的具体内容，或者是类似于 B+ 树的根节点。

### Dirty Pages

驱逐一个 dirty page 的成本要高于驱逐一般 page，因为前者需要写 disk，后者可以直接 drop，因此 DBMS 在驱逐缓冲池中的 page 时，需要权衡页面局部性和写回的开销。

除了直接在 Replacement Policies 中考虑，有的 DBMS 使用 Background Writing 的方式来处理。它们定期扫描 page table，发现 dirty page 就写入 disk，在 Replacement 发生时就无需考虑脏数据带来的问题。

## Other Memory Pools

除了存储 tuples 和 indexes，DBMS 还需要 Memory Pools 来存储其它数据，如：

- Sorting + Join Buffers
- Query Caches
- Maintenance Buffers 
- Log Buffers
- Dictionary Caches



# Lec7 Hash Tabels

## Data Structures

DBMS 为系统内部得不同部分设置了相应的数据结构，比如：

1. **Internal Meta-Data**：用于跟踪数据库和系统状态的数据，比如 page tables，page directories；
2. **Core Data Storage**：数据库中元组的基本数据结构；
3. **Temporary Data Structures**：DBMS 可以在处理查询的过程中临时构建数据结构从而加快执行速度（例如，用于连接的哈希表）；
4. **Table Indexes**：辅助数据结构，可以用来更容易地找到特定的元组。

在为 DBMS 实现数据结构时，需要考虑两个主要的设计问题：
1. 数据组织：我们需要弄清楚内存如何布局，以及支持高效访问，里面存放什么信息。
2. 并发性：我们还需要考虑如何让多个线程访问数据结构，而不会引起问题。

## Hash Table

哈希表实现了 associative array ADT（Abstract Data Type），将键映射到值。

Hash Table 主要分为两部分：

1. Hash Function：
    - How to map a large key space into a smaller domain
    - Trade-off between **being fast** vs **collision rate**

2. Hashing Scheme：
    - How to handle key collisions after hashing
    - Trade-off between **allocating a large hash table** vs **additional instructions to find/insert keys**

## Hash Functions

由于 DBMS 内使用的 Hash Function  并不会暴露在外，因此没必要使用加密哈希函数（例如，SHA-256），我们希望它速度越快，碰撞率越低越好。

当前最先进的哈希函数是 Facebook XXHash3。

## Static Hashing Schemes

静态哈希方案中的哈希表大小固定，这意味着如果哈希表的存储空间用完，就得重建一个更大的哈希表，这样的代价非常昂贵。通常，新哈希表的大小是原始哈希表大小的 2 倍。同时，为了减少发生碰撞时比较的次数，哈希表的槽数会设置成预期元素书的 2 倍，但可惜的是，（1）元素的数量很难提前知道；（2）不同数据通过哈希函数计算得到的键可能会相等；（3）没有一个完美的哈希函数。

### Linear Probe Hashing

### Robin Hood Hashing

### Cuckoo Hashing

## Dynamic Hashing Schemes

动态哈希方案可以按需扩容缩容。

### Chained Hashing

和 Linear Probe Hashing 有点像，只不过 Linear Probe Hashing 中的哈希槽大小是固定的，而 Chained Hashing 的哈希槽大小是可变的，当新插入数据时，创建一个指针指向新数据，并将指针加入到哈希槽的末尾。

### Extendible Hashing

是 project 1 将要实现的一部分，它的基本思想是哈希值的位数慢慢用，一开始数据少时，只是用哈希值的末尾几位，当数据逐渐增多时，哈希冲突越来越剧烈，就逐渐开放使用更多的哈希值位数，同时分配更多的哈希桶来存储数据，进行 rehash 操作，达到扩容的目的。

### Linear Hashing

## 总结

哈希表提供了 O(1) 的访问效率，因此被大量地应用于 DBMS 的内部实现中。即便如此，她并不适合作为 table index 的数据结构，而 table index 的首选时下节将介绍的 B+ Tree（aka "The Greatest Data Structure of All Time"）。



# Lec8 Trees Indexes

## Table Indexes

表索引作为数据库系统内部的数据结构，常常涉及范围扫描查询。有了表索引后，DBMS 在范围查询时，就不需要进行全表扫描，而是可以选择已有的最佳表索引，更快地找到范围内的元组。

表的内容和索引在逻辑上是同步的。更多的索引可以加快查询速度，但也意味着更大的存储和维护开销。

## B+Tree

B+Tree 是一种自平衡树，它将数据有序地存储，且在 search、sequential access、insertions 以及 deletions 操作的复杂度上都满足 O(logn)。

B+Tree 可以看作是 BST (Binary Search Tree) 的衍生结构，它的每个 node 可以有多个孩子，这特别契合 disk-oriented database 的数据存储方式，每个 page 存储一个 node，使得树的结构扁平化，减少获取索引给查询带来的 I/O 成本。其基本结构如下图所示：

{% asset_img B+Tree.jpg B+Tree %}

通常来说，B+Tree 是一个 M-路 搜索树（M 代表它的最大孩子数量），它有如下性质：

1. 每个节点最多存储 M-1 个 key，有 M 个 孩子；
2. B+Tree 是完全平衡的，即每个叶子节点的深度都一样；
3. 除了 root 节点，所有其它节点至少处于半满状态，即 M/2−1≤ #keys ≤ M−1； 
4. 假设每个非叶子节点中包含 k 个 keys，那么它必然有 k+1 个孩子；
5. B+Tree 的叶子节点通过双向链表串联，访问更高效。

与 B-Tree 相比，B+Tree 仅在叶子节点上存储数据，而 B-Tree 在非叶子节点上也存储数据。

### B+ Tree Node

B+ Tree 的每个节点包含一个 K/V 键值对数组，K 是从表的 attrubute(s) 中提取出来的，即 K 的类型为表的某列的类型；V 值类型取决于节点是否是叶子节点，对于非叶子节点，V 值类型为指向节点的指针，对于叶子节点，V 值类型通常有两种做法：

1. Record/Tuple Ids：存储指向最终 tuple 的指针；
2. Tuple Data：直接将 tuple data 存在 leaf node 中，但这种方式对于 [Secondary Indexes](https://docs.oracle.com/cd/E17275_01/html/programmer_reference/am_second.html) 不适用，因为 DBMS 只能将 tuple 数据存储到一个 index 中，否则数据的存储就会出现冗余，同时带来额外的维护成本。

B+Tree 节点中的 K/V 键值对数组基本是按照 K 值排序的，尽管对于 B+Tree 的定义来说不是必需的；从概念上讲，非叶子节点上的 K 只作为标记使用，指导快速查询，但并不意味着这些 K 一定存在于叶子节点上，但是传统上，非叶子节点上的 K 都出现在叶子节点上。

### Insertion

1. 根据非叶子节点的 K 从树的根部往下找到新 K/V 键值对对应的 leaf node，L；
2. 如果 L 还有空间，则将 K/V 键值对插入到 L 中的合适位置，保证 L 中的 K/V 键值对仍是有序；否则，需要将 L 均匀分裂成两个节点，同时在 parent node 上新增 entry，新增 entry 的 K 应该是 L 中的中间 K 值，若 parent node 也空间不足，则递归地分裂，直到 root node 为止。

### Deletion

和在插入后节点满了需要拆分一样，当删除键值对后如果一个节点的大小小于半满时，需要进行合并操作。

1. 从 root 开始，找到目标 entry 所处的 leaf node, L；
2. 删除该 entry；
3. 如果 L 仍然至少处于半满状态，则操作结束；否则先尝试从 siblings 那里拆借 entries，如果失败，则将 L 与相应的 sibling 合并；
4. 如果合并发生了，则可能需要递归地删除 parent node 中的 entry。

B+Tree 的 Insert、Delete 过程，可参考[这里](https://dichchankinh.com/~galles/visualization/BPlusTree.html)。

### Selection 条件

因为 B+Tree 是有序的，所以查找很快并且不需要整个键。如果查询条件中提供了键的任何“特征”，DBMS 可以使用 B+Tree 索引。而散列索引需要具体且完整的键，因此无法进行范围查询。

### Non-Unique Indexes

与哈希表一样，B+Tree 也可以处理重复的键。比如存储有相同键的 K/V 对，或者是使用关联链表。

### Duplicate Keys

（有点分不清和 Non-Unique Indexes 的关系）

### Clustered Indexes

Clustered Indexes（聚簇索引）规定了 table 本身的物理存储方式，通常即按 primary key 排序存储，因此一个 table 只能建立一个 clustered index。有些 DBMS 对每个 table 的主键都添加聚簇索引，如果该 table 没有 primary key，则 DBMS 会为其自动生成一个。

table 本身的物理存储按照聚簇索引排序后，通过聚簇索引进行条件查询时，将减少磁盘的读取次数。

### Heap Clustering

有了聚簇索引后，元组在页面中以及页面间按照聚簇索引的规则变得有序。此时访问元组时如若使用了聚类索引（主键），DBMS 就可以直接选择正确的页面，这也就解释了上面说的为什么可以减少磁盘的读取次数。

### Index Scan Page Sorting
由于直接从非聚集索引中检索元组效率低下（从磁盘中重复读取相同页面），因此 DBMS 可以首先找出它需要的所有元组，然后根据它们的页面 ID 对它们进行排序，再一次从磁盘中读取对应的元组，从而不会重复读取相同的磁盘页面。

## B+Tree Design Choices

### Node Size

根据存储介质的不同，Node Size 的选择不一样。例如，存储在 HDD 上的节点大小通常在 MB 数量级，以减少查找数据所需的磁盘读取次数，而内存节点大小可能小至 512 字节，以便将整个页面放入 CPU 缓存并减少数据碎片化。这种选择也可以取决于工作负载的类型，因为点查询更喜欢尽可能小的页面以减少不必要的额外信息加载量，而大的顺序 scan 可能更喜欢大页面以减少它需要执行的读取次数。

### Merge Threshold

由于 merge 操作引起的修改较大，有些 DBMS 选择延迟 merge 操作的发生时间，甚至可以利用其它进程来负责周期性地重建 table index。

### Variable Length Keys

B+ Tree 中存储的 key 经常是变长的，通常有三种手段来应对：

1. Pointers：存储指向 key 的指针；
2. Variable-Length Nodes：Node 的大小可以不一致，但这需要精细化的内存管理（几乎没有人这么做）；
3. Padding：对 key 的末尾进行 pad 操作，直至 key 最大长度；
4. Key Map/Indirection：内嵌一个指针数组，数组中的每个元素指向 K/V list。

### Intra-node Search

在节点内部搜索，就是在排好序的序列中检索元素，手段通常有：

1. Linear Scan：从节点头部向尾部线性搜索；
2. Binary Search：二分查找；
3. Interpolation：通过 keys 的分布统计信息来估计大概位置进行检索

## Optimizations

### Pointer Swizzling

Node 中的 V 常常使用 page id 来指向其它 Node，这样的话 DBMS 每次需要首先从 page table 中获取对应 page 的 frame_id，然后才能从 buffer pool 获取相应的 node 本身，而如果 page 已经在 buffer pool 中，我们可以直接存储其在 buffer pool 的位置，从而避免查询 page table（常常还需要使用 latch 避免竞争），提高访问效率。

### Bulk Insert

建 B+ Tree 的最快方式是先将 keys 排好序后，再从下往上建树。因此如果有大量插入操作，可以利用这种方式提高效率。

### Prefix Compression

同一个 leaf node 中的 keys 通常有相同的 prefix，为了节省空间，可以只存所有 keys 的不同的 suffix。

### Suffix Truncation

由于非叶子节点只用于引导搜索，因此没有必要在非叶子节点中储存完整的 key，我们可以只存储足够的 prefix 并保证和原来的搜索语义一样即可。

### Deduplication

Non-unique indexes 可能在节点中存储了多个相同的 K 副本，为了减少冗余，可以替换只存储唯一 K，其中 V 则使用关联列表代替。

# Lec9 Index Concurrency Control

## Latch Implementations

1. Blocking OS Mutex：睡眠锁，当该锁被其它线程占用时，当前线程进入睡眠状态，CPU 将调度其它线程。
    - 示例：`std::mutex`；
    - 优点：使用简单，不需要 DBMS 进行额外编码；
    - 缺点：由于是操作系统进行调度，代价昂贵（每次锁定/解锁大约需要 25 ns）且不可扩展。
2. Test-and-Set Spin Latch (TAS)：自旋锁，当该锁被其它线程占用时，用户可以自行决定接下来的操作，比如可以选择重试（while 循环）或允许操作系统将当前线程睡眠，因此，该方法给了 DBMS 更多的控制权。
    - 示例：`std::atomic<T>`；
    - 优点：锁定/解锁操作是高效的（因为很多事情交给了用户来做了）；
    - 缺点：不可扩展且缓存不友好，因为 CAS（compare-and-set）指令将在不同的线程中执行多次。
3. Reader-Writer Latches：读写锁。
    - 示例：`std::shared mutex`；
    - 优点：允许并发读；
    - 缺点：DBMS 需要管理读写队列，避免大量读请求淹没写请求，同时由于额外的元数据，开销比自旋锁搞。

## Hash Table Latching

对于静态哈希表，所有线程在遇到哈希冲突时都是自顶向下查看 slot，因此很容易加锁，比如为每个 page 或者 slot 加锁，这样也不会出现死锁的情况（加锁方向一致），当需要重新调整哈希表的大小时，则获取整个哈希表的全局锁即可；

1. page 锁：每个页面都有自己的读写锁来保护其全部内容。线程在访问页面之前获取读或写锁。由于一个页面中有多个 slot，这会降低一定的并行度。
2. slot 锁：每个 slot 都有一把锁，此时两个线程可以访问同一页中的不同 slot，这增加了存储（每个 slot 有一个锁变量）和计算（一次查找可能涉及很多次加锁/解锁操作）开销。

对于动态哈希表，加锁比较麻烦，但大致思路和上面一样。

## B+Tree Latching

目标：

1. 允许多线程同时读取和更新 B+Tree；
2. 同时需要注意两种情形：（1）多个线程同时修改一个节点的内容；（2）一个线程正在遍历树然而另外一个线程正在 split/merge 节点；

举例：T1 想要删除 44，T2 想要查询 41。删除 44 后，DBMS 需要 rebalance，将 H 节点拆分成两个节点。若在拆分前，T2 读取到 D 节点，发现 41 在 H 节点，此时时间片轮转到了 T1，T1 把 D 节点拆分成 H、I 两个节点，同时把 41 转移到 I 节点，之后 CPU 交还给 T2，T2 到 H 节点就找不到 41，如下图所示。

{% asset_img B+delete.jpg B+delete %}

{% asset_img B+rebalance.jpg B+rebalance %}

{% asset_img B+find.jpg B+find %}

由于 B+Tree 是树形结构，有明确的搜索顺序，因此沿着搜索路径加锁是不错的选择。

### Latch Crabbing/Coupling

它的基本思想是

1. 获取 parent 的 latch
2. 获取 child 的 latch
3. 如果 child “安全”（不会出现 split/merge 的情况，它的 parent 节点也就不需要修改），则可以释放 parent 的 latch。

### Better Latching Algorithm

在实际应用中，每次更新 B+Tree 都需要获取非叶子节点的写锁是不好的，因为实际的更新只是发生在叶子节点上，而叶子节点大多数情况下是不会影响到上层节点的，尤其是越往上影响到的可能性越小，因此总是持 “悲观” 的想法而获取搜索路径上的的非叶子节点的写锁是不合适的。

反之，可以采用类似乐观锁的思想，假设 leaf node 是安全（更新操作仅会引起 leaf node 的变化）的，在查询路径上一路获取、释放 read latch，到达 leaf node 时，若操作不会引起 split/merge 发生，则只需要在 leaf node 上获取 write latch 然后更新数据，释放 write latch 即可；若操作会引起 split/merge 发生，则重新执行一遍，此时在查询路径上一路获取、释放 write latch，即 Latch Crabbing 原始方案。

### Leaf Node Scans

以上都是从上往下的访问模式，我们介绍了相关锁的控制；而对于水平方向的访问模式，如果两个线程分别从左往右，从右往左获取写锁，就可能陷入死锁，因此部分 DBMS 在进行范围查询时总是从左往右扫描。

# Lec10 Sorting & Aggregations Algorithms

开始进入 Operator Execution 部分。

SQL 语句被 DBMS 解释成 Query Plan。

{% asset_img query_plan.jpg query_plan %}

## Sorting

Disk-Oriented DBMS 将数据持久化在磁盘中，而不会假设整张表能够完全写入内存中，这也就可能由于数据量过大而无法在内存中完全一次性的排序操作。然而很多时候排序是需要的，比如 `ORDER BY`，`DISTINCT`，`GROUP BY` 和 `JOIN` 都将需要排好序的数据以便能够快速地执行。

对于数据量太大而无法放入内存的数据，外部归并排序是一个很好的选择。它是一种分而治之的排序算法，它将数据集分成多个独立的块，然后对它们分别进行排序，并将排序结果写入磁盘，以便下一个阶段使用。

### Two-way Merge Sort

该算法的最基本版本是二路归并排序。该算法排序阶段读取每个页面，对它们单独进行排序，并将排序后的结果写回磁盘。然后，在合并阶段，它使用三个缓冲页，其中将两个已排序的页面读取到缓冲页中，并将它们合并到第三个缓冲页面中，每当第三页填满后，它就会写回磁盘并替换为空页。

复杂度：

### General (K-way) Merge Sort

归并排序的一般版本可以使用三个以上的缓冲页，从而加快排序速度。

### Double Buffering Optimization

外部归并排序的一个优化是在后端预取下一阶段的排序块并将其存储在第二个缓冲区，而系统正在处理当前排序块。这减少了 I/O 请求的等待时间。但这种优化需要使用多线程，因为预取应该在当前运行的计算发生时发生。

### Using B+Trees

有时 DBMS 使用现有的 B+ 树索引来帮助排序。特别是，如果索引是聚簇索引，数据会按照正确的顺序存储在磁盘中，I/O访问
将是顺序的，从而 DBMS 可以只遍历B+ 树的叶子结点，不需要进行排序操作。

另一方面，如果索引是非聚簇的，遍历树几乎总是更糟，因为每条记录可以存储在任何页面中，几乎所有记录访问都需要进行磁盘读取。

## Aggregations

查询计划中的聚合运算符将一个或多个元组的值折叠为单个标量值。有两种实现聚合的方法：

1. 排序；
2. 散列。

### Sorting Aggregation

DBMS 首先根据 GROUP BY Key(s)  对元组进行排序。在内存足够时，可以使用快速排序，如果内存不够，可以使用外部归并排序算法。然后，DBMS 对排序后的数据执行顺序扫描以计算聚合。

但是，有时候我们并不需要排好序的数据，比如 `GROUP BY` 和 `DISTINCT`。在这种场景下 hashing 将更好，它能有效减少排序所需的额外工作。

### Hashing Aggregation

hash 在计算上比 sort 开销更小。 DBMS 通过扫描表填充临时哈希表，对于每条记录，检查是否已经有一个条目，并根据聚合类型执行适当的修改。如果哈希表的大小太大而无法全部装入内存时，DBMS 可以将它写入磁盘。这需要两部操作：

1. Partition： Use a hash function h1 to split tuples into partitions on disk based on target hash key。这会将所有匹配的元组放入同一分区；
2. ReHash：对于磁盘上的每个分区，将其页面读入内存并基于第二个哈希函数 h2（h1 != h2） 构建一个内存中的哈希表。然后遍历这个临时哈希表的每个桶，将匹配的元组放在一起计算聚合（这假定每个分区都能装入内存）。

在 ReHash phase 中，存着 (GroupKey→RunningVal) 的键值对，当我们需要向 hash table 中插入新的 tuple 时：

1. 如果我们发现相应的 GroupKey 已经在内存中，只需要更新 RunningVal 就可以；
2. 反之，则插入新的 GroupKey 到 RunningVal 的键值对。

复杂度：

# 推荐阅读

[官方笔记](https://15445.courses.cs.cmu.edu/fall2022/notes/)

[公开课笔记](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/relational-data-model)