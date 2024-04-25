---
title: Redis
date: 2023-4-20 15:52:57
description: Redis
tags:
- Redis
- 源码分析
categories:
- Redis
---

# 基本数据结构

## 简单动态字符串

位于 `sds.h` 和 `sds.c` 中。

C语言用 '\0' 作为字符串结束符，而如果字符串内容本身就包含 '\0' 则会被截断，无法做到二进制安全。因此，redis 将**柔性字符数组**包装到 sds 数据结构中，同时使用 len 和 alloc 表示已使用字符数组的长度和预分配长度。

**注**：柔性数组只能放在结构体的末尾，而之所以使用柔性数组而不是字符指针，是因为柔性数组的地址和结构体是连续的，可以方便地通过柔性数组的首地址偏移得到结构体的首地址（`buf[-1]` 一定是 `flags` 字段，进一步可以知道所在结构体的大小）。

为精细地管理字符串，节约内存空间，设置了各个长度范围的字符串结构体。

```cpp
struct __attribute__((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 个最低有效位表示类型, 同时 5 个最高有效位表示字符串长度 */
    char buf[];
};
struct __attribute__((__packed__)) sdshdr8 {
    uint8_t len;         /* 已使用的长度 */
    uint8_t alloc;       /* 分配给 buf 的长度 - 1 （不包含结构体中的元数据以及 buf 中的
                            '\0' 结束符） */
    unsigned char flags; /* 3位最低有效位表示类型, 其余5个比特位未被使用 */
    char buf[];
};
struct __attribute__((__packed__)) sdshdr16 {
    uint16_t len;        /* 已使用的长度 */
    uint16_t alloc;      /* 分配给 buf 的长度 - 1 （不包含结构体中的元数据以及 buf 中的
                           '\0' 结束符） */
    unsigned char flags; /* 3位最低有效位表示类型, 其余5个比特位未被使用 */
    char buf[];
};
struct __attribute__((__packed__)) sdshdr32 {
    uint32_t len;        /* 已使用的长度 */
    uint32_t alloc;      /* 分配给 buf 的长度 - 1 （不包含结构体中的元数据以及 buf 中的
                           '\0' 结束符） */
    unsigned char flags; /* 3位最低有效位表示类型, 其余5个比特位未被使用 */
    char buf[];
};
struct __attribute__((__packed__)) sdshdr64 {
    uint64_t len;        /* 已使用的长度 */
    uint64_t alloc;      /* 分配给 buf 的长度 - 1 （不包含结构体中的元数据以及 buf 中的
                           '\0' 结束符） */
    unsigned char flags; /* 3位最低有效位表示类型, 其余5个比特位未被使用 */
    char buf[];
};
```

## 双向链表

位于 `adlist.h` 和 `adlist.c` 中。

redis 双向链表的节点可以存储任意类型的 value，因此，每个 list 可以设置自己的 dup、free 和 match 函数，达到多态目的。

```cpp
/* 双端链表节点 */
typedef struct listNode {
    /* 指向前驱节点的指针 */
    struct listNode *prev;
    /* 指向后继节点的指针 */
    struct listNode *next;
    /* void * 指针，指向具体的元素，节点可以是任意类型 */
    void *value;
} listNode;

/* 双端链表
 * 有记录头尾两节点，支持从链表头部或者尾部进行遍历，是早期列表键 PUSH/POP 实现高效的关键
 * 每个链表节点有记录前驱节点和后继节点的指针，可以使得列表键支持往后或者往前进行遍历
 * 有额外用 len 存储链表长度，O(1) 的时间复杂度获取节点个数，是 LLEN 命令高效的关键 */
typedef struct list {
    /* 指向链表头节点的指针，支持从表头开始遍历 */
    listNode *head;
    /* 指向链表尾节点的指针，支持从表尾开始遍历 */
    listNode *tail;
    /* 各种类型的链表可以定义自己的复制函数 / 释放函数 / 比较函数 */
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    /* 链表长度，即链表节点数量，O(1) 时间复杂度获取 */
    unsigned long len;
} list;
```

## 字典

