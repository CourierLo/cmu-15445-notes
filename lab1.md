—— Written by **CourierLo**

# Buffer Pool

Buffer Pool的最主要的作用是为了缓存磁盘中的数据页，以减小磁盘I/O。此外，提前在内存中开辟部分内存空间，当需要从磁盘中读入page(比如数据库page、B+树节点等)时，可以直接从Buffer Pool拿到内存空间。当要释放拿到的内存空间时，可以直接通过Buffer Pool进行回收，从而避免page读到内存时需要反复开辟和释放内存空间，避免系统开销。

1. 为什么要自己做缓存池，操作系统不是有pagecache吗，为什么不直接拿来用作缓存？
- 开发层面，MMAP并没有完备的stage-commit API，它会**自动地**将dirty page写回！你可以让它强制写回但是不能让它不写回。这样很难实现consistency——事务没有commit中间结果就被持久化了！为事务实现增加了不必要的麻烦。
- 性能层面，MMAP有性能局限
  - 并发：OS内核能提供的并发度不一定能够满足所有应用的吞吐量，如果内核数据结构出现瓶颈，应用开发者就束手无策。
  - 吞吐量：文件吞吐量大的时候MMAP文件频繁换出换入，换出page cache需要修改相关页表和TLB。页表问题不大，但是TLB每个核都有一个，意味着换出page cache需要同步清理所有核心的TLB！相当于一次全核中断，这对吞吐量和时延影响都很大。如果bypass掉OS page cache，自己管理一个恒定大小的buffer pool，这时候进程页表和CPU TLB不受影响。
  - 使用MMAP只能使用OS提供的pre-fetch, write-back, swap策略，上限远不如应用层自己定制。
总结一下就是，在**高并发，高磁盘写入量**的场景下，OS原有的page cache设计有较大不足，OS PageCache API也不能提供对磁盘IO的精细控制。对于这种场景的应用，应该自己实现文件缓存。

- 如果没有这种场景的应用，OS PageCache好处就很多：
  - 开发层面，不用向应用层缓存一样记录内存block和文件block的映射关系，不用自己实现脏block写回等。
  - 性能层面，不仅可以减少一次内核态到用户态的拷贝，还可以用减少系统调用的使用(直接操作内存即可，不需要使用大量的read()/write())

- Andy的墓碑上要刻"I HATE MMAP"

2. LRU-K怎么实现？ 淘汰策略？与普通LRU比的好处
- 分两个list: buffer list和history list。buffer list存储访问次数多于k次的页面，而history list则存访问次数不足k次的页面。buffer list服从LRU算法，驱逐最近那一次记录也是第k次记录最早的frame。而history list遵循FIFO策略(这个可以自定义，445要求FIFO)，驱逐第一次记录最早的frame.

- LRU算法无法区分那些被相对频繁引用和那些非常少被引用的页面，直到系统浪费了大量资源，因为它将很少被引用的页面保留在缓冲区中很长一段时间。(其实就是较少引用的page的buffer life太长了)

- 缓存污染：指系统将不常用的数据从内存移到缓存，造成常用数据的挤出，降低了缓存效率的现象。偶发性的、周期性的批量操作会使临时数据涌入缓存，挤出热点数据，导致LRU热点命中率急剧下降，缓存污染情况比较严重。而LRU-K可以降低这个问题的影响。实际应用中LRU-2是综合各种因素后最优的选择，LRU-3或者更大的K值命中率会高，但适应性差，需要大量的数据访问才能将历史访问记录清除掉。

3. k怎么选择？依据
- It turns out that, for K >2, the LRU-K aigorithm provides somewhat improved performance over LRU-2 for stable patterns of access, but is less responsive to changes in access patterns, an important consideration for some applications.

- 随着K值的增加，LRU-K算法将更多的引用历史纳入考虑。较大的K值可以改善性能，特别是在稳定的访问模式下。然而，较大的K值可能会导致算法对访问模式的变化反应较慢，因为它需要更多的引用来更新页面的优先级。不同的应用可能需要不同的K值来获得最佳性能。因此，通常需要通过实验和性能测试来确定最适合特定情况的K值。

4. 可扩展哈希
可拓展哈希表使用一个哈希函数来将键映射到特定的桶(buckets)。如果插入到一个已满的桶，会触发扩展操作。哈希表会维护一个全局深度global depth，而每个桶都有自己的局部深度local depth。可拓展哈希还会维护一个dir_目录指针数组，指向不同的桶。dir_数组会以2的指数增长，但是桶不是，多个指针可以指向同一个桶。指向同一个桶的指针被称为兄弟指针。插入一个已满的桶会有两种情况：
  - Global Depth < Local Depth: 此时新建一个桶，修改yyy1xxxx兄弟指针指向新桶，yyy0xxxx指针不变，然后redistribute, local depth+1
  - Global Depth = Local Depth: 此时新建一个桶，指针数量*2，新指针指向该桶，redistribute, local/global depth+1

