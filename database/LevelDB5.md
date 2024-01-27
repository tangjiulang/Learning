# Iterator

![](./../img/iterator-system.svg)

## MergingIterator

用于合并多个 SST 文件的迭代器

### 成员变量

* `const Comparator* comparator_`：
* `IteratorWrapper* children_`：防止迭代器过多，在堆上分配
* `int n_`：实时记录 Iterator 个数
* `IteratorWrapper* current_`：当前使用的迭代器
* `Direction direction_`：迭代方向

### 成员函数

#### FindSmallest

找到 key 最小的 Iterator

#### FindLargest

找到 key 最大的 Iterator

#### 构造函数

为每一个 Iterator 分配一个 MergingIterator

#### SeekToFirst

<img src="./../img/image-20240126171730004.png" alt="image-20240126171730004" style="zoom:67%;" />

```cpp
void SeekToFirst() override {
  for (int i = 0; i < n_; i++) {
    // 所有的都归 0
    children_[i].SeekToFirst();
  }
  // 找到 key 最小的 MergingIterator，注意现在的情况是当前所有的都被指向第 0 位的比较
  FindSmallest();
  direction_ = kForward;
}
```

#### SeekToLast

<img src="./../img/image-20240126171845839.png" alt="image-20240126171845839" style="zoom:67%;" />

#### Seek

<img src="./../img/image-20240126172112563.png" alt="image-20240126172112563" style="zoom:67%;" />

```cpp
void Seek(const Slice& target) override {
  // 将所有 Iterator 都 seek 到 >= target 的最小的位置
  for (int i = 0; i < n_; i++) {
    children_[i].Seek(target);
  }
  // 然后选一个最小的
  FindSmallest();
  direction_ = kForward;
}
```

#### Next

![image-20240126180711673](./../img/image-20240126180711673.png)

这里的 Next 不是我们理解的链表的那个 Next，它是先将所有 children 指向 key >= key() 的位置，然后从其中找一个最小的。

如上图所示，先将 Iterator2 里指向其中 11 的位置，然后将 Iterator3 里指向 12 的位置，最后将 Iterator1 里指向 13 的位置。

然后在 Iterator1，2，3 中找到 key 最小的那一个。

```cpp
void Next() override {
  assert(Valid());

  if (direction_ != kForward) {
    for (int i = 0; i < n_; i++) {
      IteratorWrapper* child = &children_[i];
      if (child != current_) {
        child->Seek(key());
        if (child->Valid() &&
            comparator_->Compare(key(), child->key()) == 0) {
          child->Next();
        }
      }
    }
    direction_ = kForward;
  }

  current_->Next();
  FindSmallest();
}
```

# Compaction

<img src="./../img/Compaction.png" style="zoom:80%;" />

![](./../img/8fb07a86-0fe0-4583-96a0-dc72f620989d.svg)

上面是对于 Compact 过程的两种关系图

在 leveldb 中，compaction 共有两种，分别叫 minor compaction 和 major compaction。

- minor compaction，将 immtable dump 到 SStable
- major compaction，level 之间的 SSTable compaction。

`leveldb::Compaction` 用来记录筛选文件的结果，其中 `inputs[2]` 记录了参与 compact 的两层文件，是最重要的两个变量

## 成员变量

`int level_`：需要压缩的 level

`uint64_t max_output_file_size_`：压缩之后最大的文件大小，等于 options->max_file_size

`Version* input_version_`：当前需要操作的版本

`VersionEdit edit_`：版本变化

`std::vector<FileMetaData*> inputs_[2]`：level 和 level+1 两层需要参与压缩的文件元数据

`std::vector<FileMetaData*> grandparents_`：grandparent 元数据

`size_t grandparent_index_`：grandparent 下表索引

`bool seen_key_`;             // Some output key has been seen

`int64_t overlapped_bytes_`：当前压缩与 grandparent files 重叠的字节数