位于 `dict.h` 和 `dict.c` 中。

dictEntry 存储键值对；dict 为字典结构，其中使用了两个哈希表，第二个哈希表一般情况下不用，只有在扩容缩容操作时，避免长时间 rehash 导致阻塞，将新申请空间存在第二个哈希表中，rehash 过程中，添加操作往第二个哈希表中进行，查找、删除和修改在两个哈希表中依次进行，除此之外，第一个哈希表中的键值对需要在服务空闲时重新计算哈希桶索引，全部迁移插入到第二个哈希表中；dictType 为字典类型，针对不同类型的键和值配置合适的复制、析构和比较函数等，达到多态目的。

```cpp
typedef struct dictEntry {
    /* void * 类型的 key，可以指向任意类型的键 */
    void *key;
    /* 联合体 v 中包含了指向实际值的指针 *val、无符号的 64 位整数、有符号的 64 位整数，以及 double 双精度浮点数。
     * 这是一种节省内存的方式，因为当值为整数或者双精度浮点数时，由于它们本身就是 64 位的，void *val 指针也是占用 64 位（64 操作系统下），
     * 所以它们可以直接存在键值对的结构体中，避免再使用一个指针，从而节省内存开销（8 个字节）
     * 当然也可以是 void *，存储任何类型的数据，最早 redis1.0 版本就只是 void* */
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* 同一个 hash 桶中的下一个条目.
                                 * 通过形成一个链表解决桶内的哈希冲突. */
    void *metadata[];           /* 一块任意长度的数据 (按 void* 的大小对齐),
                                 * 具体长度由 'dictType' 中的
                                 * dictEntryMetadataBytes() 返回. */
} dictEntry;

/* 字典类型，因为我们会将字典用在各个地方，例如键空间、过期字典等等等，只要是想用字典（哈希表）的场景都可以用
 * 这样的话每种类型的字典，它对应的 key / value 肯定类型是不一致的，这就需要有一些自定义的方法，例如键值对复制、析构等 */
typedef struct dictType {
    /* 字典里哈希表的哈希算法，目前使用的是基于 DJB 实现的字符串哈希算法
     * 比较出名的有 siphash，redis 4.0 中引进了它。3.0 之前使用的是 DJBX33A，3.0 - 4.0 使用的是 MurmurHash2 */
    uint64_t (*hashFunction)(const void *key);
    /* 键拷贝 */
    void *(*keyDup)(dict *d, const void *key);
    /* 值拷贝 */
    void *(*valDup)(dict *d, const void *obj);
    /* 键比较 */
    int (*keyCompare)(dict *d, const void *key1, const void *key2);
    /* 键析构 */
    void (*keyDestructor)(dict *d, void *key);
    /* 值析构 */
    void (*valDestructor)(dict *d, void *obj);
    /* 字典里的哈希表是否允许扩容 */
    int (*expandAllowed)(size_t moreMem, double usedRatio);
    /* 允许调用者向条目 (dictEntry) 中添加额外的元信息.
     * 这段额外信息的内存会在条目分配时被零初始化. */
    size_t (*dictEntryMetadataBytes)(dict *d);
} dictType;

struct dict {
    /* 字典类型，8 bytes */
    dictType *type;
    /* 字典中使用了两个哈希表,
     * (看看那些以 'ht_' 为前缀的成员, 它们都是一个长度为 2 的数组)
     *
     * 我们可以将它们视为
     * struct{
     *   ht_table[2];
     *   ht_used[2];
     *   ht_size_exp[2];
     * } hash_table[2];
     * 为了优化字典的内存结构,
     * 减少对齐产生的空洞,
     * 我们将这些数据分散于整个结构体中.
     *
     * 平时只使用下标为 0 的哈希表.
     * 当需要进行 rehash 时 ('rehashidx' != -1),
     * 下标为 1 的一组数据会作为一组新的哈希表,
     * 渐进地进行 rehash 避免一次性 rehash 造成长时间的阻塞.
     * 当 rehash 完成时, 将新的哈希表置入下标为 0 的组别中,
     * 同时将 'rehashidx' 置为 -1.
     */
    dictEntry **ht_table[2];
    /* 哈希表存储的键数量，它与哈希表的大小 size 的比值就是 load factor 负载因子，
     * 值越大说明哈希碰撞的可能性也越大，字典的平均查找效率也越低
     * 理论上负载因子 <=1 的时候，字典能保持平均 O(1) 的时间复杂度查询
     * 当负载因子等于哈希表大小的时候，说明哈希表退化成链表了，此时查询的时间复杂度退化为 O(N)
     * redis 会监控字典的负载因子，在负载因子变大的时候，会对哈希表进行扩容，后面会提到的渐进式 rehash */
    unsigned long ht_used[2];

    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
                    /* rehash 的进度.
                     * 如果此变量值为 -1, 则当前未进行 rehash. */
    /* 将小尺寸的变量置于结构体的尾部, 减少对齐产生的额外空间开销. */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
                         /* 如果此变量值 >0 表示 rehash 暂停
                          * (<0 表示编写的代码出错了). */
    /* 存储哈希表大小的指数表示，通过这个可以直接计算出哈希表的大小，例如 exp = 10, size = 2 ** 10
     * 能避免说直接存储 size 的实际值，以前 8 字节存储的数值现在变成 1 字节进行存储 */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */
                                /* 哈希表大小的指数表示.
                                 * (以 2 为底, 大小 = 1 << 指数) */
};
```