## Extendible Hash
可扩展hash保存的是<page id, frame id>，其中frame id是Buffer Pool中每个页的id，通过page id可以快速找到其对应的frame id。整体思路是根据全局深度将page id映射为目录项id，然后找到这个目录项指向的桶，最后再从桶中找到<page id, frame id>。
```C++
template <typename K, typename V>
void ExtendibleHashTable<K, V>::Insert(const K &key, const V &value) {
  std::scoped_lock<std::shared_mutex> lock(big_latch_);
  while (true) {
    auto id = IndexOf(key);
    if (!dir_[id]->IsFull()) {
      dir_[id]->Insert(key, value);
      return;
    }

    num_buckets_++;
    if (GetLocalDepth(id) == GetGlobalDepth()) {
      global_depth_++;
      dir_[id]->IncrementDepth();
      // dir_数量*2
      for (int i = 0; i < (1 << (GetGlobalDepth() - 1)); ++i) {
        dir_.emplace_back(dir_[i]);
      }

      // 不是再hash(key)，而是 id | (1 << (globalDepth - 1)) 新桶二进制: 1xxx
      auto new_id = id | (1 << (global_depth_ - 1));
      dir_[new_id] = std::make_shared<Bucket>(bucket_size_, GetLocalDepth(id));
      RedistributeBucket(dir_[id]);
    } else {
      // 这里的id要根据local depth来hash才行
      int mask = (1 << GetLocalDepth(id)) - 1;
      auto old_id = std::hash<K>()(key) & mask;
      dir_[old_id]->IncrementDepth();  // increase local depth of old_id or id ?
      // 需要修改兄弟指针的指向
      auto len = GetGlobalDepth() - GetLocalDepth(old_id);
      auto new_id = old_id | (1 << (GetLocalDepth(old_id) - 1));  // 应该给new_id创建新的bucket
      dir_[new_id] = std::make_shared<Bucket>(bucket_size_, GetLocalDepth(old_id));
      for (int i = 1; i < (1 << len); ++i) {
        // local_depth+1后，yyy0xxxx的dir_指向不用变, 将yyy1xxxx的指向新dir_
        dir_[new_id | (i << GetLocalDepth(new_id))] = dir_[new_id];
      }
      RedistributeBucket(dir_[old_id]);
    }
  }
}

template <typename K, typename V>
auto ExtendibleHashTable<K, V>::RedistributeBucket(std::shared_ptr<Bucket> bucket) -> void {
  std::list<std::pair<K, V>> &list = bucket->GetItems();
  for (auto it = list.begin(); it != list.end();) {
    int mask = (1 << bucket->GetDepth()) - 1;
    auto id = std::hash<K>()(it->first) & mask;
    if (bucket == dir_[id]) {
      it++;
    } else {
      dir_[id]->Insert(it->first, it->second);
      it = list.erase(it);
    }
  }
}
```
## LRU-K
LRU-K用了两条队列（两条队列里面都是LRU的策略淘汰，只不过页面淘汰时先从历史队列里面淘汰，失败才从缓冲队列中淘汰），分别是历史队列和缓冲队列，历史队列为访问次数小于k的page，一旦访问次数达到k次，那么就将page从历史队列移到缓冲队列里面。LRU-K的目的是用来替换掉最近最少使用的page，即从Buffer Pool中将这个page换出，以腾出空间给那些想要从Buffer Pool中拿缓存页的事务。**<u>唯一要注意的是Pin的数量，只有为0才能被淘汰，且如果淘汰的page是脏页，那么就需要写到磁盘里面。</u>**

## Buffer Pool
用来分配缓存页，一开始申请一大块内存空间，并将其分为若干个缓存页，并且需要为每个缓存页记录当前有多少个事务正在使用它。主要的内容有两个：
- NewPgImp具体流程为：一旦收到内存空间申请，那么就会从空闲缓存页链表里面找空闲的page分配。如果没有空闲的page了，那么就让LRU-K淘汰一个page出来。如果无法淘汰，那么就无法分片缓存页。
- FetchPgImp具体流程为：先通过可扩展hash查看一下Buffer Pool是否已经为需要fetch的page分配缓存页了，如果是，那么直接pin count+1。否则就是NewPgImp的东西了。

