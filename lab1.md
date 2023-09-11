# Buffer Pool
Buffer Pool的最主要的作用是为了缓存磁盘中的数据页，以减小磁盘I/O。此外，提前在内存中开辟部分内存空间，当需要从磁盘中读入page(比如数据库page、B+树节点等)时，可以直接从Buffer Pool拿到内存空间。当要释放拿到的内存空间时，可以直接通过Buffer Pool进行回收，从而避免page读到内存时需要反复开辟和释放内存空间，避免系统开销。
## Extendible Hash
可扩展hash保存的是<page id, frame id>，其中frame id是Buffer Pool中每个页的id，通过page id可以快速找到其对应的frame id。整体思路是根据全局深度将page id映射为目录项id，然后找到这个目录项指向的桶，最后再从桶中找到<page id, frame id>。
```C++
// 会Insert，其他都会了
template <typename K, typename V>
void ExtendibleHashTable<K, V>::Insert(const K &key, const V &value) {
  // UNREACHABLE("not implemented");
  // 全局深度的作用是将page id转换为目录项id，而局部深度则是将元素分到不同的桶中，以及找到目录项要指向的桶是哪个
  latch_.lock();
  size_t t = IndexOf(key);
  while (!dir_[t].get()->Insert(key, value)) {
    // 插入失败就会进行循环扩容
    if (dir_[t]->GetDepth() < global_depth_) {
      // 局部深度小于全局深度，那么就分裂桶
      std::shared_ptr<Bucket> old_bucket(dir_[t]);
      // 局部深度+1
      (*old_bucket).IncrementDepth();
      std::shared_ptr<Bucket> new_bucket(new Bucket(bucket_size_, dir_[t].get()->GetDepth()));
      num_buckets_++;
      // 查找所有指向old_bucket的目录项
      std::vector<int> old_pointer;
      int i = 0;
      for (auto &b : dir_) {
        if (b == old_bucket) {
          old_pointer.push_back(i);
        }
        i++;
      }
      std::vector<int> new_pointer;
      auto &old_list = (*old_bucket).GetItems();
      auto &new_list = (*new_bucket).GetItems();
      auto &sentry = old_list.front();
      // 先对目录项的指向进行分类，然后再将数据项分散到新桶和旧桶中
      for (auto b = old_pointer.begin(); b != old_pointer.end();) {
        // 拿旧桶的第一个元素代表旧桶，提取每个目录项id和第1个元素key的前局部深度个位，如果不相等，那么就是指向新桶
        if (!old_bucket->GetPointBucket(*b, sentry.first)) {
          new_pointer.push_back(*b);
          b = old_pointer.erase(b);
          continue;
        }
        b++;
      }
      // 将数据分两类
      for (auto b = old_list.begin(); b != old_list.end();) {
        // 同理以第1个元素作为基准，将旧桶的元素分到两个桶中，做法和上面一样
        if (old_bucket->GetIndex(sentry.first) != old_bucket->GetIndex(b->first)) {
          std::pair<K, V> px(b->first, b->second);
          new_list.push_back(std::move(px));
          b = old_list.erase(b);
          continue;
        }
        b++;
      }
      // 分别将指针指向对应的桶
      for (auto &b : old_pointer) {
        dir_[b] = old_bucket;
      }
      for (auto &b : new_pointer) {
        dir_[b] = new_bucket;
      }
    } else {
      // 扩容，全局深度+1，相当于目录项数量×2
      global_depth_++;
      int num = dir_.size();
      // 插入新目录项并更改指向，对于每个新的目录项，遍历一遍所有当前存在的桶，以找到其要指向的桶
      for (int i = num; i < 2 * num; i++) {
        for (auto &b : dir_) {
          // 对于遍历的每个桶，根据桶的局部深度，计算出当前正在处理的新目录项id的前局部深度个位，并将其转换为目录项id
          // 如果计算出来的目录项指向的桶刚好是正在遍历的桶，那就指向这个桶就行了
          int index = GetFrontIndex(i, (*b).GetDepth());
          if (dir_[index] == b) {
            std::shared_ptr<Bucket> p(b);
            dir_.push_back(std::move(p));
            break;
          }
        }
      }
      // 记住更新索引值，因为global_depth会变
      t = IndexOf(key);
    }
  }
  latch_.unlock();
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
  std::scoped_lock<std::mutex> s1(latch_);
  if (free_list_.empty() && replacer_->Size() == 0) {
    return nullptr;
  }
  frame_id_t frame_id;
  if (!free_list_.empty()) {
    // 从free_list中取出第1个空闲页
    frame_id = free_list_.front();
    free_list_.pop_front();
    // 获得要用的page_id，初始化page数组中的page
  } else {
    // 只能够通过LRU-K来获得空闲页，这里一定能够获得可替换页，因为上面已经检测过了
    // 直接调用换出函数，返回被换出的frame_id
    replacer_->Evict(&frame_id);
    // 存在可以替换的页，先判断该页是否为脏页，如果是则写出，在reset所有信息
    Page &replace_page = pages_[frame_id];
    if (replace_page.is_dirty_) {
      // 直接write就行了
      disk_manager_->WritePage(replace_page.GetPageId(), replace_page.GetData());
    }
    // reset该页的信息
    replace_page.ResetMemory();
    replace_page.page_id_ = INVALID_PAGE_ID;
  }
  *page_id = AllocatePage();
  Page &p = pages_[frame_id];
  p.page_id_ = *page_id;
  // 不要读盘，因为这里只是为了拿到缓存页而已
  // disk_manager_->ReadPage(*page_id, p.GetData());
  p.pin_count_++;
  p.is_dirty_ = false;
  // 将frame加入到可扩展哈希和LRU-K中
  page_table_->Insert(*page_id, frame_id);
  replacer_->RecordAccess(frame_id);
  return &pages_[frame_id];
}

auto BufferPoolManagerInstance::FetchPgImp(page_id_t page_id) -> Page * {
  std::scoped_lock<std::mutex> s1(latch_);
  frame_id_t frame_id;
  if (page_table_->Find(page_id, frame_id)) {
    if (pages_[frame_id].page_id_ == page_id) {
      // 如果找到了该页面所对应的frame_id，并且frame中记录的page_id为我们要找的，那么就返回该frame
      // 注意要更新LRU-K
      replacer_->RecordAccess(frame_id);
      // 注意每fetch1次就要pin1次
      if (pages_[frame_id].pin_count_ <= 0) {
        // 这里要重新判断，如果原来的pin_count_=0，即原来是可以删除的，则需要重新设置不让其被删除，因为有线程使用
        replacer_->SetEvictable(frame_id, false);
      }
      pages_[frame_id].pin_count_++;
      return &pages_[frame_id];
    }
  }
  // 若找不到那么就只能新建一个页了，这里调用不了那个NewPgImp，因为这里制定了page_id，上面那个是返回page_id，没有指定的
  if (free_list_.empty() && replacer_->Size() == 0) {
    return nullptr;
  }
  if (!free_list_.empty()) {
    // 从free_list中取出第1个空闲页
    frame_id = free_list_.front();
    free_list_.pop_front();
    // 获得要用的page_id，初始化page数组中的page
  } else {
    // 只能够通过LRU-K来获得空闲页，这里一定能够获得可替换页，因为上面已经检测过了
    // 直接调用换出函数，返回被换出的frame_id
    replacer_->Evict(&frame_id);
    // 存在可以替换的页，先判断该页是否为脏页，如果是则写出，在reset所有信息
    Page &replace_page = pages_[frame_id];
    if (replace_page.is_dirty_) {
      // 写出该页
      disk_manager_->WritePage(replace_page.GetPageId(), replace_page.GetData());
    }
    // reset该页的信息
    replace_page.ResetMemory();
    replace_page.page_id_ = INVALID_PAGE_ID;
  }
  Page &p = pages_[frame_id];
  p.page_id_ = page_id;
  disk_manager_->ReadPage(page_id, p.GetData());
  p.pin_count_++;
  p.is_dirty_ = false;
  // 将frame加入到可扩展哈希和LRU-K中
  page_table_->Insert(page_id, frame_id);
  replacer_->RecordAccess(frame_id);
  return &pages_[frame_id];
}
```
- - -
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