## 跳跃表

`server.h` 中的 `zskiplist` 结构和 `zskiplistNode` 结构，`t_zset.c` 中以 `zsl` 开头的函数。

{% asset_img 跳跃表.jpg 跳跃表 %}

```cpp
typedef struct zskiplistNode {
    sds ele;					       // 存储字符串类型数据
    double score;                      // 优先分数 
    struct zskiplistNode *backward;    // 指向当前节点最底层的前一个节点
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 指向本层下一节点，尾节点指向 NULL
        unsigned long span;            // forward 指向的节点和本节点之间的元素个数，span 值越大，跳过的节点个数越多
    } level[]; 						   // 柔性数组，每个节点的数组长度不一样
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;  // 跳跃表节点个数（等于元素总数，不包括头节点）
    int level;             // 跳跃表高度（除头节点外，层数最多的节点的层高）
} zskiplist;
```

跳跃表的头节点高度为 `MAXLEVEL`，其它节点创建时随机设置高度，高度越高概率越小。

重点是 `update` 和 `rank` 数组的理解。

`update[]`：记录每层比插入元素 score 或字典序小的最近节点；

`rank[]`：记录插入位置在每层中从头节点跨越了多少个节点。

## 整数集合

位于 `intset.h` 和 `intset.c` 中。

整数集合中的元素**从小到大**排列在数组中。为节省空间，有三种底层数组类型，随着插入整数的编码提升，提升整数集合底层数组的类型。（插入和删除操作效率比较低，涉及 `memmove`）

```cpp
/* 整数集合 
 * 记录不包含重复元素的各个整数(由小到大的顺序) 
 * 底层数组默认是 int16_t 类型, 可能随着新增元素的大小升级至 int32_t 或 int64_t 类型*/
typedef struct intset {
    /* 编码, 记录整数集合底层数组(contents)的类型*/
    uint32_t encoding;
    /* 记录整数集合包含的元素个数 */
    uint32_t length;
    /* 整数集合的底层实现, 虽声明为 int8_t 类型,但真正的类型取决于 encoding */
    int8_t contents[];
} intset;
```

## 压缩列表

位于 `ziplist.h` 和 `ziplist.c` 中。

压缩列表适合存储小整数以及短字符串，并且元素数量较少。

{% asset_img 压缩列表.jpg 压缩列表 %}

压缩列表中的 entry 在内存中有三个字段，previous_entry_length 字段表示**前一个元素**的字节长度，占1个或者5个字节，当前一个元素的长度小于254字节时，用1个字节表示；当前一个元素的长度大于或等于254字节时，用5个字节来表示。而此时 previous_entry_length 字段的第1个字节是固定的0xFE，后面4个字节才真正表示前一个元素的长度。假设已知当前元素的首地址为p，那么 p-previous_entry_length 就是前一个元素的首地址，从而实现压缩列表从尾到头的遍历。

