# LevelDB

![img](../img/kvstore_leveldb_kyotocabinet_small.jpg)

## 基本组件

#### 字节序

- 将低序字节存储在起始地址，称为小端；
- 将高序字节存储在起始地址，称为大端；

LevelDB 采用小端

#### Slice

查询一个区间的数据

主要操作为拷贝构造函数

```cpp
class LEVELDB_EXPORT Slice {
 public:
  Slice() : data_(""), size_(0) {}
  Slice(const char* d, size_t n) : data_(d), size_(n) {}
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}
  Slice(const char* s) : data_(s), size_(strlen(s)) {}
  Slice(const Slice&) = default;
  Slice& operator=(const Slice&) = default;
  const char* data() const { return data_; }
  size_t size() const { return size_; }
  bool empty() const { return size_ == 0; }

  // Return the ith byte in the referenced data.
  // REQUIRES: n < size()
  char operator[](size_t n) const;

  // Drop the first "n" bytes from this slice.
  void remove_prefix(size_t n);

  int compare(const Slice& b) const;

  // Return true iff "x" is a prefix of "*this"
  bool starts_with(const Slice& x) const {
    return ((size_ >= x.size_) && (memcmp(data_, x.data_, x.size_) == 0));
  }

 private:
  const char* data_;
  size_t size_;
};
```

成员变量：`data_`（数据地址），`size_`（数据长度）

成员函数：`remove_prefix`，`start_with`，`compare`

`remove_prefix`，将前 `n bytes` 的数据从 `slice` 中移除

`start_with`，判断 `x` 是否是 `slice` 的前缀

`compare`，判断两个 `slice` 是否相同以及或者谁是谁的前缀

#### Status

用于记录 LevelDB 中状态信息，保存错误码和对应的字符串错误信息(不过不支持自定义)。

![image-20231228145628610](../img/image-20231228145628610.png)

##### code

```cpp
enum Code {
    kOk = 0,
    kNotFound = 1,
    kCorruption = 2,
    kNotSupported = 3,
    kInvalidArgument = 4,
    kIOError = 5
};
```

#### 编码

LevelDB 中分为定长和变长编码，其中变长编码目的是为了减少空间占用。其基本思想是：每一个 Byte 最高位 bit 用 0/1 表示该整数是否结束，用剩余 7bit 表示实际的数值，在 protobuf 中被广泛使用。

<img src="../img/image-20231228153218126.png" alt="image-20231228153218126" style="zoom: 50%;" />

#### Option

```cpp
struct LEVELDB_EXPORT Options {
  Options();
  const Comparator* comparator;
  bool create_if_missing = false;
  bool error_if_exists = false;
  bool paranoid_checks = false;
  Env* env;
  Logger* info_log = nullptr;
  size_t write_buffer_size = 4 * 1024 * 1024;
  int max_open_files = 1000;
  Cache* block_cache = nullptr;
  size_t block_size = 4 * 1024;
  int block_restart_interval = 16;
  size_t max_file_size = 2 * 1024 * 1024;
  CompressionType compression = kSnappyCompression;
  int zstd_compression_level = 1;
  bool reuse_logs = false;
  const FilterPolicy* filter_policy = nullptr;
};

struct LEVELDB_EXPORT ReadOptions {
  bool verify_checksums = false;
  bool fill_cache = true;
  const Snapshot* snapshot = nullptr;
};

struct LEVELDB_EXPORT WriteOptions {
  WriteOptions() = default;
  bool sync = false;
};

}  // namespace leveldb
```

##### Option 通用

1. `Comparator`：被用来表中key比较，默认是字典序

2. `create_if_missing`：打开数据库，如果数据库不存在，是否创建新的

3. `error_if_exists`：打开数据库，如果数据库存在，是否抛出错误

4. `paranoid_checks`：如果为 true，则实现将对其正在处理的数据进行积极检查，如果检测到任何错误，则会提前停止。 这可能会产生不可预见的后果：例如，一个数据库条目的损坏可能导致大量条目变得不可读或整个数据库变得无法打开。

