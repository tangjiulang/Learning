# WiscKey

#### 关键思想

Key 和 Value 分开，Key 存于 LSM 中，Value 存于 vlog 中

为了处理未排序的值（这需要在范围查询期间进行随机访问），WiscKey使用SSD设备的并行随机读取特性

WiscKey利用独特的崩溃一致性和垃圾收集技术来有效管理值日志。

WiscKey通过删除LSM树日志而不牺牲一致性来优化性能，从而减少小写入带来的系统调用开销。

#### 设计目的

低写入放大：写入放大会引入额外不必要的**写入**

低读取放大：大读取放大会导致两个问题。首先，通过为每次查找发出多次读取，查找的吞吐量会显著降低。其次，加载到内存中的大量数据会降低缓存的效率。

SSD 优化：有效地利用了顺序写入和并行随机读取，以便应用程序能够充分利用设备的带宽。

支持使 LSM-tree 流行的现代功能

## 键值分离

![WiscKey -- Separating Keys from Values in SSD-conscious Storage · Columba  M71's Blog](img\wisckey-layout.png)

由于 key 的大小小于 value，导致 Wisckey 的 LSM 树会小于 LevelDB 的 LSM 树，搜索一个信息搜索的表文件更少，并且由于 <key，addr> 比 <key，value> 小，所以缓存中可以存更多的 <key，addr> 键值对，通过这个可以优化读取