{% asset_img 压缩列表entry.jpg 压缩列表entry %}

encoding 字段同样长度可变，为1字节、2字节或者5字节，用来表示**当前元素**的数据和字节长度；content 字段则是存储的实际内容。

{% asset_img 压缩列表元素的编码.jpg 压缩列表元素的编码  %}

压缩列表中的 entry 被解码后保存在 zlentry 结构体中。

```cpp
/* 这是一个很关键的结构体，将 ziplist 节点信息填充成一个 zlentry 结构体，方便后面进行函数操作
 * 需要注意这并不是一个 ziplist 节点在内存中实际的编码布局，只是为了方便我们使用
 * */
typedef struct zlentry {
    /* 存储下面 prevrawlen 所需要的字节数 */
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    /* 存储前一个节点的字节长度 */
    unsigned int prevrawlen;     /* Previous entry len. */
    /* 存储下面 len 所需要的字节数 */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    /* 存储当前节点的字节长度 */
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    /* prevrawlensize + lensize 当前节点的头部字节，
     * 其实是 prevlen + encoding 两项占用的字节数 */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    /* 存储当前节点的数据编码格式 */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    /* 指向当前节点开头第一个字节的指针 */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;
```

## 快速列表

位于 `quicklist.h` 和 `quicklist.c` 中。

quicklist 是一个双向链表，链表中的每个节点是一个压缩列表。

# 数据对象

Redis 基于上述数据结构创建了一个对象系统，这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象这五种类型的对象，每种对象都用到了至少一种我们前面所介绍的数据结构。

位于 `server.h` 中的 `redisObject` 数据结构（robj）的 type 字段表示对象类型；encoding 字段表示底层数据存储结构（针对某一种类型对象，redis 可能会根据情况采用不同的数据结构存储）；ptr 指向数据所在的底层数据结构的存储位置。对象也可能使用多种数据结构存储，比如有序集合可以采用字典和跳跃表同时存储，分别利用两者的单点查询和范围查询优势，由于数据都是用指针存储，因此额外开销只是相关数据结构的 header 相关字段。

```cpp
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */

/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

# Redis 处理客户端命令

## 相关数据结构

客户端数据结构，位于 `server.h` 中。

```cpp
typedef struct client {
    uint64_t id;            /* 客户端唯一 id */
    redisDb *db;            /* 指向当前选择的数据库 */
    robj *name;             /* 客户端名称 */
    sds querybuf;           /* 输入缓冲区，recv 函数接收的客户端命令请求暂时缓存在此处 */
    int argc;               /* 命令请求的参数个数 */
    robj **argv;            /* 命令请求的参数内容依次解析到此处 */

    list *reply;            /* 存储待返回给客户端的命令回复数据 */
    unsigned long long reply_bytes; /* reply 列表所有节点的存储空间之和 */
    size_t sentlen;         /* 已返回给客户端的字节数 */

    time_t lastinteraction; /* 客户端上次与服务器交互时间，用于客户端的超时处理 */

    /* Response buffer */
    size_t buf_peak; /* Peak used size of buffer in last 5 sec interval. */
    mstime_t buf_peak_last_reset_time; /* keeps the last time the buffer peak value was reset */
    int bufpos;
    size_t buf_usable_size; /* Usable size of buffer. */
    char *buf;
    
    ......
} client;
```

服务端数据结构，位于 `server.h` 中。

```cpp
struct redisServer{
	char *configfile; 						/* 配置文件绝对路径 */
	int dbnum; 								/* 数据库的数目 */
	redisDb *db; 							/* 数据库数组 */
    dict *commands; 						/* 命令字典，放置命令名称到命令对象的映射 */
    aeEventLoop *el; 						/* 事件循环 */
    int port; 								/* 服务器监听端口号，可通过参数 port 配置，默认为 6379 */
	char * bindaddr[CONFIG_BINDADDR_MAX]; 	/* 绑定的所有 IP */
    int bindaddr_count; 					/* 用户配置的 IP 地址数目 */
    socketFds ipfd; 						/* 存储所有 IP 地址创建的 socket 文件描述符 */
	list *clients; 							/* 当前连接的所有客户端 */
    int maxidletime; 						/* 最大空闲时间 */
    