5. `env`：封装了平台相关接口

6. `info_log`：db 日志句柄

7. `write_buffer_size`：memtable 的大小(默认 4mb)

   - 值大有利于性能提升

   - 但是内存可能会存在两份，太大需要注意oom

   - 过大刷盘之后，不利于数据恢复

8. `max_open_files`：允许打开的最大文件数

9. `block_cache`：block 的缓存

10. `block_size`：每个 block 的数据包大小(未压缩)，默认是 4k

11. `block_restart_interval`：block 中记录完整 key 的间隔

12. `max_file_size`：生成新文件的阈值(对于性能较好的文件系统可以调大该阈值，但会增加数据恢复的时间)，默认 2k

13. `compression`：数据压缩类型，默认是 kSnappyCompression，压缩速度快

14. `reuse_logs`：是否复用之前的 MANIFES 和 log files

15. `filter_policy`：block 块中的过滤策略，支持布隆过滤器

##### Read Option

1. `verify_checknums`：是否对从磁盘读取的数据进行校验
2. `fill_cache`：读取到block数据，是否加入到cache中
3. `snapshot`：记录的是当前的快照

##### Write Option

`sync`：是否同步刷盘，也就是调用完 write 之后是否需要显式 fsync

##### Configs

1. `kNumLevels`：磁盘上最大的 level 个数，默认为 7
2. `kL0_CompactionTrigger`：第 0 层 SSTable 个数到达这个阈值时触发压缩，默认值为 4
3. `kL0_SlowdownWritesTrigger`：第 0 层 SSTable 到达这个阈值时，延迟写入 1ms，将 CPU 尽可能的移交给压缩线程，默认值为 8
4. `kL0_StopWritesTrigger`：第 0 层 SSTable 到达这个阈值时将会停止写，等到压缩结束，默认值为 12
5. `kMaxMemCompactLevel`：新压缩产生的 SSTable 允许最多推送至几层(目标层不允许重叠)，默认为 2
6. `kReadBytesPeriod`：在数据迭代的过程中，也会检查是否满足压缩条件，该参数控制读取的最大字节数
7. `MaxBytesForLevel` 函数：每一层容量大小为上一层的 10 倍
8. `MaxGrandParentOverlapBytes`：$level - n$ 和 $leveldb-n+2$ 之间重叠的字节数，默认大小为 $10*max\_file\_size$

#### SkipList

LevelDB 的线段跳表

##### Node

```cpp
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node {
  explicit Node(const Key& k) : key(k) {}
  Key const key;
  Node* Next(int n);
  void SetNext(int n, Node* x);

  Node* NoBarrier_Next(int n);
  void NoBarrier_SetNext(int n, Node* x);
 private:
  std::atomic<Node*> next_[1];
};
```

在 `Skiplist` 中，`Node` 代表的是跳表中的节点

结构体包含两个变量，当前节点的 Key 值和指向下一个节点的 `next_[1]`

结构体包含了四个方法 `Next、SetNext、NoBarrier_Next、NoBarrier_SetNext`。前两个方法对应加载与读取（内存使用 `barrier` 按需进行），后两个方案对应非屏障方案（可以乱序执行）加载与读取。这四种方案均对 `next_[1]` 进行操作。

##### Iterator

`Skiplist` 中的工具类

```cpp
class Iterator {
   public:
    explicit Iterator(const SkipList* list);
    bool Valid() const;
    const Key& key() const;
    void Next();
    void Prev();
    void Seek(const Key& target);
    void SeekToFirst();
    void SeekToLast();
   private:
    const SkipList* list_;
    Node* node_;
  };
```

有两个成员变量，SkipList 和当前 Node

##### Skiplist

抛开 Iterator 后，和普通的 Skiplist 一样，从 level 高的地方开始找，没找到就跳下一个，下一个如果大于要找的值或者没有下一个，level 就下一层，重复上面的步骤直到找到或者 level == 0 停止

