---
title: Ceph源码分析
date: 2022-10-22 14:52:57
description: 《Ceph源码分析》笔记
tags:
- Ceph
categories:
- Ceph
hidden: true
---

**《Ceph源码分析》摘要**

# Ceph整体架构

## Ceph设计目标

1. 高可用性：Ceph使用数据多副本、纠删码提供数据冗余。
2. 高可扩展性
    - 集群容量可以伸缩，可以任意添加和删除存储节点、存储设备。
    - 系统性能随集群的增加而线性增加。
3. 大规模：Ceph存储系统的规模可以扩展到成千上万节点。

## Ceph基本架构图

Ceph整体架构有三层

1. 最底层最核心的 RADOS 对象存储系统：RADOS（reliable, autonomous, distributed object store）是一个可靠的、自组织的、可自动修复、自我管理的分布式对象存储系统。其内部包括 ceph-osd 后台服务进程和 ceph-mon 监控进程。  
2. 中间层librados库：用于本地或远程通过网络访问RADOS对象存储系统。支持多种语言，如C/C++、Java、Python等。
3. 最上层Ceph不同形式的存储接口实现
    - 块存储结构。
    - 对象存储接口。
    - 文件系统接口。

{% asset_img ceph基本架构图.png Ceph基本架构图 %}

## Ceph客户端接口

### RBD

RBD（rados block device）通过 librbd 库对应用提供块存储，主要面向云平台的虚拟机提供虚拟磁盘。传统 SAN 就是块存储，通过 SCSI 或者 FC 接口给应用提供一个独立的 LUN 或者卷。RBD 类似于传统的 SAN 存储，都提供数据块级别的访问。  

### CephFS

CephFS 通过在 RADOS 基础之上增加了 MDS ( Metadata Server) 来提供文件存储。它提供了 libcephfs 库和标准的 P0SIX 文件接口。 CephFS 类似于传统的 NAS 存储，通过 NFS 或者 CIFS 协议提供文件系统或者文件目录服务。

### RadosGW

（看不太明白

## RADOS

RADOS完成了一个存储系统的核心功能，包括以下部分：

### Monitor

Monitor模块为整个存储集群提供全局的配置和系统信息。

1. 它是一个独立部署的daemon进程，通过组成Monitor集群来保证自己的高可用性。
2. 通过Paxos算法实现自己数据的一致性。
3. 提供整个存储系统的节点信息等全局配置信息。

Cluster Map保存了系统的全局信息，主要包括：

1. Monitor Map
    - 包括集群的fsid；
    - 所有Monitor的地址和端口；
    - current epoch。
2. OSD Map：所有OSD的列表和OSD的状态等。
3. MDS Map：所有的MDS的列表和状态。

### 对象存储

这里所说的对象是指 RADOS 对象，要和 RadosGW 的 S3 或者 Swift 接口的对象存储区分开来。对象是数据存储的基本单元，一般默认 4MB 大小。下图是一个对象的示意图。

{% asset_img 对象示意图.png 对象示意图 %}

一个对象由三个部分组成：

1. 对象标志ID；
2. 对象的数据，其在本地文件系统中对应一个文件，对象的数据就保存在文件中；
3. 对象的元数据，以key-value形式保存。

### pool和PG的概念

pool是一个抽象的存储池。它规定了数据冗余的类型以及对应的副本分布策略（副本类型和纠删码类型等）。一个pool由多个PG构成。

PG（placement group）是一个对象的集合，该集合中的所有对象有相同的放置策略：对象的副本都分布在相同的OSD列表上。一个对象只能属于一个PG，一个PG对应于放置在其上的OSD列表。一个OSD上可以分布多个PG。

{% asset_img PG概念示意图.png PG概念示意图 %}

1. PG1和PG2都属于同一个pool，都是副本类型，且都是两副本；
2. PG1和PG2里包含许多对象，PG1 上的所有对象，主从副本分布在 0SD1 和
    0SD2 上，PG2 上的所有对象的主从副本分布在0SD2 和 OSD3 上；
3. 一个对象只能属于一个PG，一个PG包含多个对象；
4. 一个 PG 的副本分布在对应的 OSD 列表中。在一个 OSD 上可以分布多个 PG。示例中 PG1 和 PG2 的从副本都分布在 0SD2 上。

### 对象寻址过程

1. 对象到 PG 的映射：使用静态哈希算法得到对象的所属 PG。

    `pg_id = hash(object_id) % pg_num `；

2. PG 到 OSD 列表的映射：CRUSH 算法决定 PG 上对象的副本如何分布在 OSD 上。

### 数据读写过程

主 OSD 必须等待所有从 OSD 返回正确应答，才可以向客户端返回写操作正确的应答。

{% asset_img 读写过程.png 读写过程 %}

### 数据均衡

1

### Peering

1

### Recovery和Backfill

1

### 纠删码

1

### 快照和克隆

1

### Cache Tier

类 **Objecter** 负责请求是发给 **Cache** **Tier** 层，还是发给 **Storage** **Tier** 层。（这里对应的层是 SSD 或 HDD，而如果换成内存和持久层，是否需要改变）

{% asset_img cache tier 结构.png cache tier 结构 %}

### Scrub

1

### 本章小结

1

# Ceph通用模块

Ceph 源代码通用库中的一些关键而又复杂的数据结构。大多定义于`src/common/`中

1. Object 和 Buffer普遍使用；
2. 线程池 ThreadPool 可以提高消息处理的并发能力；
3. Finisher 提供异步操作时来执行回调函数；
4. Throttle 在系统的各个模块各个环节都能看到，用来限制系统的请求，避免瞬时大量请求对系统的冲击；
5. SafteTimer 提供定时器，为超时和定时任务等提供了相应的机制。

## Object

对象 Object 默认为4MB大小的数据块。一个对象对应本地文件系统（RADOS）中的一个文件。在代码实现中，有 object、sobject、hobject、ghobject 等不同的类。大多位于`src/include/object.h`中。

底层 Blob 大小为 64KB。

### object_t

object_t 对应本地文件系统中的一个文件，name 就是对象名。

```c++
struct object_t{
	string name;
	......
}
```

### file_object_t

```cpp
struct file_object_t {
  uint64_t ino, bno;
  mutable char buf[34];

  file_object_t(uint64_t i=0, uint64_t b=0) : ino(i), bno(b) {
    buf[0] = 0;
  }
  
  const char *c_str() const {
    if (!buf[0])
      snprintf(buf, sizeof(buf), "%llx.%08llx", (long long unsigned)ino, (long long unsigned)bno);
    return buf;
  }

  operator object_t() {
    return object_t(c_str());
  }
};
```

### sobject_t

sobject_t 在 object_t 之上增加了 snapshot 信息，用于标识是否是快照对象。

```C++
struct sobject_t {
	object_t oid;
	snapid_t snap; // 快照对象的对应的快照序号；若不是快照对象（也就是head对象），snap字段则为CEPH_NOSNAP值
	......
}
```

### hobject_t

hobject_t 是 hash object 的缩写。位于`src\common\hobject.h`中。

```c++
struct hobject_t {
  public:
    object_t oid;
    snapid_t snap;

  private:
    uint32_t hash; // hash和key不能同时设置，hash值一般设为pg的id值
    bool max;
    uint32_t nibblewise_key_cache;
    uint32_t hash_reverse_bits;
	......
	
  public:
    int64_t pool; // 所在pool的id
    std::string nspace; // 一般为空，用于标识特殊的对象

  private:
    std::string key; // 对象的特殊标记
    ......
}
```

### ghobject_t

在 hobject_t 基础上，添加 generation 和 shard_id 字段，用于纠删码模式下的PG。

```c++
struct ghobject_t {
    static const gen_t NO_GEN = UINT64_MAX;

    bool max = false;
    shard_id_t shard_id = shard_id_t::NO_SHARD; // 标识对象所在的OSD在纠删码类型的PG中的序号；如果在副本类型的PG中，那么字段就设置为NO_SHARD(-1)
    hobject_t hobj;
    gen_t generation = NO_GEN; // 记录对象的版本号。当PG为纠删码类型时，写操作需要区别写前后两个版本的object，此时该字段保存对象的上一个版本，当写失败时，可以rollback到上一个版本
    ......
}
```

## Buffer

buffer 是一个命名空间，里面定义了 buffer 相关的数据结构，管理 ceph 的内存。

### buffer::raw

位于`src/include/buffer_raw.h`中，是一个基类，其子类完成buffer数据空间的分配。

```c++
class ceph::buffer::v15_2_0::raw{
protected:
	char *data;  // 数据指针
	unsigned len; // 数据长度
	
public:
	ceph::atomic<unsigned> nref{0}; // 引用计数
	std::aligned_storage<sizeof(ptr_node),alignof(ptr_node)>::type bptr_storage; // 大小为sizeof(ptr_node)，alignof(ptr_node)对齐的类型。用于ptr_node的构造(new placement方式)，实际并未使用
	
    // 指定数据段的校验值
    std::pair<size_t, size_t> last_crc_offset {std::numeric_limits<size_t>::max(), std::numeric_limits<size_t>::max()};
    std::pair<uint32_t, uint32_t> last_crc_val;
	mutable ceph::spinlock crc_spinlock; 
    ......
}
```

