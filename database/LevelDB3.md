# LevelDB 迭代器

## Iterator基类

![](./../img/5a1916be-620b-4c07-9d33-61eae5b1d44d.svg)

LevelDB各个组件用不同的格式进行数据存取。在LevelDB内部、外部、各个不同阶段又不可避免的需要从不同的视角遍历这些数据。如果每一个层次的数据遍历都需要详细的关心全部数据存储格式，无疑将使得整个过程变得无比的冗余复杂。Iterator的出现正式为了解决这个问题，Iterator在各个层次上，向上层实现提供了：**无须了解下层存储细节的情况下，通过统一接口对下层数据进行遍历的能力。**采用虚函数进行抽象

### 接口

- Seek到某一位置：Seek，SeekToFirst，SeekToLast；
- 访问前驱后继：Next，Prev；
- 判断当前位置是否有效：Valid；
- 获取当前位置数据信息：key，value，status；
- 可以注册多个Cleanup方法，当Iterator析构前做一些清理操作。

### 基本 Iterator

#### MemTable Iterator

在 Memtable Skiplist 的格式上的 Iterator 实现。

MemTableIterator 是对 memtable 的迭代，本质还是对 skiplist 的迭代。

#### Block::Iter 

针对 SST 文件 Block 存储格式的 Iterator 实现。遍历的过程中解析重启点，拼接 key 的共享部分和特有部分，获取对应的 value 值。

#### Version::LevelFileNumIterator

Version中记录了当前所有文件按层次划分的二维数组。其中Level1层之上的文件由于相互之间没有交集且有序，可以利用文件信息中的最大最小Key来进行二分查找。LevelFileNumIterator就是利用这个特点实现的对文件元信息进行遍历的Iterator。其中每个项记录了当前文件最大key到文件元信息的映射关系。这里的文件元信息包含文件号及文件长度。

### 组合 Iterator

#### TwoLevelIterator

TwoLevelIterator 实现逻辑上有层次关系的数据的遍历操作。组合了 **index iterator** 和 **data iterator** 两层迭代器，其中 index iterator 记录从数据 key 值到 data iterator 的映射，而 data iterator 则负责真正数据 key 到 value 的映射。生成 TwoLevelIterator 时，需要提供 index Iterator 及 BlockFunction 函数，其中 BlockFunction 实现了 index iterator value 值的反序列化以及对应的 data iterator 的生成。

#### MergingIterator

通过 MergingIterator 可以实现多个有序数据集合的归并操作。其中包含多个 **child iterator** 组成的集合。对 MergingIterator 的遍历会有序的遍历其 child iterator 中的每个元素。

### 功能 Iterator

#### Table::Iterator

对SST文件的遍历，可以利用上面介绍的 TwoLevelIterator，其 **index iterator** 为 Index Block 的 Block::Iter，**data iterator **为 Data Block 的 Block::Iter

#### **NewInternalIterator**

LevelDB 作为整体同样通过 Iterator 向外部用户提供遍历全部数据的能力。这里使用 MergingIterator 将 Memtable，Immutable memtable 及各层 SST 文件的 Iterator 归并起来，使得外部使用者不用关心具体的内部实现而有序的循环 LevelDB 内部的数据，LevelDB 首先实现了 NewInternalIterator，在 NewInternalIterator 的基础上，LevelDB 有封装了 DBIter 来处理快照，过滤已删除 key。

<img src="./../img/iterator2.png" style="zoom:67%;" />

#### Compaction过程Iterator

Compaction过程中需要对多个文件进行归并操作，并将结果输出到新的下层文件。

- 如果有 Level0 文件，则包含所有 level0 文件的 Table::Iterator
- 其他 Level 文件，加入 NewConcatenationIterator，作为一个 TwoLevelIterator，由 Version::LevelFileNumIterator 作为 index iterator，Table::Iterator 作为 data iterator

<img src="./../img/iterator3.png" style="zoom:67%;" />

# 版本管理

## version

`Version`用于表示某次 compaction 或者打开/恢复的数据库状态。为什么需要版本的概念呢？？

针对共享的资源，有三种方式：

- 悲观锁 ，这是最简单的处理方式。加锁保护，读写互斥。效率低。
- 乐观锁，它假设多用户并发的事物在处理时不会彼此互相影响，各事务能够在不产生锁的的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。 果其他事务有更新的话，正在提交的事务会进行回滚；这样做不会有锁竞争更不会产生死锁， 但如果数据竞争的概率较高，效率也会受影响 。
- MVCC，MVCC 是一个数据库常用的概念。Multiversion concurrency control 多版本并发控制。每一个执行操作的用户，看到的都是数据库特定时刻的的快照 (snapshot), writer 的任何未完成的修改都不会被其他的用户所看到；当对数据进行更新的时候并是不直接覆盖，而是先进行标记，然后在其他地方添加新的数据(这些变更存储在 versionedit)，从而形成一个新版本，此时再来读取的 reader 看到的就是最新的版本了。所以这种处理策略是维护了多个版本的数据的，但只有一个是最新的(versionset 中维护着全局最新的 seqnum)。

## 数据结构

### 成员变量

- `VersionSet* vset_`：隶属于哪个 versionset

- `Version* next_`

- `Version* prev_`

- `int refs_`：有多少服务还引用着这个版本

- `std::vector<FileMetaData*> files_[config::kNumLevels]`：当前版本所有数据

- `FileMetaData* file_to_compact_`：用于 seek 次数超过阈值之后需要压缩的文件

- `int file_to_compact_level_`：用于 seek 次数超过阈值之后需要压缩的文件所在的 level

- `double compaction_score_`：用于检查 size 超过阈值之后需要压缩的文件

- `int compaction_level_`：用于检查 size 超过阈值之后需要压缩的文件所在的 level

### 成员函数

- 析构函数：删除当前版本中引用技术为 0 的 file

- `NewConcatenatingIterator`：创建两层迭代器

- `AddIterators`：保存每一层的迭代器，其中第 0 层和非 0 层创建的迭代器不一样

- `Get`：用于从 SST 中读取所需要的数据

- `UpdateStats`：当查找文件而没有查找到时，更新 seek 次数状态

- `RecordReadSample`：统计读的样本，主要是用在迭代器中

- `GetOverlappingInputs`：获取指定 level 层与所给范围 key 重叠的 SST 文件

- `OverlapInLevel`：判断是否与当前 level 层有 key 重叠

- `PickLevelForMemTableOutput`：选择内存中数据 dump 到磁盘中的哪一层

- `NumFiles`：表示某一层有多少个 SST 文件

- `ForEachOverlapping`：搜索与指定范围 key 的 SST 集合

### 实现

CompactMemTable 和 BackgroundCompaction 过程中会导致新文件的产生和旧文件的删除。每当这个时候都会有一个新的对应的 Version 生成，并插入 VersionSet 链表头部。

也就是说 Version 是在 Compact 的过程中诞生的，Version 的作用就是实现 SST 的 MVCC，从而做到快照读的作用。

## State 类

```cpp
struct State {
  GetStats stats;  // Holds first matching file
  int matches;

  static bool Match(void* arg, int level, FileMetaData* f) {
  State* state = reinterpret_cast<State*>(arg);
  state->matches++;
  if (state->matches == 1) {
    // Remember first match.
    state->stats.seek_file = f;
    state->stats.seek_file_level = level;
  }
  // We can stop iterating once we have a second match.
  return state->matches < 2;
  } 
};

State state;
state.matches = 0;
ForEachOverlapping(ikey.user_key, internal_key, &state, &State::Match);
```