插入的时候会随机一个 height 值，这个值代表 node 的高度，和查询相似，但是每次在 level 要减一的时候，我们会将当前 node 保存起来，在找到最后一个的时候，再从下往上一层一层的添加到 node 的后面。

删除类似插入，找到所有下一层是我们删除的 node 的节点 pre，然后将 pre.next = node.next，然后直到 level == 0，将 node[n] 回收

#### 内存管理

##### 成员变量

- `alloc_ptr_`：当前已使用内存的指针

- `blocks_`：实际分配的内存池

- `alloc_bytes_remaining_`：剩余内存字节数

- `memory_usage_`：记录内存的使用情况

- `kBlockSize`：一个块大小(4k)

#### 成员函数

需要注意的是 `AllocateFallback` 函数

```cpp
char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) {
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}
```

当 `bytes` 大于 `kBlockSize/4` 时，是大内存，如果我们直接分配一个 `Block` 的话，那么剩下剩下的空位比较小，那么此 `Block` 复用次数比较少，并且此时可能有 `1/4` 的左侧资源被浪费，所以选择按需分配

当 `bytes` 小于 `kBlockSize/4` 时，此时新开一个 `Block`，既可以大量复用，又可以减少左侧资源的浪费

左侧资源指当前 `Block` 的 `alloc_bytes_remaining_`

#### 引用计数

引用计数是一种内存管理方式。在 LevelDB 中，`memtable/version` 等组件，可能被多个线程共享，因此不能直接删除，需要等到没有线程占用时，才能删除。

#### 各种 key 和 compare

##### 各种 key

- `user_key`：用户输入的数据 `key`(`slice` 格式)

- `ParsedInternalKey`：

  - 是对`InternalKey`的解析，因为`InternalKey`是一个字符串

  - `Slice user_key`;

  - `SequenceNumber sequence`

  -   `ValueType type`

- InternalKey：

  - 内部 `key`，常用来比如 `key` 比较等场景

  - `std::string rep_`

- `memtable key`：
  - 故名思义：存储在 `memtable` 中的 `key`，这个 `key` 比较特殊他是包含 `value` 的

- `lookup key`：

  - 用于 `DBImpl::Get` 中

  - 成员变量

    - `char space_[200]`，`lookup` 中有一个细小的内存优化，就是类似 `string` 的 `sso` 优化。其优化在于，当字符串较短时，直接将其数据存在栈中，而不去堆中动态申请空间，这就避免了申请堆空间所需的开销。

    - `const char* start_`

    - `const char* kstart_;`

    - `const char* end_;`

关系如下图所示

<img src="../img/f919f5bb-7a17-4edb-918b-1d98edd3e01f.png" style="zoom:50%;" />

##### 各种 compare

###### 成员函数

- `Compare()`：支持三种操作：大于/等于/小于

- `Name()` ：比较器名字，以 LevelDB 开头

- `FindShortestSeparator`：

  - 这些功能用于减少索引块等内部数据结构的空间需求。

  - 如果 `*start < limit`，则将 `*start` 更改为 `[start,limit)` 中的短字符串。

  - 简单的比较器实现可能会以 `*start` 不变返回，即此方法的实现不执行任何操作也是正确的。

  - 然后在调用 `Compare` 函数

- `FindShortSuccessor`：将 `*key` 更改为 `string >= *key.Simple` 比较器实现可能会在 `*key` 不变的情况下返回，即，此方法的实现是正确的。

特点：必须要支持线程安全

#### WriteBatch

WriteBatch 使用批量写来提高性能，支持 put 和 delete。

##### 成员变量

- `rep_`：WriteBatch 具体数据

- `WriteBatchInternal`：内部工具性质的辅助类

##### 成员函数

- `Put`：存储 `key` 和 `value` 信息

- `Delete`：追加删除 `key` 信息