下列类继承了`buffer::raw`，实现了data对应内存空间的申请，定义于`src/common/buffer.cc`中：

1. class raw_combined：分配的对象与 data buffer 分配在一个 buffer 上，data 处于 buffer 的开头，object 在 buffer 尾；
2. class raw_malloc：实现了用 malloc 函数分配内存空间的功能；
3. class buffer::raw_posix_aligned：调用了函数 posix_memaligii 来申请内存地址对齐的内存空间；
4. class raw_static：data buffer 使用 static buffer；
5. class buffer::raw_hack_aligned：在系统不支持内存对齐申请的情况下自己实现了内存地址的对齐；
6. class buffer::raw_char：使用了 C++ 的 new 操作符来申请内存空间； 
7. class buffer::raw_claimed_char：不负责释放资源，因此可以是局部变量。针对new创建的buffer，则需手动显示释放；
8. class buffer::raw_claim_buffer：接收自定义删除器。

### buffer::ptr

是对于 buffer::raw 的一个部分数据段，定义于`src/include/buffer.h`中。

```c++
class CEPH_BUFFER_API ptr{
    raw *_raw;
    unsigned _off, _len; // 起始偏移量，长度
    ......
}
```

{% asset_img raw和ptr示意图.png raw和ptr示意图 %}

### buffer::list

是多个buffer::ptr的列表，也就是多个内存数据段的列表，定义于`src/include/buffer.h`。

```c++
class CEPH_BUFFER_API list{
public:
	class buffers_t{ // 底层单链表实现
		ptr_hook _root;
		ptr_hook *_tail;
		......
            
        // 添加到头部
		void push_front(reference item) {
            item.next = _root.next;
            _root.next = &item;
            _tail = _tail == &_root ? &item : _tail;
        }
	}
private:
	buffers_t _buffers; // 自己的私有bits
	unsigned _len; // 所有 ptr 的数据总长度
	......
}
```

**（list结构大改，还需要仔细阅读原书和源码**

## 线程池

ThreadPool定义于 `src/common/WorkQueue.h` 中。

```c++
class ThreadPool : public md_config_obs_t {
protected:
	CephContext *cct;
  	std::string name; 						// 线程名？
  	std::string thread_name; 
  	std::string lockname; 					// 锁名
  	ceph::mutex _lock; 						// 线程互斥锁，也是工作队列访问互斥的锁
  	ceph::condition_variable _cond;			// 锁对应的条件变量
  	bool _stop; 							// 线程池是否停止工作的标志
  	int _pause; 							// 暂停中止线程池的标志
  	int _draining;
 	ceph::condition_variable _wait_cond;
    
    std::vector<WorkQueue_*> work_queues; 	// 工作队列[集和]
    int next_work_queue = 0;				// 下一次访问的工作队列
    
    std::set<WorkThread*> _threads;			// 线程池中的工作线程
    std::list<WorkThread*> _old_threads;  	// 等待进joined操作的线程
  	int processing;
    ......
}
```

一般情况下，一个工作队列对应一个类型的后台处理任务，一个线程池对应一个工作队列，专门用于处理该类型的任务。如果是后台任务，又不紧急，就可以将多个工作队列放置到一个线程池中，该线程池可以处理不同类型的任务。

线程池的实现主要包括：

1. 线程池的启动过程；
2. 线程池对应的工作队列管理；
3. 线程池对应的执行函数如何执行任务。

### 线程池的启动

`ThreadPool::start()` 用来启动线程池，其在加锁的情况下，调用函数 `start_threads`，该函数检査当前线程数，如果小于配置的线程池，就创建新的工作线程。定义于 `src/common/WorkQueue.cc`。

### 工作队列

工作队列（WorkQueue) 定义了线程池要处理的任务。

1. 任务类型在模板参数中指定；
2. 在构造函数里，就把自己加入到线程池的工作队列集合中；
3. WorkQueue 实现了部分功能：进队列、出队列、加锁；
4. 部分功能需要使用者定义，如定义保存任务的容器，添加和删除的方法，以及如何处理任务的方法。

```c++
template<class T>
class WorkQueue : public WorkQueue_ {
    ThreadPool *pool;
    
    // Add a work item to the queue.
    virtual bool _enqueue(T *) = 0;
    // Dequeue a previously submitted work item.
    virtual void _dequeue(T *) = 0;
    // Dequeue a work item and return the original submitted pointer.
    virtual T *_dequeue() = 0;
    
protected:
    // Process a work item. Called from the worker threads.
    virtual void _process(T *t, TPHandle &) = 0;
    
    WorkQueue(std::string n,
	    	ceph::timespan ti, ceph::timespan sti,
	    	ThreadPool* p)
      		: WorkQueue_(std::move(n), ti, sti), pool(p) {
      	pool->add_work_queue(this);
    }
    
    bool queue(T *item) { // 进队列
     	pool->_lock.lock();
     	bool r = _enqueue(item);
     	pool->_cond.notify_one();
     	pool->_lock.unlock();
    	return r;
    }
    void dequeue(T *item) { // 出队列
      	pool->_lock.lock();
      	_dequeue(item);
      	pool->_lock.unlock();
    }
    void clear() {
      	pool->_lock.lock();
      	_clear();
      	pool->_lock.unlock();
    }
    void lock() { pool->lock(); }
    void unlock() { pool->unlock(); }
    ......
}
```

### 线程池的执行函数

`ThreadPool::worker` 定义于 `src/common/WorkQueue.cc` 中。

1. 首先检查_stop标志，确保线程池没有关闭；
2. 调用函数 `join_old_threads` 把旧的工作线程释放掉。检査如果线程数量大于配置的数量 \_num\_threads，就把当前线程从线程集合中删除，并加入_old_threads队列中，并退出循环；
3. 如果线程池没有中止（_pause）且work_queues不为空，就从next\_work\_queue开始，边理每一个工作队列，如果工作会裂不为空，就取出一个item，调用工作队列的处理函数做处理。

### 超时检查

TPHandle是一个有意思的事情，定义于 `src/common/WorkerQueue.h` 中。每次线程函数执行时，都会设置一个 grace 超时时间，当线程执行超过该时间，就认为是 unhealthy 的状态。当执行时间超过 suicide_grace 时，OSD 就会产生断言而导致自杀，代码如下：  

```c++
class ThreadPool::TPHandle : public HBHandle {
        friend class ThreadPool;
        CephContext *cct;				
        ceph::heartbeat_handle_d *hb;	// 心跳
        ceph::timespan grace;			// 超时
        ceph::timespan suicide_grace;	// 自杀的超时时间

      public:
        TPHandle(CephContext *cct, ceph::heartbeat_handle_d *hb,
                 ceph::timespan grace, ceph::timespan suicide_grace)
            : cct(cct), hb(hb), grace(grace), suicide_grace(suicide_grace) {}
        void reset_tp_timeout() override final;
        void suspend_tp_timeout() override final;
    ......
    };
```

### ShardedThreadPool

ThreadPool 实现的线程池，其每个线程都有机会处理工作队列的任意一个任务。这就会导致一个问题：如果任务之间有互斥性，那么正在处理该任务的两个线程有一个必须等待另一个处理完成后才能处理，从而导致线程的阻塞，性能下降。

具体如何实现 Shard 方式，还需要使用者自己去实现。其基本的思想就是：每个线程对应一个任务队列，所有需要顺序执行的任务都放在同一个线程的任务队列里，全部由该线程执行。 定义于`src/common/WorkerQueue.h`中 

### Finisher

类Finisher用来完成回调函数Context的执行，其内部有一个FinisherThread线程来用于执行Context回调函数。定义于 `src/common/Finisher.h` 中。

### Throttle

类Throttle用来限制消费的资源数量（也常称为槽位 “slot”），当请求的 slot 数量达到max值时，请求就会被阻塞，直到有新的槽位释放出来，定义于`src/common/Throttle.h`中，代码如下：

```c++
class Throttle final : public ThrottleInterface {
  	CephContext *cct;
  	const std::string name;
  	PerfCountersRef logger;
  	std::atomic<int64_t> count = { 0 }, max = { 0 };  	// 当前占用的slot数量和slot数量的最大值
  	std::mutex lock;									// 等待的锁
  	std::list<std::condition_variable> conds;			// 等待的条件变量
    ......
public:
    bool get(int64_t c = 1, int64_t m = 0);				// 获取c个slot，如果m不为0值，则将max设置为m的值；成功获取c个slot后，就返回true，否则阻塞等待
    bool get_or_fail(int64_t c = 1);					// 当获取不到c个slot时，直接返回false，不阻塞等待
    int64_t put(int64_t c = 1) override;				// 释放c个slot资源
    ......
}
```

  ### SafeTimer

类SafeTimer实现了定时器的功能，定义于`src/common/Timer.h`中，代码如下：