注意上面两个操作都需要更新LRU-K，因为相当于访问了一次缓存页。
```C++
auto BufferPoolManagerInstance::NewPgImp(page_id_t *page_id) -> Page * {
  std::scoped_lock<std::mutex> lock(latch_);

  frame_id_t frame_id;
  if (!free_list_.empty()) {
    frame_id = free_list_.front();
    free_list_.pop_front();
  } else if (replacer_->Evict(&frame_id)) {
    page_table_->Remove(pages_[frame_id].page_id_);
  } else {
    return nullptr;
  }

  // if page is dirty, write back
  if (pages_[frame_id].IsDirty()) {
    disk_manager_->WritePage(pages_[frame_id].GetPageId(), pages_[frame_id].GetData());
  }

  // reset the memory and metadata
  *page_id = AllocatePage();
  pages_[frame_id].ResetMemory();
  pages_[frame_id].page_id_ = *page_id;
  pages_[frame_id].pin_count_ = 1;
  pages_[frame_id].is_dirty_ = false;

  page_table_->Insert(*page_id, frame_id);

  replacer_->RecordAccess(frame_id);
  replacer_->SetEvictable(frame_id, false);

  return &pages_[frame_id];
}

auto BufferPoolManagerInstance::FetchPgImp(page_id_t page_id) -> Page * {
  std::scoped_lock<std::mutex> lock(latch_);
  frame_id_t frame_id;

  if (page_table_->Find(page_id, frame_id)) {
    pages_[frame_id].pin_count_++;
    replacer_->RecordAccess(frame_id);
    replacer_->SetEvictable(frame_id, false);
    return &pages_[frame_id];
  }

  // not in buffer pool, need to read from disk
  // get a frame_id first
  if (!free_list_.empty()) {
    frame_id = free_list_.front();
    free_list_.pop_front();
  } else if (replacer_->Evict(&frame_id)) {
    page_table_->Remove(pages_[frame_id].GetPageId());
  } else {
    return nullptr;
  }

  // if the old page is dirty, write back
  if (pages_[frame_id].IsDirty()) {
    disk_manager_->WritePage(pages_[frame_id].GetPageId(), pages_[frame_id].GetData());
  }

  // reset the memory and metadata, do we have to reset mem?
  pages_[frame_id].ResetMemory();
  // read from disk
  disk_manager_->ReadPage(page_id, pages_[frame_id].GetData());
  pages_[frame_id].page_id_ = page_id;
  pages_[frame_id].pin_count_ = 1;
  pages_[frame_id].is_dirty_ = false;

  replacer_->RecordAccess(frame_id);
  replacer_->SetEvictable(frame_id, false);

  page_table_->Insert(page_id, frame_id);

  return &pages_[frame_id];
}
```

---

锁优化，没时间不理了
```C++
auto BufferPoolManagerInstance::FetchPgImp(page_id_t page_id) -> Page * {
  std::mutex &m = hash_latches_[page_id % hash_latches_size_];
  frame_id_t frame_id;
  if (page_table_->Find(page_id, frame_id)) {
    m.lock();
    pages_[frame_id].pin_count_++;
    replacer_->RecordAccess(frame_id);
    replacer_->SetEvictable(frame_id, false);
    auto res = &pages_[frame_id];
    m.unlock();
    return res;
  }

  std::scoped_lock<std::mutex> lock(latch_);
  std::scoped_lock<std::mutex> lock1(m);

  if (page_table_->Find(page_id, frame_id)) {
    pages_[frame_id].pin_count_++;
    replacer_->RecordAccess(frame_id);
    replacer_->SetEvictable(frame_id, false);
    return &pages_[frame_id];
  }

  // not in buffer pool, need to read from disk
  // get a frame_id first
  if (!free_list_.empty()) {
    frame_id = free_list_.front();
    free_list_.pop_front();
  } else if (replacer_->Evict(&frame_id)) {
    printf("%06ld LEAD S%d [BPM FCH EVT] Page %d Frame %d Evict.\n", GetTime(), gettid(), pages_[frame_id].page_id_,
           frame_id);
    page_table_->Remove(pages_[frame_id].GetPageId());
  } else {
    return nullptr;
  }

  // if the old page is dirty, write back
  if (pages_[frame_id].IsDirty()) {
    disk_manager_->WritePage(pages_[frame_id].GetPageId(), pages_[frame_id].GetData());
  }

  // reset the memory and metadata, do we have to reset mem?
  pages_[frame_id].ResetMemory();
  // read from disk
  disk_manager_->ReadPage(page_id, pages_[frame_id].GetData());
  pages_[frame_id].page_id_ = page_id;
  pages_[frame_id].pin_count_ = 1;
  pages_[frame_id].is_dirty_ = false;

  replacer_->RecordAccess(frame_id);
  replacer_->SetEvictable(frame_id, false);

  page_table_->Insert(page_id, frame_id);

  return &pages_[frame_id];
}
```

```C
// First try
// Assume NHASH = 11
Thread1             Thread2
FetchPgImp(1)        
                    FetchPgImp(13)
Lock(1)
                    Lock(2)
Find(frame 1)
                    Not found, try to evict frame 1 
                    Reset memory of frame 1
Read(frame 1)
```
所以`LRUKReplacer`在选择一个frame_id进行`evict`，需要对该frame的page_id的哈希桶上锁，直到`FetchPgImp`的末尾。


打算用CoucurrentLRU重新实现一份

--- 

实测，如果使用6.s081 lock lab的策略，会导致交叉上锁，虽然不会死锁，但是gradescope不允许这种行为。而且写性能会变得很差。