    ......
}
```

命令数据结构，位于 `server.h`  中。Redis 支持的所有命令都在全局变量 `struct redisCommand redisCommandTable[]` 中。

```cpp
struct redisCommand {
	char *declared_name;			/* 命令名称 */
	redisCommandProc *proc; 		/* 命令处理函数 */
	int arity;						/* 命令参数数目（-N 表示参数数目必须 >= N；N 表示参数数目必须 = N） */
	int flags;						/* 命令的二进制标志 */
	long long microseconds, calls;	/* 服务器启动至今该命令总执行时间；该命令总执行次数。用于统计 */
};
```

当服务端收到一个命令请求时，如果到 redisCommandTable 中查找命令，耗时太高。因此，Redis 在服务器初始时，将命令表转化为一个字典。

事件循环数据结构，位于 `ae.h` 中。文件事件（socket 的可读可写时间）和时间事件（定时任务）都封装在该数据结构中

```cpp
/* State of an event based program */
/* 事件处理器的状态 */
typedef struct aeEventLoop {
    /* 目前已经注册的最大文件描述符 */
    int maxfd;   /* highest file descriptor currently registered */
    /* 目前追踪的最大文件描述符数量，redis 初始化的时候会设置好的一个固定值
     * 初始化事件循环器的时候会把这个值赋值给 setsize，见 server.h 文件的 CONFIG_FDSET_INCR
     * server.maxclients + RESERVED_FDS（32） + 96 */
    int setsize; /* max number of file descriptors tracked */
    /* 用于生成时间事件的 ID */
    long long timeEventNextId;
    /* 已注册的文件事件，在初始化的时候会初始化 setsize 个位置 */
    aeFileEvent *events; /* Registered events */
    /* 已就绪的文件事件 */
    aeFiredEvent *fired; /* Fired events */
    /* 时间事件链表的头节点 */
    aeTimeEvent *timeEventHead;
    /* 事件处理器的开关 */
    int stop;
    /* 多路复用库的私有数据 */
    void *apidata; /* This is used for polling API specific data */
    /* 在处理事件前要执行的函数 */
    aeBeforeSleepProc *beforesleep;
    /* 在处理事件后要执行的函数 */
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```

## 服务端启动

### 初始化

1. 初始化配置，包括用户可配置的参数，以及命令表数组初始化成字典；

    位于 `server.c` 中的 `initServerConfig` 函数执行。

2. 加载并解析配置文件；

    位于 `config.c` 中的 `loadServerConfig` 函数执行。

3. 初始化服务器内部变量，比如数据库、全局变量和共享对象等；

    位于 `server.c` 中的 `initServer` 函数执行。

4. 创建事件循环 eventLoop，分配结构体所需内存，创建 epoll；

    位于 `ae.c` 中的 `aeCreateEventLoop` 函数执行。

### 启动监听

1. 创建 socket 并启动监听；

    位于 `server.c` 中的 `listenToPort` 函数执行。

2. 创建文件事件和时间事件；

3. 开启事件循环；

    位于 `ae.c` 中的 `aeMain` 函数执行。

## 命令处理

服务端启动完成后，只需要等待客户端连接并发送命令请求即可。

### 命令解析

命令请求首先通过 `readQueryFromClient` 函数处理后存储在客户端的 `querybuf` 中，并调用 `processInputBuffer` 函数解析命令请求，将参数个数和各个参数存储在客户端的 `argc` 和 `argv` 中。

### 命令调用

解析完成后，会调用 `processCommand` 函数处理命令请求，包括一系列的校验以及最后调用 `call` 函数，通过 `c->cmd->proc(c)` 执行命令。**(后续如果想要读命令的执行细节，就只需要关注这一块就可以了)**

### 返回结果

结果保存在客户端的 `reply` 或 `buf` 中。