```c++
template <class Mutex> class CommonSafeTimer {
    CephContext *cct;
    Mutex &lock;
    std::condition_variable_any cond;
    bool safe_callbacks;							// 是否是safe_callbacks

    friend class CommonSafeTimerThread<Mutex>;
    class CommonSafeTimerThread<Mutex> *thread;		// 定时器执行线程
    
    using clock_t = ceph::mono_clock;
    using scheduled_map_t = std::multimap<clock_t::time_point, Context *>;
    scheduled_map_t schedule;	// 目标时间和定时任务执行函数Context
    using event_lookup_map_t = std::map<Context *, scheduled_map_t::iterator>;
    event_lookup_map_t events;	// 定时任务<-->定时任务在schedule中的位置映射
    bool stopping;				// 是否停止
    ......
public:
    // 添加定时任务
    Context *add_event_after(ceph::timespan duration, Context *callback);
    Context *add_event_after(double seconds, Context *callback);
    Context *add_event_at(clock_t::time_point when, Context *callback);
    Context *add_event_at(ceph::real_clock::time_point when, Context *callback);
    // 取消定时任务
    bool cancel_event(Context *callback);
    void cancel_all_events();
    // 定时任务执行
    /* 循环检查schedule中的任务是否到期，由于schedule中是按照时间升序排列的，因此第一任务没有到期就终止循环；
       如果第一任务到期，则调用callback执行，需要注意如果是非safe_callbacks，则需要先获取lock再执行callback函数。*/
    void timer_thread(); 
    ......
}
using SafeTimer = class CommonSafeTimer<ceph::mutex>;
```

# Ceph网络通信

本章介绍 Ceph 网络通信模块，这是客户端和服务器通信的底层模块，用来在客户端和服务器之间接收和发送请求。其实现功能比较清晰，是一个相对较独立的模块理解起来比较容易，所以首先介绍它。  

## Ceph网络通信框架

—个分布式存储系统需要一个稳定的底层网络通信模块，用于各节点之间的互联互通。对于一个网络通信系统，要求如下：

1. 高性能：带宽和延时；
2. 稳定可靠：数据不丢包，在网络中断时，实现重连等异常处理。

网络通信模块的实现在源代码`src/msg`的目录下，其首先定义了一个网络通信的框架，三个子目录里分别对应：Simple、Async、XIO三种不同的实现方式。

Simple 是比较简单，目前比较稳定的实现，系统默认的用于生产环境的方式。它最大的特点是：每一个网络链接，都会创建两个线程，一个专门用于接收，一个专门用于发送。这种模式实现比较简单，但是对于大规模的集群部署，大量的链接会产生大量的线程，会消耗 CPU 资源，影响性能。   

Async 模式使用了基于事件的 I/O 多路复用模式。这是目前网络通信中广泛采用的方式， 但是在 Ceph 中，官方宣称这种方式还处于试验阶段，不够稳定，还不能用于生产环境（📕中的Ceph是10.2.1老版本，**新版本中似乎只有Async模式了**）。

XIO 方式使用了开源的网络通信库 accelio 来实现。这种方式需要依赖第三方的库 accelio 稳定性， 需要对 accelio 的使用方式以及代码实现都比较熟悉。目前也处于试验阶段。特别注意的是， 前两种方式只支持 TCP/IP 协议，XIO 可以支持 Infiniband 网络。

在 msg 目录下定义了网络通信的抽象框架，它完成了通信接口和具体实现的分离。在其下分别有 `msg/simple` 子目录、 `msg/Async` 子目录、 `msg/xio` 子目录，分别对应三种不同的实现。 

### Message

类Message是所有消息的基类，任何要发送的消息，都要继承该类，定义于`src/msg/Message.h`中，格式如下图所示。

{% asset_img 消息发送格式.png 消息发送格式 %}

其中，header是消息头，类似一个消息的信封（envelope），user\_data是用于要发送的实际数据，footer是一个消息的结束标记，如下所示：

```c++
class Message : public RefCountedObject {
protected:
    ceph_msg_header header; 	// 消息头
    ceph_msg_footer footer;		// 消息尾
    ceph::buffer::list payload; // "front" unaligned blob
    ceph::buffer::list middle;  // "middle" unaligned blob
    ceph::buffer::list data;	// data payload (page-alignment will be preserved where possible)

    // 消息相关的时间戳
    utime_t recv_stamp;			// 开始接收数据的时间戳
    utime_t dispatch_stamp;		// dispatch的时间戳
    utime_t throttle_stamp;		// 获取throttle的slot的时间戳
    utime_t recv_complete_stamp;// 接收完成的时间戳

    ConnectionFRef connection;	// 网络连接类

    uint32_t magic = 0;			// 消息的魔术字
	
    boost::intrusive::list_member_hook<> dispatch_q; //boost::intrusive需要的字段
    ......
}
```

下面分别介绍其中的重要参数。

ceph_msg_header 为消息头，它定义了消息传输相关的元数据，在`src/include/msgr.h`中：

```c++
struct ceph_msg_header {
	__le64 seq;       		// 当前session内消息的唯一序号
	__le64 tid;       		// 消息的全局唯一id
	__le16 type;      		// 消息类型
	__le16 priority;  		// 优先级
	__le16 version;   		// 消息编码的版本

	__le32 front_len;		// payload的长度
	__le32 middle_len;		// middle的长度
	__le32 data_len;  		// data的长度
	__le16 data_off;  		// 对象的数据偏移量

	struct ceph_entity_name src; 	// 消息源

	// 一些旧的代码，用于兼容，如果为0就忽略
	__le16 compat_version;
	__le16 reserved;
	__le32 crc;       		// 消息头的crc32c校验信息
} __attribute__ ((packed));
```

ceph_msg_footer 为消息的尾部， 附加了一些 crc 校验数据和消息结束标志：

```c++
struct ceph_msg_footer {
	__le32 front_crc, middle_crc, data_crc;
                                // 分别对应crc校验码
	__le64  sig;				// 消息的64为signature
	__u8 flags;					// 结束标志
} __attribute__ ((packed));
```

消息带的数据分别保存在 payload、 middle、 data 这三个 bufferlist 中。 payload —般保
存 Ceph 操作相关的元数据， middle 目前没有使用到， data—般为读写的数据。

### Connection

类Connection对应端（port）对端的socket链接的封装。定义于`src/msg/Connection.h`中，其最重要的接口是可以发送消息：

```c++
struct Connection : public RefCountedObjectSafe {
    mutable ceph::mutex lock = ceph::make_mutex("Connection::lock"); // 锁保护Connnection的所有字段
    Messenger *msgr;								
    RefCountedPtr priv;								// 链接的私有数据
    int peer_type = -1;								// 链接的peer类型
    int64_t peer_id = -1; 							// [msgr2 only] the 0 of osd.0, 4567 or client.4567
    safe_item_history<entity_addrvec_t> peer_addrs; // peer的地址（和📕中的类型不一样）
    utime_t last_keepalive, last_keepalive_ack;		// 最后一次发送keepalive和最后一次接收keepalive的ACK的时间
    bool anon = false; 								// < anonymous outgoing connection
private:
    uint64_t features = 0;							// 一些feature的标志位

public:
    bool is_loopback = false;
    bool failed = false; 							// 当值为true时，该链接为lossy链接已经失效了

    int rx_buffers_version = 0;						// 接收缓冲区的版本
    std::map<ceph_tid_t, std::pair<ceph::buffer::list, int>> rx_buffers; // 接收缓冲区
    												// 消息的标识ceph_tid --> (buffer, rx_buffers_version)的映射
    ......
    virtual int send_message(Message *m) = 0;		// 最重要的功能是发消息的接口
}
```

### Dispatcher

类Dispatcher是消息分发的接口，定义于`src/msg/Dispatcher.h`中，其分发消息的接口为：

```c++
virtual bool ms_dispatch(Message *m) { ceph_abort(); }
virtual void ms_fast_dispatch(Message *m) { ceph_abort(); }
```

### Messenger

Messenger 是整个网络抽象模块，定义了网络模块的基本 API 接口。网络模块对外提供的基本功能，就是能在节点之间发送和接受消息。Messenger定义于`src/msg/Messenger.h`中。

向一个节点发送消息的命令如下：

```c++
virtual int send_to(Message *m, int type, const entity_addrvec_t &addr) = 0;
```

注册一个Dispatcher用来分发消息的命令如下：

```c++
void add_dispatcher_head(Dispatcher *d) {
        bool first = dispatchers.empty();
        dispatchers.push_front(d);
        if (d->ms_can_fast_dispatch_any())
            fast_dispatchers.push_front(d);
        if (first)
            ready();
}
```

### 网络连接的策略

Policy定义了Messenger处理Connection的一些策略，位于`/src/msg/Policy.h`中：

```c++
/**
 * A Policy describes the rules of a Connection. Is there a limit on how
 * much data this Connection can have locally? When the underlying connection
 * experiences an error, does the Connection disappear? Can this Messenger
 * re-establish the underlying connection?
 */
template <class ThrottleType> struct Policy {
    bool lossy; 	// 如果为 true, 当该连接出现错误时就删除
    bool server; 	// 如果为 true, 为服务端，都是被动连接
    bool standby;	// 如果为 true, 该连接处于等待状态
    bool resetcheck;// 如果为 true, 该连接出错后重连

    /// Server: register lossy client connections.
    bool register_lossy_clients = true;
    // The net result of this is that a given client can only have one
    // open connection with the server.  If a new connection is made,
    // the old (registered) one is closed by the messenger during the accept
    // process.

    /**
     *  The throttler is used to limit how much data is held by Messages from
     *  the associated Connection(s). When reading in a new Message, the
     * Messenger will call throttler->throttle() for the size of the new
     * Message.
     */
    // 该 connection 相关的流控操作
    ThrottleType *throttler_bytes;
    ThrottleType *throttler_messages;
    
    // 本地端的一些feature标志
    static constexpr uint64_t features_supported{CEPH_FEATURES_SUPPORTED_DEFAULT};
    // 远程端需要的一些 feature 标志
	uint64_t features_required;
    ......
    }
```