- `Append`：多个 WriteBatch 还可以继续合并

- `Iterate`：

  - 遍历该 `batch` 结构，为了方便扩展，参数使用的是 `Handler` 基类，对应的是抽象工厂模式

  - `MemTableInserter` 子类：对 `memtable` 的操作

- `ApproximateSize`：内存状态信息

![](../img/image.png)

这里运用了两个技巧 `b->rep_.data()+8` 和 `&b->rep_[8]`

其中前一个是去掉 `rep_` 中的 `SeqNum` 字段，但是是只读，后面一个是将 `SeqNum` 后面转换成 `char*` 传入 `EncodeFixed32` 对数字 `n` 进行编码

重点讲解一下 `Iterate`

```cpp
Status WriteBatch::Iterate(Handler* handler) const {
  Slice input(rep_);
  if (input.size() < kHeader) {
    return Status::Corruption("malformed WriteBatch (too small)");
  }
// 删除 SeqNum 还有 key number
  input.remove_prefix(kHeader);
  Slice key, value;
  int found = 0;
  while (!input.empty()) {
    found++;
    // 解析当前的 key value 状态，[kTypeValue, kTypeDeletion]
    char tag = input[0];
    // 取出来了，直接删除
    input.remove_prefix(1);
    switch (tag) {
      case kTypeValue:
        if (GetLengthPrefixedSlice(&input, &key) &&
            GetLengthPrefixedSlice(&input, &value)) {
          // 插入到 memtable 内
          handler->Put(key, value);
        } else {
          return Status::Corruption("bad WriteBatch Put");
        }
        break;
      case kTypeDeletion:
        if (GetLengthPrefixedSlice(&input, &key)) {
          handler->Delete(key);
        } else {
          return Status::Corruption("bad WriteBatch Delete");
        }
        break;
      default:
        return Status::Corruption("unknown WriteBatch tag");
    }
  }
  if (found != WriteBatchInternal::Count(this)) {
    return Status::Corruption("WriteBatch has wrong count");
  } else {
    return Status::OK();
  }
}
```

#### Env 家族

<img src="../img/c7cc89ec-b621-4bd6-9eb9-da89005ef598.png" style="zoom:50%;" />

由上图可知，该内就是一个纯虚类

`New` 开头的成员函数，其 `Status` 表示函数执行状态，而实际生成的对象是在函数的第二个参数，现代 `C++` 不建议这样操作。

最好利用现代 `C++` 的移动语义，并且将传入参数设置为 `const`

有三个实现版本

##### `PosixEnv`

封装了 `posix` 标准下所有接口

##### `WindowsEnv`

封装了 `win` 相关接口

##### `EnvWrapper`

虽然该类也继承 `Env`，但是他和上述两个作用不一样

将所有调用转发到其他的 `Env` 实现。

可能对希望覆盖另一个 `Env` 的部分功能的用户有用。

默认还是使用 `Env` 实现类

了解 `EnvWrapper` 前，我们先了解一下代理设计模式

###### 代理设计模式

有些时候，我们想做一些事但是自己没有资源或者自己做不好，就会想着花点钱请专业的人帮我们做。这是一种代理模式。

我是一个用户，我现在需要保洁服务，那么我就需要联系保洁公司，让他们安排相关的专员来帮我完成这件事。

##### `WritableFile & PosixWritableFile`

###### `WritableFile`

用于顺序写入的文件抽象。 实现必须提供缓冲，因为调用者可能一次将小数据量数据追加到文件中。`WritableFile` 是里面的函数都是纯虚函数。

`WritableFileImpl` 是 `WritableFile` 的具体实现，主要利用成员 `FileState*` 进行工作

```cpp
class WritableFileImpl : public WritableFile {
 public:
  WritableFileImpl(FileState* file) : file_(file) { file_->Ref(); }
  ~WritableFileImpl() override { file_->Unref(); }
  Status Append(const Slice& data) override { return file_->Append(data); }
  Status Close() override { return Status::OK(); }
  Status Flush() override { return Status::OK(); }
  Status Sync() override { return Status::OK(); }

 private:
  FileState* file_;
};
```

