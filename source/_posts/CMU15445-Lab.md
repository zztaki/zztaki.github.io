---
title: CMU15445-Lab
date: 2023-03-14 21:36:47
description: CMU15445-Lab
tags:
- CMU15445
- 公开课
- 数据库
categories:
- 数据库
---

# CMU15445 Fall2022

## HW1 SQL

### sqlite3 基础

1. 命令行键入 `sqlite3 db_name.db` 进入数据库；
2. `.tables` 展示检查数据库内容；
3. `.scheme table_name` 熟悉表的模式（结构）（包含什么属性、索引，主键和外键是什么）；

### 实验问题

1. q2：使用 `||` 拼接两个字符串；
2. q3：使用 `data('now')` 返回当前时间，`substr()` 截取子字符串；
3. q6：sql 子查询似乎很慢...
4. q9：[NTILE()](https://www.sqlitetutorial.net/sqlite-window-functions/sqlite-ntile/)；with 视图；
5. q10：[Recursive CTEs](https://sqlite.org/lang_with.html)；窗口函数 `ROW_NUMBER()`；



## Project1 Storage Manger

### 实验问题

#### [**Extendible Hash Table**](https://15445.courses.cs.cmu.edu/fall2022/project1/#extendible-hash-table)

这是本实验的第一个任务，实现一个可扩展的哈希表，这里的可扩展只需要考虑根据需求增加哈希表的大小，而不需要考虑缩小或压缩。其中哈希表的 `Find(k, v)` 和 `Remove(k)` 接口实现相对比较容易，比较麻烦的是 `Insert(k, v)` 接口，该接口需要考虑扩展哈希表以及为部分键值对重新分配哈希桶的问题。在此之前需要明确几个概念，

1. 哈希表的全局深度：整个哈希表用到了多少 bit 将 key 映射到某个哈希桶；
2. 哈希表中的某个桶的局部深度：该桶用到了多少个 bit 将 key 映射到该桶；
3. 重排哈希桶：当某个哈希桶满时，将新增哈希桶，并利用更多的 bit 将 key 映射到具体哈希桶中，从而减小哈希冲突（哈希桶的最大大小固定）。

从全局深度和局部深度的概念中不难得到`gloabl_depth >= max(local_depth_1, ...)`，因此，当想要插入新数据时，但新数据映射到的哈希桶满了，则

1. 若 `gloabl_depth = local_depth`，那么很遗憾，即使是使用了 `gloabl_depth` 个 bit 去映射，这个数据映射到的哈希桶仍然是满的，因此毫无疑问，我们需要利用到更多的 bit，以减弱哈希冲突，即 `gloabl_depth++`，相应的，哈希桶的数量应该翻倍。在这里引入了一个新问题，那就是我们是否需要立即分配一倍的哈希桶空间？这似乎有点像操作系统中的 `COW（Copy On Write）` 问题，我们这个时候只是一个哈希桶出现了冲突，而如果直接暴力地分配一倍的哈希桶空间，可能是浪费的，那么，如果不立即分配哈希桶，我们增加的 ”哈希桶“ 该指向哪里呢？试想一下，新增哈希桶是因为现在将利用更多的 bit 来进行映射，那么在没有利用更多 bit 映射之前，这些 key 将映射到哪个地方呢？

    ```cpp
    if (global_depth_ == GetLocalDepthInternal(idx)) {  // global depth is not enough, need more bit to hash
        for (int i = 0; i < (1 << global_depth_); i++) {
        	dir_.push_back(dir_[i]);
        }
        global_depth_++;
      }
    ```

2. 当前已满的哈希桶局部深度 `local_depth` 应该 +1，这是因为使用原 `local_depth` 的 bit 数量映射已经不足以解决该桶的哈希冲突了。

    ```cpp
    dir_[idx]->IncrementDepth();
    ```

3. 最后则是重排哈希桶，此时冲突桶使用到了更多的 bit 来进行映射，此时即触发了一次 `COW` 操作，需要真正地进行分配新哈希桶空间了，并对冲突桶中已满的所有数据进行重排，以决定哪些留在原桶，哪些从原桶中删除并加入到新桶中。

    ```cpp
    template <typename K, typename V>
    auto ExtendibleHashTable<K, V>::RedistributeBucket(std::shared_ptr<Bucket> bucket, size_t idx)
        -> std::shared_ptr<Bucket> {
      std::shared_ptr<Bucket> new_bucket = std::make_shared<Bucket>(Bucket(bucket_size_, bucket->GetDepth()));
      for (auto it = bucket->GetItems().begin(); it != bucket->GetItems().end();) {
        if (IndexOf(it->first, bucket->GetDepth()) == idx) {
          it++;
        } else {
          new_bucket->GetItems().push_back({it->first, it->second});
          it = bucket->GetItems().erase(it);
        }
      }
      return new_bucket;
    }
    ```

    再将指向新桶的指针加入到哈希表 `directory` 中的正确位置。**值得注意的是，新桶指针可能会覆盖 directory 中的多个位置，具体数量与 `gloabl_depth_ - local_depth_` 的值有关。**

4. 此时哈希冲突可能解决了，也可能发生巧合，就是原来已满的哈希桶中的所有数据使用多一 bit 映射时仍然映射到同一位置，因此需要递归进行 `Insert(k, v)`。

#### [**LRU-K Replacement Policy**](https://15445.courses.cs.cmu.edu/fall2022/project1/#lru-k-replacer)

LRU-K 缓存替换算法，问题的关键是设计相关的数据结构。`time_map_` 之所以需要设置成 `multimap`，是因为缓存中后向k重用距离为 ` inf` 的对象可能会有多个。

```cpp
// map frame_id_t to the queue of its k-access timeStamps
std::unordered_map<frame_id_t, std::queue<size_t>> refs_map_;
// real cache set, map frame_id to its iterator in time_map_
std::unordered_map<frame_id_t, std::multimap<size_t, frame_id_t>::iterator> cache_map_;
// multimap timeStamp to frame_id_t, its size should equal cache_map_'s size
std::multimap<size_t, frame_id_t> time_map_;
// the set contain all evictable frame_id
std::unordered_set<frame_id_t> evictable_frame_id_set_;
```

**踩坑：`SetEvictable(frame_id_t frame_id, bool set_evictable)` 时如果 frame_id 不在缓存中时，直接 return 就可以了。**

 #### [**Buffer Pool Manager Instance**](https://15445.courses.cs.cmu.edu/fall2022/project1/#buffer-pool-instance)

使用前两步完成的扩展哈希表和 LRU-K 缓存替换算法，构建一个缓冲池，相关接口的注释都比较完整。

1.  `NewPgImp(page_id_t)` 和 `FetchPgImp(page_id_t)` 接口实现时一定要记得显示地进行 `pages_[frame_id].pin_count++` 操作，而不仅仅是执行 `replacer.SetEvictable(frame_id, false)`。
2. `UnpinPgImp` 中 is_dirty： `pages_[frame_id].is_dirty_ |= is_dirty`。
3. `DeletePgImp` 需要检查删除的页在 buffer pool 中是否是脏的，如若是脏还需记得刷盘。

### 感受

人麻了，刚写完代码本地测完感觉很好，线上测试狂冒红光，为了过线上测试花了一天。。。

（很多奇怪的错误可能是由于锁的粒度太细了，用大锁就OK）

## Project2 B+Tree Index

### 实验问题

#### task1：B+Tree Pages

1. internal node 和 leaf node 能容纳的 K/V 键值对数目不同。internal node 中的 K/V 对中的 V 是 page_id，而 leaf node 中的 K/V 对中的 V 是 rid（{page_id, slot_num}），同时 leaf node 还包含额外元数据（next_page_id_）；
2. B+Tree 非叶子节点的 `min_size = (max_size + 1)/2`，叶子节点的 `min_size = max_size / 2`；
3. 叶子节点插入后的键值对数量达到 max_size_ 则拆分；非叶子节点插入前的 child 数量为 max_size 则拆分；

#### task2：B+Tree Data Structure

1. Search

    {% asset_img B+Tree_search.jpg B+Tree_search %}

    - 需要注意图中的 `/*C是叶结点*/` 应该和 while 循环是同一缩进；

    - 原书中的 非叶子结点 K/V 键值对是如下组织的，忽略最后一个 K，因此和实验中忽略第一个 K 的组织方式略有不同，最终会导致取子树 index 时出现加一减一之类的细微区别（寻找非叶子节点中的第一个严格大于查询键的 Key 的索引更简洁）。

        {% asset_img node.jpg node %}