### 网络模块的使用

通过下面最基本的服务器和客户端的实例程序，了解如何调用网络通信模块提供的接口来完成收发请求消息的功能。相关代码位于`/src/test/msgr/test_msgr.cc`中。

1. Server程序分析：

    - 调用Messenger的函数create创建一个Messenger实例：

        ```c++
        Messenger *server_msgr2 = Messenger::create(g_ceph_context, string(GetParam()), entity_name_t::OSD(0), "server", getpid());
        ```

    - 设置Messenger属性：

        ```c++
        server_msgr2->set_auth_client(&dummy_auth);
        server_msgr2->set_auth_server(&dummy_auth);
        ```

    - 对于server，需要bind服务端地址：

        ```c++
        bind_addr.parse("v2:127.0.0.1:16801");
        server_msgr2->bind(bind_addr);
        ```

    - 创建Dispatcher，并添加到Messenger：

        ```c++
        MarkdownDispatcher srv_dispatcher(true);
        server_msgr2->add_dispatcher_head(&srv_dispatcher);
        ```

    - 启动Messenger：

        ```c++
        server_msgr2->start();
        server_msgr2->wait();  // 📕中说本函数必须等start完成才能调用(源代码似乎是等待1000s)
        ```

2. Client程序分析

    - 调用Messenger的函数create创建一个Messenger实例：

        ```c++
        client_msgr = Messenger::create(g_ceph_context, string(GetParam()),
                                        entity_name_t::CLIENT(-1), "client", getpid());
        ```

    - 设置相关策略：

        ```c++
        client_msgr->set_default_policy(Messenger::Policy::lossy_client(0));
        client_msgr->set_auth_client(&dummy_auth);
        client_msgr->set_auth_server(&dummy_auth);
        ```

    - 创建Dispatcher并添加，用于接收消息：

        ```c++
        MarkdownDispatcher cli_dispatcher(false);
        client_msgr->add_dispatcher_head(&cli_dispatcher);
        ```

    - 启动消息：

        ```c++
        client_msgr->start();
        ```

    - 下面开始发送请求，先获取目标Server的链接：

        ```c++
        ConnectionRef conn2 = client_msgr->connect_to(
                    server_msgr2->get_mytype(), server_msgr2->get_myaddrs());
        ```

    - 通过 Connection 来发送请求消息 ：

        ```c++
        m = new MPing();
        ASSERT_EQ(conn2->send_message(m), 0);
        ```

## Async实现