其中 `FileState` 的 `Ref` 和 `Unref` 都是简单的引用计数操作，`Append` 就是将 `Slice` 插入到文件末尾，并且根据 `BlockSize` 分块

######  `PosixWritableFile`

```cpp
class PosixWritableFile final : public WritableFile;
```

**成员变量**

* `kWritableFileBufferSize`：作为缓冲区，默认大小为 64k
* `char buf_[kWritableFileBufferSize]`
* `pos_`：`buf_` 当前已经使用的字节位置
* `fd_`：当前文件，对应的 fd
* `is_manifest_`：判断是否是 `manifest` 文件，因为 `manifest` 文件需要实时刷盘
* `filename_`：文件名字
* `dirname_`：文件所在的目录

**成员函数**

```cpp
Status Append(const Slice& data) override {
    size_t write_size = data.size();
    const char* write_data = data.data();

    size_t copy_size = std::min(write_size, kWritableFileBufferSize - pos_);
    // 当前 buffer 所能写入的最大 size
    std::memcpy(buf_ + pos_, write_data, copy_size);
    write_data += copy_size;
    write_size -= copy_size;
    pos_ += copy_size;
    if (write_size == 0) {
    	return Status::OK();
    }
    
    // kWritableFileBufferSize - pos_ < write_size
	Status status = FlushBuffer();
	// 将满的 buffer 刷入磁盘中，并将 Buffer 置为空
    if (!status.ok()) {
    	return status;
    }

    if (write_size < kWritableFileBufferSize) {
        std::memcpy(buf_, write_data, write_size);
        pos_ = write_size;
        return Status::OK();
    }
    return WriteUnbuffered(write_data, write_size);
}
```

##### `RandomAccessFile&PosixRandomAccessFile/PosixMmapReadableFile`

###### `Limiter`

1. `Helper` 类限制资源使用，避免耗尽。目前用于限制只读文件描述符和 `mmap` 文件使用，这样我们就不会用完文件描述符或虚拟内存，或者遇到非常大的数据库的内核性能问题。
2. 使用 `atomic` 原子变量，所以线程安全

###### `RandomAccessFile`

`RandomAccessFile` 用于随机读取文件内容的文件抽象。

线程安全。

###### `PosixRandomAccessFile`

```cpp
class PosixRandomAccessFile final : public RandomAccessFile;
```

**成员变量**

- `has_permanent_fd_`：表示是否每次 `read` 都需要打开文件

- `fd_`：`has_permanent_fd_=false`，`fd_` 一直等于 -1

- `fd_limiter_`：资源限制相关的

- `filename_`：文件名

**成员函数**

简单的构造函数还有析构函数，以及 `Read` 函数

###### `PosixMmapReadableFile`

LevelDB 中使用 `mmap` 共享内存来提高性能。

1. `mmap()` 函数可以把一个文件或者 `Posix` 共享内存对象映射到调用进程的地址空间。映射分为两种：

   1. 文件映射：文件映射将一个文件的一部分直接映射到调用进程的虚拟内存中。一旦一个文件被映射之后就可以通过在相应的内存区域中操作字节来访问文件内容了。映射的分页会在需要的时候从文件中（自动）加载。这种映射也被称为基于文件的映射或内存映射文件。

      <img src="../img/7687105b-152f-4dec-a7ff-36bfa976b7c4.png" style="zoom:50%;" />

   2. 匿名映射：一个匿名映射没有对应的文件。相反，这种映射的分页会被初始化为 0。

2. `mmap()` 系统调用在调用进程的虚拟地址空间中创建一个新映射。

<img src="../img/156036ef-0e40-4e67-9fb3-7873d8527fc1.png" alt="img" style="zoom:50%;" />

参数解析：