`size_t level_ptrs_[config::kNumLevels]`：用于记录某个 user_key 与 >=level+2 中每一层不重叠的文件个数

## 重要函数

### IsTrivialMove

本次参与 Compact 的 SST 能否直接移动到上一层

```cpp
bool Compaction::IsTrivialMove() const {
  const VersionSet* vset = input_version_->vset_;
  // 1. 参与的 SST 只有一个文件
  // 2. Level 层的 SST 文件和 Level + 1 层没有 Overlap
  // 3. Level 层的 SST 文件和 Level + 2 层的 Overlap 小于阈值（避免后面 Level + 1 层和 Level + 2 层 Overlap 涉及到的文件过大）
  return (num_input_files(0) == 1 && num_input_files(1) == 0 &&
          TotalFileSize(grandparents_) <=
              MaxGrandParentOverlapBytes(vset->options_));
}
```

### AddInputDeletions

```cpp
void Compaction::AddInputDeletions(VersionEdit* edit) {
  for (int which = 0; which < 2; which++) {
    for (size_t i = 0; i < inputs_[which].size(); i++) {
      edit->RemoveFile(level_ + which, inputs_[which][i]->number);
    }
  }
}
```

将需要删除的文件添加到 Edit 中

**因为 input 经过变化生成 output，因此 input 对应到 deleted_file 容器，output 进入 added_file 容器。需要注意在前面在 add 的时候，先忽略掉 deleted 里面的，因为 IsTrivialMove 是直接移动文件**。

### IsBaseLevelForKey

判断当前 user_key 在` >= (level+2)` 层中是否已经存在。主要是用于 key 的 `type=deletion` 时可不可以将该 key 删除掉。

```cpp
bool Compaction::IsBaseLevelForKey(const Slice& user_key) {
  // Maybe use binary search to find right entry instead of linear search?
  const Comparator* user_cmp = input_version_->vset_->icmp_.user_comparator();
  for (int lvl = level_ + 2; lvl < config::kNumLevels; lvl++) {
    const std::vector<FileMetaData*>& files = input_version_->files_[lvl];
    while (level_ptrs_[lvl] < files.size()) {
      FileMetaData* f = files[level_ptrs_[lvl]];
      if (user_cmp->Compare(user_key, f->largest.user_key()) <= 0) {
        // We've advanced far enough
        if (user_cmp->Compare(user_key, f->smallest.user_key()) >= 0) {
          // Key falls in this file's range, so definitely not base level
          return false;
        }
        break;
      }
      level_ptrs_[lvl]++;
    }
  }
  return true;
}
```

主要还是利用 level_ptrs_ 找到对应的 SST 文件

但是这个函数我们可以发现，实际上是一个单增的，所以我们可以推测出，删除的时候一定是有序的，并且是从小到大删除。

### ShouldStopBefore

为了避免合并到 Level + 1 层之后和 Level + 2层重叠太多，导致下次合并 Level + 1时候时间太久，因此需要及时停止输出，并生成新的 SST

```cpp
bool Compaction::ShouldStopBefore(const Slice& internal_key) {
  const VersionSet* vset = input_version_->vset_;
  // Scan to find earliest grandparent file that contains key.
  const InternalKeyComparator* icmp = &vset->icmp_;
  while (grandparent_index_ < grandparents_.size() &&
         icmp->Compare(internal_key,
                       grandparents_[grandparent_index_]->largest.Encode()) >
             0) {
    if (seen_key_) {
      overlapped_bytes_ += grandparents_[grandparent_index_]->file_size;
    }
    grandparent_index_++;
  }
  seen_key_ = true;

  if (overlapped_bytes_ > MaxGrandParentOverlapBytes(vset->options_)) {
    // Too much overlap for current output; start new output
    overlapped_bytes_ = 0;
    return true;
  } else {
    return false;
  }
}
```

很简单的一个函数，其目的在于找到一个合适的 Internal Key 进行 compact

### ReleaseInputs

释放内存 