📕中讲述的是Simple实现，但现在Async网络通信模型成为了Ceph默认的通信方式。参考了[bandaoyu的博客](https://blog.csdn.net/bandaoyu/article/details/111962161)。

### 基本类

1. NetHandler：NetHandler封装了Socket的基本功能。定义于`src/msg/async/net_handler.h`中。

    ```c++
    namespace ceph {
    class NetHandler {
        int generic_connect(const entity_addr_t &addr,
                            const entity_addr_t &bind_addr, bool nonblock);
    
        CephContext *cct;
    
      public:
        int create_socket(int domain, bool reuse_addr = false); // 创建socket
        explicit NetHandler(CephContext *c) : cct(c) {}
        int set_nonblock(int sd);								// 设置socket为非阻塞
        int set_socket_options(int sd, bool nodelay, int size); // 设置socket的选项：nodelay, buffer size
        int connect(const entity_addr_t &addr, const entity_addr_t &bind_addr);
    
        /**
         * Try to reconnect the socket.
         *
         * @return    0         success
         *            > 0       just break, and wait for event
         *            < 0       need to goto fail
         */
        int reconnect(const entity_addr_t &addr, int sd);
        int nonblock_connect(const entity_addr_t &addr, 		// 非阻塞connect
                             const entity_addr_t &bind_addr);
        void set_priority(int sd, int priority, int domain);	// 设置优先级
    };
    } // namespace ceph
    ```

2. Worker：Worker类是工作线程的抽象接口，同时添加了listen和connect接口用于服务端和客户端的网络处理。其内部创建一个EventCenter类，该类保存相关处理的事件。定义于`src/msg/async/Stack.h`中。

    ```c++
    class Worker {
    	std::atomic_uint references;
        EventCenter center;	// 事件处理中心，处理该center的所有的事件
        
        // server端
        virtual int listen(entity_addr_t &addr, unsigned addr_slot,
                           const SocketOptions &opts, ServerSocket *) = 0;
        // client主动连接
        virtual int connect(const entity_addr_t &addr, const SocketOptions &opts,
                            ConnectedSocket *socket) = 0;
        ......
    };
    ```

3. PosixWorker：PosixWorker实现了Worker接口，定义于`src/msg/async/PosixStack.h`中。

    ```	c++
    class PosixWorker : public Worker {
        ceph::NetHandler net;
        void initialize() override;
    
      public:
        PosixWorker(CephContext *c, unsigned i) : Worker(c, i), net(c) {}
        // 实现了Server端的sock功能：
        // 底层调用NetHandler的功能，实现了socket的bind，listen等操作，最后返回ServerSocket对象
        int listen(entity_addr_t &sa, unsigned addr_slot, const SocketOptions &opt,
                   ServerSocket *socks) override;
        // 实现了主动连接请求。返回ConnectedSocket对象。
        int connect(const entity_addr_t &addr, const SocketOptions &opts,
                    ConnectedSocket *socket) override;
        ......
    };
    ```

4. NetworkStack：是网络协议栈的接口，定义于`src/msg/async/Stack.h`中。

    ```c++
    class NetworkStack {
        ceph::spinlock pool_spin;		
        bool started = false;
    protected:
        CephContext *cct;
        std::vector<Worker *> workers;	// worker工作队列
        ......
    };
    ```

5. PosixNetwork：实现了linux的tcp/ip协议接口，定义于`src/msg/async/PosixStack.h`中。DPDKStack实现了DPDK的接口，RDMAStack实现了IB的接口。

    ```C++
    class PosixNetworkStack : public NetworkStack {
        std::vector<std::thread> threads;	// 线程池
    
        virtual Worker *create_worker(CephContext *c, unsigned worker_id) override {
            return new PosixWorker(c, worker_id);
        }
    
      public:
        explicit PosixNetworkStack(CephContext *c);
    
        void spawn_worker(std::function<void()> &&func) override {
            threads.emplace_back(std::move(func));
        }
        void join_worker(unsigned i) override {
            ceph_assert(threads.size() > i && threads[i].joinable());
            threads[i].join();
        }
    };
    ```

    Worker可以理解为工作者线程，其对应一个thread线程。为了兼容其它协议的设计，对应线程定义在了PosixNetworkStack类里。

    通过上述分析可知，一个Worker对应一个线程，同时对应一个 事件处理中心EventCenter类。

6. EventDriver：是一个抽象接口，定义了添加事件监听，删除事件监听，获取触发的事件的接口。定义于`src/msg/async/Event.h`中。

    ```c++
    /*
     * EventDriver is a wrap of event mechanisms depends on different OS.
     * For example, Linux will use epoll(2), BSD will use kqueue(2) and select will
     * be used for worst condition.
     */
    class EventDriver {
      public:
        virtual ~EventDriver() {} // we want a virtual destructor!!!
        virtual int init(EventCenter *center, int nevent) = 0;
        virtual int add_event(int fd, int cur_mask, int mask) = 0;
        virtual int del_event(int fd, int cur_mask, int del_mask) = 0;
        virtual int event_wait(std::vector<FiredFileEvent> &fired_events,
                               struct timeval *tp) = 0;
        virtual int resize_events(int newsize) = 0;
        virtual bool need_wakeup() { return true; }
    };
    ```

    针对不同的IO多路复用机制，实现了不同的类。SelectDriver实现了select的方式；EpollDriver实现了epoll的网络事件处理方式；KqueueDriver是FreeBSD实现kqueue事件处理模型。

7. EventCenter：主要保存事件（包括fileevent，timeevent和外部事件）和处理实践的相关函数。定义于`src/msg/async/Event.h`中。

    ```
    
    ```

    # CRUSH 数据分布算法

    相比一致性哈希的优势：

    1. 新增一个 OSD，只会有部分其它 OSD 上的数据迁移至新增盘；减少一个 OSD，只有该 OSD 上的数据会迁移至其他盘；
    2. 保存多副本时，一致性哈希只能返回一个副本，而 CRUSH 可以获取多个副本。

    ## 例子

    {% asset_img crush例子1.png crush例子1 %}

    {% asset_img crush例子2.png crush例子2 %}

    {% asset_img crush例子3.png crush例子3 %}

    {% asset_img crush例子4.png crush例子4 %}

    ## 数据结构

    位于 `src/crush/crush.h` 中。

    ### crush_map

    结构 crush_map 定义了静态的所有 Cluster Map 的 bucket。bucket 为动态申请的二维数组，保存了所有的 bucket 结构。rules 定义了所有的 crush_rule 结构：

    ```cpp
    /** @ingroup API
     *
     * A crush map define a hierarchy of crush_bucket that end with leaves
     * (buckets and leaves are called items) and a set of crush_rule to
     * map an integer to items with the crush_do_rule() function.
     *
     */
    struct crush_map {
            /*! An array of crush_bucket pointers of size __max_buckets__.
             * An element of the array may be NULL if the bucket was removed with
             * crush_remove_bucket(). The buckets must be added with crush_add_bucket().
             * The bucket found at __buckets[i]__ must have a crush_bucket.id == -1-i.
             */
    	struct crush_bucket **buckets;
            /*! An array of crush_rule pointers of size __max_rules__.
             * An element of the array may be NULL if the rule was removed (there is
             * no API to do so but there may be one in the future). The rules must be added
             * with crush_add_rule().
             */
    	struct crush_rule **rules;
    	......
    };
    ```

    ### crush_bucket

    结构 crush_bucket 用于保存 Bucket 相关的信息。

    ```cpp
    /** @ingroup API
     *
     * A bucket contains __size__ __items__ which are either positive
     * numbers or negative numbers that reference other buckets and is
     * uniquely identified with __id__ which is a negative number.  The
     * __weight__ of a bucket is the cumulative weight of all its
     * children.  A bucket is assigned a ::crush_algorithm that is used by
     * crush_do_rule() to draw an item depending on its weight.  A bucket
     * can be assigned a strictly positive (> 0) __type__ defined by the
     * caller. The __type__ can be used by crush_do_rule(), when it is
     * given as an argument of a rule step.
     *
     * A pointer to crush_bucket can safely be cast into the following
     * structure, depending on the value of __alg__:
     *
     * - __alg__ == ::CRUSH_BUCKET_UNIFORM cast to crush_bucket_uniform
     * - __alg__ == ::CRUSH_BUCKET_LIST cast to crush_bucket_list
     * - __alg__ == ::CRUSH_BUCKET_STRAW2 cast to crush_bucket_straw2
     *
     * The weight of each item depends on the algorithm and the
     * information about it is available in the corresponding structure
     * (crush_bucket_uniform, crush_bucket_list or crush_bucket_straw2).
     *
     * See crush_map for more information on how __id__ is used
     * to reference the bucket.
     */
    struct crush_bucket {
    	__s32 id;        /* bucket 的 id，一般为负值 */
    	__u16 type;      /* 类型，如果是0，就是 OSD */
    	__u8 alg;        /* bucket 的选择算法 */
            /*! @cond INTERNAL */
    	__u8 hash;       /* bucket 的 hash 函数 */
    	/*! @endcond */
    	__u32 weight;    /* bucket 的权重 */
    	__u32 size;      /* bucket 下的 item 的数量 */
        __s32 *items;    // 子 bucket 在 crush__bucket 结构 buckets 数组的下标，
        				 // 这里特别要注意的是，其子 item 的 curshjbucket 结构体都
        				 // 统一保存在 crush map 结构中的 buckets 数组中 ，这里只保存其在数组中的下标
    };
    
    ```

    ### crush_rule

    ```cpp
    struct crush_rule {
    	__u32 len;								// steps 的数组的长度
    	__u8 __unused_was_rule_mask_ruleset;    // ruleset 的编号
    	__u8 type;								// 类型
    	__u8 deprecated_min_size;				// 最小 size
    	__u8 deprecated_max_size;				// 最大 size
    	struct crush_rule_step steps[0];  		// 操作步
    };
    
    /*
     * CRUSH uses user-defined "rules" to describe how inputs should be
     * mapped to devices.  A rule consists of sequence of steps to perform
     * to generate the set of output devices.
     */
    struct crush_rule_step {
    	__u32 op;      // step 操作步的操作码
        
    	__s32 arg1;    // 如果是 take, 参数就是选择的 bucket 的 id 号
        			   // 如果是 select, 就是选择的数量
    	__s32 arg2;    // 如果是 select, 是选择的类型
    };
    
    ```

    ## 算法

    ### crush_do_rule

    函数 `crush_do_rule` 根据 step 的数量，循环调用相关的函数选择 bucket。如果是深度优先，就调用函数 `crush_choose_firstn`（多副本模式），如果是广度优先，就调用函数 `crush_choose_indep` （纠删码模式）来选择。

    ### cursh_choose_firstn

    函数调用 `crush_bucket_choose` 选择需要的副本数，并对选择出来的 OSD 做了相关的冲突检査，如果冲突或者失效或者过载，继续选择新的 OSD。

    ### bucket 算法

    Bucket 随机选择算法解决了如何从 Bucket 中选择出需要的子 item 问题。

    {% asset_img bucket选择算法.png bucket选择算法 %}

    # Ceph 客户端

    ## Librados

    Librados 是 RADOS 对象存储系统访问的接口库，它提供了 pool 的创建、删除、对象的创建、删除、读写等基本操作接口。

    {% asset_img librados架构图.png librados架构图 %}

    在最上层是类 RadosClient，它是 Librados 的核心管理类，处理整个 RADOS 系统层面以及 pool 层面的管理。类 Ioctxlmpl 实现单个 pool 层的对象读写等操作。OSDC 模块实现了请求的封装和通过网络模块发送请求的逻辑，其核心类 Objecter 完成对象的地址计算、消息的发送等工作。

    ### RadosClient

    位于 `src/librados/RadosClient.h` 中。

    ### IoCtxImpl

    位于 `src/librados/IoCtxImpl.h` 中。

    一个 pool 对应一个 IoCtxImpl 对象，可以在该 pool 里创建，删除对象，完成对象数据读写等各种操作。

    1. 将请求封装成 `ObjectOperation` 类；
    2. 再添加 pool 的地址信息，封装成 `Objecter::Op` 对象；
    3. 调用函数 `objecter->op_submit` 发送给相应的 OSD，当操作完成后，调用相应的回调函数通知。

    ## OSDC

    OSDC 是客户端比较底层的模块，其核心在于封装操作数据，**计算对象的地址**，发送请求和处理超时。

    ### ObjectOperation

    位于 `src/osdc/Objecter.h` 中。

    ```cpp
    struct ObjectOperation {
      osdc_opvec ops;    // 多个操作
      int flags = 0;     // 操作的标志
      int priority = 0;	 // 优先级
    
      boost::container::small_vector<ceph::buffer::list*, osdc_opvec_len> out_bl;  // 每个操作对应的输出缓存区队列
    
      boost::container::small_vector<fu2::unique_function<void(boost::system::error_code, int,
    			      const ceph::buffer::list& bl) &&>, osdc_opvec_len> out_handler; // 每个操作对应的回调函数队列
      boost::container::small_vector<int*, osdc_opvec_len> out_rval;	// 每个操作对应的操作结果队列
      boost::container::small_vector<boost::system::error_code*, osdc_opvec_len> out_ec;  // 每个操作对应的操作错误码
    }
    ```

    每个操作是一个 OSDOp 类型，定义于 `src/osd/osd_types.h` 中。

    ```cpp
    struct OSDOp {
      ceph_osd_op op;						// 每个 ceph_osd_op 封装一个操作的操作码和相关的输人和输出参数
      sobject_t soid;						// 操作对象
    
      ceph::buffer::list indata, outdata;	// 输入和输出的 bufferlist
      errorcode32_t rval = 0;				// 操作结果
    }
    ```

    ### op_target

    封装在 `Objecter` 类内部。

    结构 op_target 封装了对象所在的 PG，以及 PG 对应的 OSD 列表等地址信息:

    ```cpp
    struct op_target_t {
        int flags = 0;				// 标志
    
        epoch_t epoch = 0;  		///< latest epoch we calculated the mapping
    
        object_t base_oid;			// 操作的对象
        object_locator_t base_oloc;	// 对象的 pool 信息
        object_t target_oid;		// 最终操作的目标对象
        object_locator_t target_oloc; // 最终目标对象的 pool 信息
        // 在这里因为由 cache tier 的存在，导致最终操作的目标和 pool 不同
     	......   
    }
    ```

    ### Op

    封装在 `Objecter` 类内部。

    结构 Op 封装了完成一个操作的相关上下文信息，包括 target 地址信息、链接信息等：

    ```cpp
    struct Op : public RefCountedObject {
        OSDSession *session = nullptr;	// OSD 相关 session 信息
        int incarnation = 0;			// 引用次数
    
        op_target_t target;				// 地址信息
    
        ConnectionRef con = nullptr;  // for rx buffer only
        uint64_t features = CEPH_FEATURES_SUPPORTED_DEFAULT; // explicitly specified op features
    
        osdc_opvec ops;					// 对应多个操作的封装
    
        snapid_t snapid = CEPH_NOSNAP;	// 快照 id
        SnapContext snapc;				// pool 层级的快照信息
        ceph::real_time mtime;
    
        ceph::buffer::list *outbl = nullptr;	// 输出的 bufferlist 
        boost::container::small_vector<ceph::buffer::list*, osdc_opvec_len> out_bl; // 每个操作对应的 bufferlist
        boost::container::small_vector<
          fu2::unique_function<void(boost::system::error_code, int,
    				const ceph::buffer::list& bl) &&>,
          osdc_opvec_len> out_handler;		// 每个操作对应的回调函数
        boost::container::small_vector<int*, osdc_opvec_len> out_rval; // 每个操作对应的输出结果
        boost::container::small_vector<boost::system::error_code*,
    				   osdc_opvec_len> out_ec;
    }
    ```

    ### Striper

    先不管。

    ### ObjectCacher

    位于 `src/osdc/ObjectCacher.h` 中。

    **有机会在这块修改缓存策略，同时关注一下 `src/client/Client.cc` 中的 `Client::_read_async`**。

    ## 客户端写操作

    结合原书和代码看。

    IoCtxImpl 调用函数 `objecter->op_submit(Op *op)` 发送给相应的 OSD 步骤中，非常关键的是：**使用 `_calc_target` 函数计算对象的目标 OSD**。

    1. 根据 t->base_oloc.pool 的 pool 信息，获取 pg_pool_t 对象；
    2. 检査 cache tier，如果是读操作，并且有读缓存，就设置 target_oloc.pool 为该 pool 的 read tier；如果是写操作类似；
    3. 调用函数 osdmap->object_locator_to_pg 获取目标对象所在的 PG；
    4. 调用函数 osdmap->pg_to_up_acting_osds，通过 CRUSH 算分，获取该 PG 对应的OSD 列表；
    5. 如果是写操作，target 的 OSD 就设置为主 OSD; 如果是读操作，如果设置了 CEPH_OSD_FLAG_BALANCE_READS 标志，就随机选择一个副本读取。如果设置了 CEPH_OSD_FLAG_LOCALIZE_READS 标志，就尽可能选择本地副本读取。

    ## Cls

    Cls 是 Ceph 的一个模块扩展，它允许用户自定义对象的操作接口和实现方法，为用户提供了一种比较方便的接口扩展方式。目前 rbd 和 lock 等模块都使用了这种机制。

    # Ceph 数据读写

    ## OSD 相关数据结构

    ### pool

    位于 `src/osd/osd_types.h` 中。

    ```cpp
    struct pg_pool_t{
      enum {
        TYPE_REPLICATED = 1,     // replication
        //TYPE_RAID4 = 2,   // raid4 (never implemented)
        TYPE_ERASURE = 3,      // erasure-coded
      };   
      utime_t create_time;
      uint64_t flags = 0;           ///< FLAG_*
      __u8 type = 0;                /// replication 或 ErasureCode
      __u8 size = 0, min_size = 0;  /// 副本数目和最少有效副本数目
      __u8 crush_rule = 0;          /// Pool 对应的 crush 规则号
      __u8 object_hash = 0;         /// 通过对象名映射到 PG 的 hash 函数。
      pg_autoscale_mode_t pg_autoscale_mode = pg_autoscale_mode_t::UNKNOWN;
    
    private:
      __u32 pg_num = 0, pgp_num = 0;  /// Pool 里 PG 的数量 和 PGP 的值。
      ......
    }
    ```

    ### PG

    位于 `src/osd/osd_types.h` 中。

    pg_t 只是一个 PG 的静态描述信息，

    ```cpp
    struct pg_t {
      uint64_t m_pool; // pg 所在的 pool
      uint32_t m_seed; // pg 的序号
    }
    ```

    spg_t 在 pg_t 的基础上，加了一个 shard_id 字段，代表了该 PG 所在的 OSD 在对应的 OSD 列表中的序号。

    ```cpp
    struct spg_t {
      pg_t pgid;
      shard_id_t shard;
    }
    ```

    ### OSDMap

    位于 `src/osd/OSDMap.h` 中。

    定义了 ceph 整个集群的全局信息，它由 Monitor 实现管理，并以全量或者增量的方式向整个集群扩散。每一个 epoch 对应的 OSDMap 都需要持久化保存在 meta 下对应对象的 omap 属性中。

    ```cpp
    class OSDMap {
    
    private:
      // 系统相关信息
      uuid_d fsid;
      epoch_t epoch;        // what epoch of the osd cluster descriptor is this
      utime_t created, modified; // epoch start time
      int32_t pool_max;     // the largest pool num, ever
      uint32_t flags;
    
      // OSD 相关信息
      int num_osd;         // not saved; see calc_num_osds
      int num_up_osd;      // not saved; see calc_num_osds
      int num_in_osd;      // not saved; see calc_num_osds
      int32_t max_osd;
      std::vector<uint32_t> osd_state;
      std::shared_ptr<addrs_s> osd_addrs;
      mempool::osdmap::vector<__u32>   osd_weight;   // 16.16 fixed point, 0x10000 = "in", 0 = "out"
      mempool::osdmap::vector<osd_info_t> osd_info;
      std::shared_ptr< mempool::osdmap::vector<uuid_d> > osd_uuid;
      mempool::osdmap::vector<osd_xinfo_t> osd_xinfo;
        
      entity_addrvec_t _blank_addrvec;
    
      // PG 相关信息
      std::shared_ptr<PGTempMap> pg_temp;  // temp pg mapping (e.g. while we rebuild)
      std::shared_ptr< mempool::osdmap::map<pg_t,int32_t > > primary_temp;  // temp primary mapping (e.g. while we rebuild)
      std::shared_ptr< mempool::osdmap::vector<__u32> > osd_primary_affinity; ///< 16.16 fixed point, 0x10000 = baseline
    
      // pool 相关信息
      mempool::osdmap::map<int64_t,pg_pool_t> pools;
      mempool::osdmap::map<int64_t,std::string> pool_name;
      mempool::osdmap::map<std::string, std::map<std::string,std::string>> erasure_code_profiles;
      mempool::osdmap::map<std::string,int64_t, std::less<>> name_pool;
    
      // crush 相关信息
      std::shared_ptr<CrushWrapper> crush;       // hierarchical map
      ......
    }
    ```

    ### object_info_t

    位于 `src/osd/osd_types.h` 中。

    结构 object_info_t 保存了一个对象的元数据信息和访问信息。其做为对象的一个属性，持久化保存在对象 xattr 中，对应的 key 为  OI_ATTR (”_”），value 就是 oject_info_t 的 encode 后的数据。

    ```cpp
    struct object_info_t {
      hobject_t soid; // 对应的对象
      eversion_t version, prior_version; // 对象的当前版本，前一个版本
      version_t user_version; // 用户操作的版本
      osd_reqid_t last_reqid; // 最后请求的请求 id
    
      uint64_t size; // 对象的大小
      utime_t mtime; // 修改的时间
      utime_t local_mtime; // 修改的本地时间
        
      // note: these are currently encoded into a total 16 bits; see
      // encode()/decode() for the weirdness.
      typedef enum {
        FLAG_LOST        = 1<<0,
        FLAG_WHITEOUT    = 1<<1, // object logically does not exist
        FLAG_DIRTY       = 1<<2, // object has been modified since last flushed or undirtied
        FLAG_OMAP        = 1<<3, // has (or may have) some/any omap data
        FLAG_DATA_DIGEST = 1<<4, // has data crc
        FLAG_OMAP_DIGEST = 1<<5, // has omap crc
        FLAG_CACHE_PIN   = 1<<6, // pin the object in cache tier
        FLAG_MANIFEST    = 1<<7, // has manifest
        FLAG_USES_TMAP   = 1<<8, // deprecated; no longer used
        FLAG_REDIRECT_HAS_REFERENCE = 1<<9, // has reference
      } flag_t;
    
      flag_t flags;
        
      uint64_t truncate_seq, truncate_size;
      std::map<std::pair<uint64_t, entity_name_t>, watch_info_t> watchers;
    
      // opportunistic checksums; may or may not be present
      __u32 data_digest;  ///< data crc32c
      __u32 omap_digest;  ///< omap crc32c
      ......
    }
    ```

    ### ObjectState

    位于 `src/osd/object_state.h` 中。

    在 object_info_t 的基础上，添加一个 exists 字段，用来标记对象是否存在。object_info_t 可能是从缓存的 attrs[OI_ATTR] 中获取的，并不能确定对象是否存在。

    ```cpp
    struct ObjectState {
      object_info_t oi;
      bool exists;         ///< the stored object exists (i.e., we will remember the object_info_t)
    }
    ```

    ### SnapSetContext

    位于 `src/osd/osd_internal_types.h` 中。

    SnapSetContext 保存了快照的相关信息，即 SnapSet 的上下文信息。

    ```cpp
    struct SnapSetContext {
      hobject_t oid;   // 对象
      SnapSet snapset; // SnapSet 对象快照相关的记录
      int ref;		   // 本结构的引用计数
      bool registered : 1; // 是否在 SnapSet Cache 中记录
      bool exists : 1;     // snapset 是否存在
    
      explicit SnapSetContext(const hobject_t& o) :
        oid(o), ref(0), registered(false), exists(true) { }
    };
    ```

    ## 读写流程

    OSD 服务端收到消息后，分三个阶段进行读写。

    ### 接收请求

    读写请求都是从 `void OSD::ms_fast_dispatch(Message *m)` 开始，

    # 本地对象存储

    RADOS 本地对象存储基于本地文件系统实现，默认的本地文件系统为 XFS（现已经是 bluestore），对应到本地文件系统里，一个对象就是一个固定大小的文件。对象所属的 PG 会作为文件目录，当一个 PG 内（注： 这里应该是目录内）所包含的文件超过一定程度时（在目录内文件太多会造成文件系统的 lookup 性能损耗），相应的目录会进行分裂。

    RADOS 保存数据有三种方式：object data、object xattr、object omap。data 是保存对象的数据，xattr 是保存对象的扩展属性，这要求支持对象存储的本地文件系统（一般是 XFS）支持扩展属性。如果要设置的对象的 key/value 因为操作系统限制，不能存储在文件的扩展属性中，还存在另外一种方式保存 omap， omap 是 object map 的简称，是将元数据保存在本地文件系统之外的独立 key-value 存储系统中，在使用 filestore 时是 leveldb，在使用 bluestore 时是 rocksdb。

    对象如何在本地文件系统中组织的代码实现在 `src/os` 中。本章将介绍在单个 OSD 上数据如何写入磁盘中。

    ## ObjectStore 对象存储接口

    位于 `src/os/ObjectStore.h` 中。

    所有的对象存储引擎都必须继承和实现 ObjectStore 定义的接口。

    ### 初始化

    1. create 创建 ObjectStore 实例；
    2. mkfs 创建 ObjectStore 相关的系统信息；
    3. mount 加载 ObjectStore 相关的系统信息；
    4. statfs 获取 ObjectStore 相关的系统信息。

    ### 获取属性

    1. getattr 获取对象的一个 xattr；（****）
    2. omap_get 获取对象的 omap 信息。

    ## omap

    omap 就是 object map 的简称，是一些键值对，保存在本地文件系统之外的独立的 key-value 存储系统中，例如 leveldb、rockdb 等。xattrs 保存一些比较小而经常访问的信息。omap 保存一些大而不是经常访问的数据。

    位于 `src/os/ObjectMap.h` 的 ObjectMap 类定义了 omap 的抽象接口。

    omap 在 leveldb 中的存储分两步，保存（1）"hobject_seq" + ghobject_key(oid) -> header 的键值对，header 中包含对象在 leveldb 中的唯一表示 seq；（2）"USER" + header_key(header->seq) + "AXATTR" + key -> value (omap 的值)。

    ### 部分代码实现分析

    1. lookup_create_map_header：获取对象的 header。
        - 首先在 caches 里查找，caches 缺失则调用底层 KeyValueDB 的 db->get() 查找；
        - 上述查找成功直接返回，否则调用 _generate_new_header 来设置 header 并更新 omap 状态，再调用 set_map_header 把新 header 设置到 leveldb 和 caches 中。
    2. get_attrs：获取对象的属性。
        - lookup_map_header 获取对象的 header；
        - db->get 获取具体属性。

    ## BlueStore

    速通，对于关注的缓存部分以及 IO 流程着重了解，其余粗略扫描。 

    ### [架构设计](https://zhuanlan.zhihu.com/p/91015613)
    
    FileStore 与 BlueStore 的对比：
    
    1. FileStore 先写 `Journal`，再写文件系统，会有一倍写放大，而文件系统一般都是日志型文件系统（xfs、ext系列），也会写 `Journal`，因此维护了两份日志；同时 FileStore 没有针对 SSD 进行优化。
    2. `BlueStore`选择绕过文件系统，直接接管裸设备，直接进行对象数据 IO 操作，同时元数据存放在`RocksDB`，大大缩短了整个对象存储的IO路径。`BlueStore`可以理解为一个支持`ACID`事物型的本地日志文件系统。
    
    BlueStore 在减少写日志开销上的努力：
    
    1. 针对大范围的覆盖写，只在其前后非磁盘块大小对齐的部分使用 `Journal`，即 `RMW`，其他部分直接重定向写 `COW` 即可。
        - `RMW (Read-Modify-Write)`：指当覆盖写发生时，如果本次改写的内容不足一个 `BlockSize`，那么需要先将对应的块读上来，然后在内存中将原内容和待修改内容 `Merge`，最后将新的块写到原来的位置。但是 `RMW` 也带来了两个问题：**一是**需要额外的读开销；**二是 **`RMW` 不是原子操作，如果磁盘中途掉电，会有数据损坏的风险。为此需要引入 `Journal`，先将待更新数据写入 `Journal`，然后再更新数据，最后再删除`Journal`对应的空间；
        - `COW (Copy-On-Write)`：指当覆盖写发生时，不是更新磁盘对应位置已有的内容，而是新分配一块空间，写入本次更新的内容，然后更新对应的地址指针（**rocksdb “原子”更新元数据？**），最后释放原有数据对应的磁盘空间。理论上 `COW` 可以解决 `RMW` 的两个问题，但是也带来了其他的问题：（1）`COW `机制破坏了数据在磁盘分布的物理连续性。经过多次 `COW` 后，顺序读将会便会随机读。（2）针对小于块大小的覆盖写采用 `COW` 会得不偿失。**是因为**：（1）将新的内容写入新的块后，原有的块仍然保留部分有效内容，不能释放无效空间，而且再次读的时候需要将两个块读出来做 `Merge` 操作，才能返回最终需要的数据，将大大影响读性能；（2）存储系统一般元数据越多，功能越丰富，元数据越少，功能越简单。而且任何操作必然涉及元数据，所以元数据是系统中的热点数据。`COW` 涉及空间重分配和地址重定向，将会引入更多的元数据，进而导致系统元数据无法全部缓存在内存里面，性能会大打折扣。
    2. 基于以上设计理念，`BlueStore` 的写策略综合运用了 `COW` 和 `RMW` 策略。**非覆盖写**直接分配空间写入即可（**新分配一块空间，写入内容，然后“原子”写入元数据？**）；**块大小对齐的覆盖写**采用 `COW` 策略；**小于块大小的覆盖写**采用 `RMW` 策略。
    3. rocksdb 基于为其量身定做的本地文件系统 blueFS，blueFS 向块设备写入元数据时，需要先写日志，再写元数据（WAL），为加速这一过程，将 .log 和 .sst 分开存储，.log 使用速度更快的存储介质；在引入BlueFS后，BlueStore将所有存储空间从逻辑上分了3个层次：
        - **慢速空间(Block)**：存储对象数据，可以使用 `HDD`，由`BlueStore`管理。
        - **高速空间(DB)**：存储 `RocksDB` 的 `sst` 文件，可以使用 `SSD`，由 `BlueFS` 管理。
        - **超高速空间(WAL)**：存储 `RocksDB` 的 `log` 文件，可以使用 `NVME`，由 `BlueFS `管理。
    
    BlueStore 读写流程概览：
    
    1. 读取数据时，先从 RocksDB 找到对应的磁盘空间，然后通过 `BlockDevice` 读出数据；
    2. 处理写请求时会进入事物的状态机，简单流程就是先写数据，然后再原子的写入对象元数据和分配结果元数据。写入数据如果是对齐写入，则最终会调用 `do_write_big`；如果是非对齐写，最终会调用 `do_write_small`。
    
    ### [BitMap 分配器](https://zhuanlan.zhihu.com/p/91018497)
    
    ### Stupid 分配器
    
    ### [BlockDevice](https://zhuanlan.zhihu.com/p/91020703)
    
    设备的空间管理以及使用由 BlueFS 和 BlueStore 决定，BlockDevice 仅仅提供同步 IO 和异步 IO 的操作接口。
    
    **BlueStore 通过 libaio 实现异步 IO，这也是默认的 IO 模式。**
    
    {% asset_img aio.jpg aio %}
    
    AIO 的读写实际是在 IOContext 结构体的成员变量 pending_aios 中追加了一个和 libaio 相关的 aio_t 结构。
    
    写：调用 `io_prep_pwritev` 将写的数据放入buf中，此时还在内存，没有写入磁盘。
    
    读：调用 `io_prep_pread` ，并开辟一块block对齐的内存，准备读数据，还没有从磁盘读。
    
    提交：准备完读写之后，就需要调用 `aio_submit` 提交准备的 IO 了。
    
    IO 提交后就可以返回了，BlueStore 起了一个单独的线程检查 IO 的完成情况，当真正完成的时候，执行回调函数通知调用方。
    
    ### [FreelistManager](https://zhuanlan.zhihu.com/p/91025247)
    
    ### [blueStore 中的对象 IO](https://zhuanlan.zhihu.com/p/92397191)
    
    1. Onode：**对象元数据在内存中的数据结构**；
    
        ```cpp
        struct Onode{
            Collection *c;  // Onode 所属 PG
            ghobject_t oid; // 对象
            mempool::bluestore_cache_meta::string key; // key under PREFIX_OBJ where we are stored
            bluestore_onode_t onode;  // onode 磁盘数据结构
            bool exists;              // true if object logically exists
            bool cached;              // Onode is logically in the cache
                                      // (it can be pinned and hence physically out
                                      // of it at the moment though)
            ExtentMap extent_map;     // 有序的Extent逻辑空间集合，持久化在RocksDB。lexetnt--->blob
            std::shared_ptr<int64_t> cache_age_bin;  ///< cache age bin
            ......
        }
        ```
    
    2. bluestore_onode：**对象元数据在磁盘中的数据结构**；
    
        ```cpp
        /// onode: per-object metadata
        struct bluestore_onode_t {
            uint64_t nid = 0;                    ///< numeric id (locally unique)
          	uint64_t size = 0;                   ///< object size
          	// mempool to be assigned to buffer::ptr manually，对象扩展属性
          	std::map<mempool::bluestore_cache_meta::string, ceph::buffer::ptr> attrs;
        }
        ```
    
    3. PExtent：**对应一段连续的磁盘空间**；offset 和 length 都是块大小对齐。
    
        ```cpp
        struct bluestore_pextent_t {
            uint64_t offset = 0; // 磁盘上的物理偏移
            uint32_t length = 0; // 数据段的长度
        }
        ```
    
    4. LExtent：Extent 是对象内的基本数据管理单元，数据压缩、数据校验、数据共享等功能都是基于 Extent 粒度实现的。这里的Extent 是对象内的，并不是磁盘内的，所以称为 lextent，和磁盘内的 pextent 以示区分。
    
        ```cpp
        struct Extent {
            uint32_t logical_offset = 0; // 对象内逻辑偏移，不需要块对齐。
            uint32_t length = 0; // 逻辑段长度，不需要块对齐。
            
            // 当logical_offset是块对齐时，blob_offset始终为0；
            // 不是块对齐时，将逻辑段内的数据通过Blob映射到磁盘物理段会产生物理段内的偏移称为blob_offset。
            uint32_t blob_offset = 0;
            BlobRef  blob;                    ///< the blob with our data
        }
        ```
    
    5. blob：Blob 包含磁盘上物理段的集合，即 `bluestore_pextent_t` 的集合。
    
        ```cpp
        struct Blob {
            // reference count
            std::atomic_int nref = {0};
        
            mutable bluestore_blob_t blob;
        }
        
        struct bluestore_blob_t {
            PExtentVector extents;
        }
        ```
    
    ### [Cache](https://zhuanlan.zhihu.com/p/92396291)
    
    1. 数据结构：
    
        ```cpp
        // A generic Cache Shard
        struct CacheShard {
            PerfCounters *logger; // 统计缓存命中率等信息
            ceph::recursive_mutex lock = {ceph::make_recursive_mutex("BlueStore::CacheShard::lock") };
         	std::atomic<uint64_t> max = {0}; // 对于onode，为最大缓存对象数；对于buffer，为最大缓存大小
            std::atomic<uint64_t> num = {0}; // 现有缓存的对象数
            boost::circular_buffer<std::shared_ptr<int64_t>> age_bins;  
            ......
        }
        
        // A Generic onode Cache Shard
        struct OnodeCacheShard : public CacheShard {
            std::array<std::pair<ghobject_t, ceph::mono_clock::time_point>, 64> dumped_onodes;
        	......
        }
        
        // A Generic buffer Cache Shard
        struct BufferCacheShard : public CacheShard {
            std::atomic<uint64_t> num_extents = {0};  // 当前缓存的extent数
            std::atomic<uint64_t> num_blobs = {0};    // 当前缓存的blob数
            uint64_t buffer_bytes = 0; // buffer缓存数据总大小
        }
        
        // LruOnodeCacheShard
        struct LruOnodeCacheShard : public BlueStore::OnodeCacheShard {
            list_t lru;
            ......
        }
        
        // LruBufferCacheShard
        struct LruBufferCacheShard : public BlueStore::BufferCacheShard {
         	list_t lru;
            ......
        }
        
        struct TwoQBufferCacheShard : public BlueStore::BufferCacheShard {
          	list_t hot;      ///< "Am" hot buffers
          	list_t warm_in;  ///< "A1in" newly warm buffers
          	list_t warm_out; ///< "A1out" empty buffers we've evicted
        
          	enum {
            	BUFFER_NEW = 0,
            	BUFFER_WARM_IN,   ///< in warm_in
            	BUFFER_WARM_OUT,  ///< in warm_out
            	BUFFER_HOT,       ///< in hot
            	BUFFER_TYPE_MAX
            };
            uint64_t list_bytes[BUFFER_TYPE_MAX] = {0}; //< bytes per type
        }
        ```
    
        由于缓存的绝大部分操作都要加锁而且Onode使用一个链表，为了更高的性能，需要对Cache进行分片，
    
    2. 初始化
    
        ```cpp
        BlueStore::BlueStore(CephContext *cct, const string& path)
            : ObjectStore(cct, path),
        {
            ......
            set_cache_shards(1); // 初始化cache
        }
        
        void BlueStore::set_cache_shards(unsigned num)
        {
          	size_t oold = onode_cache_shards.size();
          	size_t bold = buffer_cache_shards.size();
          	ceph_assert(num >= oold && num >= bold);
          	onode_cache_shards.resize(num);
          	buffer_cache_shards.resize(num);
          	for (unsigned i = oold; i < num; ++i) {
          	  	onode_cache_shards[i] = 
          	      	OnodeCacheShard::create(cct, cct->_conf->bluestore_cache_type,logger);
          	}
          	for (unsigned i = bold; i < num; ++i) {
           		buffer_cache_shards[i] = 
                	BufferCacheShard::create(cct, cct->_conf->bluestore_cache_type,logger);
          	}
        }
        
        // OnodeCacheShard
        BlueStore::OnodeCacheShard *BlueStore::OnodeCacheShard::create(
            CephContext* cct,
            string type,
            PerfCounters *logger)
        {
         	 BlueStore::OnodeCacheShard *c = nullptr;
          	// Currently we only implement an LRU cache for onodes
          	c = new LruOnodeCacheShard(cct);
          	c->logger = logger;
          	return c;
        }
        
        // BuferCacheShard
        BlueStore::BufferCacheShard *BlueStore::BufferCacheShard::create(
            CephContext* cct,
            string type,
            PerfCounters *logger)
        {
          	BufferCacheShard *c = nullptr;
          	if (type == "lru")
          		c = new LruBufferCacheShard(cct);
          	else if (type == "2q")
            	c = new TwoQBufferCacheShard(cct);
          	else
            	ceph_abort_msg("unrecognized cache type");
          	c->logger = logger;
          	return c;
        }
        
        size_t OSD::get_num_cache_shards()
        {
          	return cct->_conf.get_val<Option::size_t>("osd_num_cache_shards");
        }
        ```
    
    3. 元数据
    
        BlueStore主要的元数据有两种类型：Collection、Onode；其中Collection常驻内存，缓存的便是Onode了。
    
        - **Collocation对应PG在BlueStore中的内存数据结构，Cnode对应PG在BlueStore中的磁盘数据结构。**在创建 pg 的时候，会将 pg 的元信息存在 kv 中；在osd上电的时候，会创建collection，并从kv加载所有collection信息，同时会指定该 PG 内对象的数据和元数据缓存到哪个 buffercache 和 onodecache，参见函数 _open_collections。
    
            ```cpp
            struct Collection : public CollectionImpl {
                BlueStore *store;
                OpSequencerRef osr;  		// 每个PG有一个osr，在BlueStore层面保证读写事物的顺序性和并发性。
                BufferCacheShard *cache;    // < our cache shard
                bluestore_cnode_t cnode;	// pg的磁盘结构
                ceph::shared_mutex lock =
                  ceph::make_shared_mutex("BlueStore::Collection::lock", true, false);
            
                bool exists;
            
                SharedBlobSet shared_blob_set;      ///< open SharedBlobs
            
                // cache onodes on a per-collection basis to avoid lock contention.
                OnodeSpace onode_space;
            	......
            }
            
            // PG的磁盘数据结构
            struct bluestore_cnode_t {
                uint32_t bits;
            }
            
            struct OnodeSpace {
                OnodeCacheShard *cache;
            private:
                /// forward lookups
                mempool::bluestore_cache_meta::unordered_map<ghobject_t,OnodeRef> onode_map;
            }
            ```
    
        - 每个对象对应一个Onode数据结构，而Object是属于PG的，所以Onode也属于PG，为了方便针对PG的操作(删除Collection时，不需要完整遍历Cache中的Onode队列来逐个检查与被删除Collection的关系)，引入了中间结构OnodeSpace使用一个map来记录Collection和Onode的映射关系。单个BlueStore上面有海量的Onode，不一定能够全部缓存在内存里面，只能最大程度的缓存Onode信息在内存。
    
    4. 数据：对象的数据使用`BufferSpace`来管理。作为一个二级索引。
    
        ```cpp
        struct BufferSpace {
            // buffer map
            mempool::bluestore_cache_meta::map<uint32_t, std::unique_ptr<Buffer>> buffer_map;
        
            // we use a bare intrusive list here instead of std::map because
            // it uses less memory and we expect this to be very small (very
            // few IOs in flight to the same Blob at the same time).
            // 包含脏数据的缓存队列
            state_list_t writing;   ///< writing buffers, sorted by seq, ascending
        }
        ```
    
        当数据写完时，会根据标记决定是否缓存。`BlueStore::BufferSpace::_finish_write`；当读取完成时，也会根据标记决定是否缓存。`bluestore_default_buffered_read`、`BlueStore::_do_read`。（**这里和缓存准入有关系吗？**）
    
    5. Trim：BlueStore的元数据和数据都是使用内存池管理的，后台有内存池线程监控内存的使用情况，超过内存使用的限制便会做trim，丢弃一部分缓存的元数据和数据。（**缓存分配方案也需要重点关注这一块！！！**）
    
        ```cpp
        void *BlueStore::MempoolThread::entry() {
        	......   
        }
        ```
    
        
    
    # Ceph 自动分层
    
    
    



​		



[Ceph源码分析](https://item.jd.com/12072602.html)

[shimingyah个人博客](https://shimingyah.github.io/tags/#Ceph-ref)

[鱼香肉丝没有鱼](https://www.zhihu.com/column/c_1175452658096476160)