`addrs`：映射内存的启示地址，可以指定，也可以设置为 `NULL`，设置为 `NULL` 时，内核会自己寻找一块内存

`len`：表示映射到进程地址空间的字节数，它从被影射文件开头起第 `offset` 个字节处开始算，`offset` 通常设置成 0

`prot`：共享内存的保护参数，具体见下表

|  prot取值  |   功能说明   |
| :--------: | :----------: |
| PROT_READ  |     可读     |
| PROT_WRITE |     可写     |
| PROT_EXEC  |    可执行    |
| PROT_NONE  | 数据不可访问 |

`flags`：控制映射操作各个方面的选项的位掩码。这个掩码必须只包含下列值中一个。

|         flags          | 功能说明                                                     |
| :--------------------: | ------------------------------------------------------------ |
|       MAP_SHARED       | 变动是共享的                                                 |
|      MAP_PRIVATE       | 变动是私有的，意思是只在本进程有效，这就意味着会触发 `cow(copy on write)` 机制 |
|       MAP_FIXED        | 不允许系统选择与指定地址不同的地址。 如果无法使用指定的地址， `mmap()` 将失败。 如果指定了 MAP_FIXED，则 `addr` 必须是页面大小的倍数。 如果 MAP_FIXED 请求成功，则 `mmap()` 建立的映射将替换从 `addr` 到 `addr + len` 范围内进程页面的任何先前映射。不鼓励使用此选项。 |
|      MAP_NOCACHE       | 此映射中的页面不会保留在内核的内存缓存中。 如果系统内存不足，MAP_NOCACHE 映射中的页面将首先被回收。 该标志用于几乎没有局部性的映射，并为内核提供一个提示，即在不久的将来不太可能再次需要该映射中的页面。 |
|    MAP_HASSEMAPHORE    | 通知内核该区域可能包含信号量并且可能需要特殊处理。           |
| MAP_ANONYMOUS/MAP_ANON | 该内存不是从文件映射而来，其内容会被全部初始化为 0，而且 `mmap` 最后的两个参数会被忽略 |

`fd`：对应被打开的文件套接字，一般是通过 `open` 系统调用。`mmap` 返回后，该 `fd` 可以 `close` 掉

`offset`：文件起始位置

3. `munmap()` 系统调用执行与 `mmap()` 相反的操作，即从调用进程的虚拟地址空间中删除映射。

<img src="../img/555bb551-cf68-4752-9bde-edcedb189ab1.png" alt="img" style="zoom:50%;" />

4. `msync`：用于刷盘策略。内核的虚拟内存算法保持内存映射文件与内存映射区的同步，因此内核会在未来某个时刻同步，如果我们需要实时，则需要 `msync` 同步刷。

<img src="../img/c1ce0567-f6bc-45c6-be29-821890862033.png" alt="img" style="zoom:50%;" />

5. 需要特别注意两个信号：SIGBUG 和 SIGSEGV
   1. SIGBUG：表示我们在内存范围内访问，但是超出了底层对象的大小
   2.  SIGSEGV：表示我们已经在内存映射范围外访问了。

<img src="../img/6bfe07d3-4db3-4b0f-9d96-26db07167a20.png" alt="img" style="zoom:50%;" />

<img src="../img/d060fc91-49bf-45a5-aa51-916b52bddd99.png" alt="img" style="zoom:50%;" />

##### `SequentialFile`

顺序读取的抽象

- `PosixSequentialFile`

  - 使用 `read()` 在文件中实现顺序读取访问，非线程安全。

  - `fd_`：文件描述符

  - `filename_`：文件名称

- `WindowsSequentialFile`

##### `FileLock`

`FileLock` 的作用是：由于 Linux 和 Windows 对文件的句柄的抽象 ( fd 和 handle ) 不同，使用方式不同，所以用 FileLock 抽象。

其目的是在需要同时使用文件句柄和文件名的时候，可以通过 `FileLock` 对象直接找到

###### `PosixFileLock`
